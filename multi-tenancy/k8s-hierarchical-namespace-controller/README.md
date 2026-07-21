## Prerequisites check

```bash
# Confirm your cluster from the previous exercise is still running
kubectl get nodes
kubectl cluster-info

# Confirm you're on Kubernetes 1.27+ (HNC v1.1 requires it)
kubectl version
```

### Step 1 — Install HNC

HNC is installed as a single manifest. It deploys a controller, a set of CRDs, and a validating webhook.

```bash
# Install HNC v1.1.0
kubectl apply -f https://github.com/kubernetes-sigs/hierarchical-namespaces/releases/download/v1.1.0/default.yaml

Wait for the controller to come up before doing anything else — the webhook will reject requests if the controller isn't ready:

# Watch until all HNC pods are Running
kubectl rollout status deployment/hnc-controller-manager \
  -n hnc-system --timeout=120s

# Confirm the CRDs landed
kubectl get crd | grep hnc
# You should see:
# hierarchicalresourcequotas.hnc.x-k8s.io
# hierarchyconfigurations.hnc.x-k8s.io
# subnamespaceanchors.hnc.x-k8s.io
```

### Step 2 — Install the kubectl-hns plugin

The kubectl hns plugin is how you manage the hierarchy. Without it you're writing raw HierarchyConfiguration YAML by hand.

```bash
# Install krew (if you don't have it already)

# Set the version to match your HNC installation
HNC_VERSION=v1.1.0

# Detect your OS and arch
OS=$(uname | tr '[:upper:]' '[:lower:]')
ARCH=$(uname -m | sed -e 's/x86_64/amd64/' -e 's/aarch64/arm64/')

echo "Downloading for: $OS/$ARCH"

# Download the binary
curl -fsSL \
  "https://github.com/kubernetes-sigs/hierarchical-namespaces/releases/download/${HNC_VERSION}/kubectl-hns_${OS}_${ARCH}" \
  -o /tmp/kubectl-hns

# Make it executable
chmod +x /tmp/kubectl-hns

# Move it somewhere on your PATH
sudo mv /tmp/kubectl-hns /usr/local/bin/kubectl-hns

# Verify
kubectl hns version
kubectl hns --help

# If you're on WSL and sudo isn't available
# Install to your user bin directory instead
mkdir -p ~/.local/bin
mv /tmp/kubectl-hns ~/.local/bin/kubectl-hns

# Make sure ~/.local/bin is on your PATH
export PATH="$HOME/.local/bin:$PATH"
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Verify
kubectl hns --help
kubectl hns version

kubectl get pods -n hnc-system
# All pods should show Running before the plugin can talk to the server
```

### Step 3 — Create the namespace hierarchy

Modelling: platform → app-prod and app-staging

```bash
# Create the parent namespace normally
kubectl create namespace platform

# Create children using HNC subnamespace anchors
# This is different from kubectl create namespace —
# HNC owns these and tracks the parent relationship
kubectl hns create app-prod -n platform
kubectl hns create app-staging -n platform

# Verify the tree
kubectl hns tree platform

# Output should look like this:
platform
├── [s] app-prod
└── [s] app-staging

[s] indicates subnamespaces

# The [s] means subnamespace — HNC created and owns it. If you try to delete it with kubectl delete namespace app-prod HNC will block you and tell you to use kubectl hns delete instead.
```

### Step 4 — Configure NetworkPolicy propagation

Configure HNC to copy NetworkPolicy objects from parent to children automatically.

```bash
# Set NetworkPolicy objects to propagate downward
kubectl hns config set-resource networkpolicies \
  --group networking.k8s.io \
  --mode Propagate

# Verify the config
kubectl hns config describe

# You should see networkpolicies listed under Propagate mode. There are three modes worth knowing for the interview:

Propagate — copies from parent to all descendants, keeps them in sync
Remove — deletes any propagated copies (used to undo Propagate)
Ignore — HNC leaves the resource alone entirely (default)
```
### Step 5 — Apply a policy to the parent and watch it propagate

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: platform
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector: {}
  egress:
  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
EOF
```

Now check whether it appeared in the children — without you touching those namespaces:

```bash
# Check app-prod
kubectl get networkpolicy -n app-prod

# Check app-staging
kubectl get networkpolicy -n app-staging

# Describe one to confirm it's an HNC-managed copy
kubectl describe networkpolicy deny-all-ingress -n app-prod

# In the describe output, look for this annotation `hnc.x-k8s.io/inherited-from: platform` — it proves HNC owns the copy, not you:
```

### Step 6 — Update the parent and watch children sync

This update demonstrates the live sync behaviour:

```bash
# Add an allow rule to the parent policy
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-internal
  namespace: platform
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: platform
EOF

# Within a few seconds, check children
kubectl get networkpolicy -n app-prod
# Both deny-all-ingress AND allow-internal should appear
```

### Step 7 — Override a policy in one child

This is where HNC gets interesting — and where you need to understand conflict resolution.

```bash
# Apply a DIFFERENT policy with the same name in app-prod
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: app-prod
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector: {}   # allow from ALL namespaces — wider than parent
EOF

# Expected output: Error from server (Forbidden): error when applying patch:

# HNC's admission webhook is blocking you from directly editing a propagated copy. The object in app-prod is owned by the parent namespace platform, so HNC treats it as read-only in the child. You can't patch it in place. The correct way to override is to delete the propagated copy first, then apply your local version:

# The inherited-from annotation should be gone
kubectl describe networkpolicy deny-all-ingress -n app-prod | grep -i inherited
# Expected: no output — this is now locally owned

# Check HNC flagged the conflict
kubectl hns describe app-prod
# Look for a Conditions section mentioning CannotPropagateObject or similar

# Note

# - HNC prevents accidental edits to propagated objects — good for safety
# - But overriding requires a delete-then-recreate, which creates a brief window of no policy between the delete and the apply
# - In a production cluster that window is a security gap — any traffic that was being denied by that policy is temporarily allowed

The mitigation is to apply the local override policy under a different name rather than overriding the parent's policy by name. That way both coexist with no gap:

# Better pattern — don't reuse the parent's policy name

cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-namespaces-ingress   # different name — no conflict, no gap
  namespace: app-prod
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector: {}
EOF

```

Check what happened in app-prod:

```bash
kubectl describe networkpolicy deny-all-ingress -n app-prod
# The hnc.x-k8s.io/inherited-from annotation will be GONE
# This is now a locally-owned policy, not an HNC copy

# Check the HNC status for conflicts
kubectl hns describe app-prod
```
### Step 8 — Add a ResourceQuota to the hierarchy

HNC also supports propagating ResourceQuotas. This is powerful for platform teams:

```bash
# First enable ResourceQuota propagation
kubectl hns config set-resource resourcequotas \
  --group "" \
  --mode Propagate

# Apply a quota to the parent
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: platform-quota
  namespace: platform
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    pods: "20"
EOF


# Verify it propagated

kubectl get resourcequota -n app-prod
kubectl get resourcequota -n app-staging
```

