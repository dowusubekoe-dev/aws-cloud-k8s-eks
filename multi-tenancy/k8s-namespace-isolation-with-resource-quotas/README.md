## Prerequisites

```bash
# Install kind (Kubernetes in Docker) — simpler than full Vagrant for this exercise
Confirm that you have Docker installed and running, then install kind:

brew install kind   # or: choco install kind

# Create a 3-node cluster
kind create cluster --name tenancy-lab --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF

kubectl cluster-info --context kind-tenancy-lab
```

### Step 1: Create a Namespace

```bash
kubectl create namespace team-alpha
kubectl create namespace team-beta
```

### Step 2: Apply LimitRange to the Namespaces

LimitRange sets defaults and ceilings per container. Without it, a pod with no `resources: block` gets scheduled with zero guaranteed CPU/memory — which breaks `ResourceQuota accounting`.

```bash
# create a file limitrange-alpha.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: team-alpha-limits
  namespace: team-alpha
spec:
  limits:
  - type: Container
    default:          # applied when a container omits limits:
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:   # applied when a container omits requests:
      cpu: "100m"
      memory: "128Mi"
    max:              # hard ceiling per container
      cpu: "2"
      memory: "2Gi"
    min:
      cpu: "50m"
      memory: "64Mi"
```
Use kubectl to apply the LimitRange to the both `team-alpha` and `team-beta` namespaces:

```bash
kubectl apply -f limitrange-alpha.yaml
kubectl apply -f limitrange-beta.yaml
```
OR

```bash
sed 's/team-alpha/team-beta/g' limitrange-alpha.yaml | kubectl apply -f -
```

### Step 3: Apply ResourceQuota to the Namespaces

Quota is the namespace-wide ceiling across all pods combined.

```bash
# create a file quota-alpha.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-alpha-quota
  namespace: team-alpha
spec:
  hard:
    requests.cpu: "2"
    requests.memory: "4Gi"
    limits.cpu: "4"
    limits.memory: "8Gi"
    pods: "10"
    services: "5"
    persistentvolumeclaims: "4"
```
Use kubectl to apply the ResourceQuota to the both `team-alpha` and `team-beta` namespaces:

```bash
kubectl apply -f quota-alpha.yaml
kubectl apply -f quota-beta.yaml
```
OR

```bash
sed 's/team-alpha/team-beta/g' quota-alpha.yaml | kubectl apply -f -
```
Verify if both quotas and limits are applied:

```bash
kubectl describe resourcequota team-alpha-quota -n team-alpha
kubectl describe limitrange team-alpha-limits -n team-alpha

kubectl describe resourcequota team-beta-quota -n team-beta
kubectl describe limitrange team-beta-limits -n team-beta
```

### Step 4 — Deploy a test workload to each namespace

```bash
# deploy-alpha.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: team-alpha
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
      team: alpha
  template:
    metadata:
      labels:
        app: web
        team: alpha
    spec:
      containers:
      - name: web
        image: nginx:alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: team-alpha
spec:
  selector:
    app: web
    team: alpha
  ports:
  - port: 80
    targetPort: 80
```

Use kubectl to apply the deployment and service to the `team-alpha` namespace:

```bash
kubectl apply -f deploy-alpha.yaml
sed 's/team-alpha/team-beta/g; s/team: alpha/team: beta/g; s/alpha\./beta\./g' deploy-alpha.yaml | kubectl apply -f -

# Confirm pods are running
kubectl get pods -n team-alpha
kubectl get pods -n team-beta

# Note the team-beta service ClusterIP — you'll need it for the network test
kubectl get svc -n team-beta
```

### Step 5 — Apply NetworkPolicy: deny all cross-namespace traffic

By default Kubernetes allows all pod-to-pod traffic. A NetworkPolicy changes that, but only if your CNI enforces it.
The DNS egress rule is easy to miss — without it, pods can't resolve service names at all, which makes the test ambiguous (connection refused vs DNS failure are very different failures).

```bash
# netpol-deny-alpha.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: team-alpha
spec:
  podSelector: {}        # applies to ALL pods in this namespace
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector: {}    # only allow from pods IN THE SAME namespace
  egress:
  - to:
    - podSelector: {}    # only allow to pods IN THE SAME namespace
  - ports:               # allow DNS resolution (otherwise nothing works)
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
```
Use kubectl to apply the NetworkPolicy to the `team-alpha` namespace:

```bash
kubectl apply -f netpol-deny-alpha.yaml
sed 's/team-alpha/team-beta/g' netpol-deny-alpha.yaml | kubectl apply -f -
```

### Step 6 — Prove the isolation works

```bash
# Get the ClusterIP of the team-beta service
BETA_IP=$(kubectl get svc web -n team-beta -o jsonpath='{.spec.clusterIP}')
echo "team-beta ClusterIP: $BETA_IP"

# Exec into a team-alpha pod and try to reach team-beta
ALPHA_POD=$(kubectl get pod -n team-alpha -l app=web -o jsonpath='{.items[0].metadata.name}')

kubectl exec -n team-alpha $ALPHA_POD -- \
  wget -qO- --timeout=5 http://$BETA_IP
# Get a shell in one of the team-alpha pods

# Expected result: timeout or connection refused — NOT the nginx welcome page
```
Then confirm in-namespace traffic still works:

```bash
# alpha pod to alpha service — this SHOULD work
ALPHA_IP=$(kubectl get svc web -n team-alpha -o jsonpath='{.spec.clusterIP}')
kubectl exec -n team-alpha $ALPHA_POD -- wget -qO- --timeout=5 http://$ALPHA_IP

# Expected: nginx welcome page HTML

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, nginx is successfully installed and working.
Further configuration is required for the web server, reverse proxy,
API gateway, load balancer, content cache, or other features.</p>

<p>For online documentation and support please refer to
<a href="https://nginx.org/">nginx.org</a>.<br/>
To engage with the community please visit
<a href="https://community.nginx.org/">community.nginx.org</a>.<br/>
For enterprise grade support, professional services, additional
security features and capabilities please refer to
<a href="https://f5.com/nginx">f5.com/nginx</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

### Step 7 — Add a selective allow rule (the interesting part)

In order to allow `team-alpha's` monitoring pod to scrape metrics from `team-beta`, you want to create a targeted allow rule rather than opening up general traffic.

```bash
# netpol-allow-monitoring.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-alpha-monitoring
  namespace: team-beta
spec:
  podSelector:
    matchLabels:
      app: web          # only affects team-beta web pods
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: team-alpha
      podSelector:
        matchLabels:
          role: monitoring   # only pods with this label get through
    ports:
    - port: 80

```
Use kubectl to apply the NetworkPolicy to the `team-beta` namespace:

```bash
kubectl apply -f netpol-allow-monitoring.yaml

# Label the alpha pod to simulate a monitoring pod
kubectl label pod -n team-alpha $ALPHA_POD role=monitoring

# Re-run the curl — now it should succeed
kubectl exec -n team-alpha $ALPHA_POD -- wget -qO- --timeout=5 http://$BETA_IP

```

#### Diagnose first — don't guess

```bash
kubectl exec -n team-alpha $ALPHA_POD -- wget -qO- --timeout=5 http://$BETA_IP
wget: download timed out
command terminated with exit code 1


# 1. Confirm the label actually got applied
kubectl get pod -n team-alpha $ALPHA_POD --show-labels
# You should see: role=monitoring in the labels column

# 2. Confirm the allow policy landed in team-beta
kubectl get networkpolicy -n team-beta
# Should show BOTH: deny-all-ingress AND allow-alpha-monitoring

# 3. Describe the allow policy to make sure it looks right
kubectl describe networkpolicy allow-alpha-monitoring -n team-beta

NetworkPolicy is like a firewall — you need to open both sides. Apply this:

# netpol-allow-egress-to-beta.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring-egress-to-beta
  namespace: team-alpha
spec:
  podSelector:
    matchLabels:
      role: monitoring        # only the monitoring pod gets this egress
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: team-beta
      podSelector:
        matchLabels:
          app: web
    ports:
    - port: 80
      protocol: TCP
  - ports:                    # keep DNS working
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP

kubectl apply -f netpol-allow-egress-to-beta.yaml
```

Verify both policies are in place, then retest

```bash
# team-alpha should now have 3 policies
kubectl get networkpolicy -n team-alpha

# team-beta should have 2 policies
kubectl get networkpolicy -n team-beta

# Retest
kubectl exec -n team-alpha $ALPHA_POD -- wget -qO- --timeout=5 http://$BETA_IP

This time you should get the nginx welcome page HTML back.
```

Then confirm non-monitoring pods are still blocked

```bash
# Get a second alpha pod (or create one without the monitoring label)
kubectl run test-pod \
  --image=busybox:1.36 \
  --restart=Never \
  -n team-alpha \
  -- sleep 3600

# This pod has no role=monitoring label — should still timeout
kubectl exec -n team-alpha test-pod -- \
  wget -qO- --timeout=5 http://$BETA_IP
# Expected: timeout

# Clean up
kubectl delete pod test-pod -n team-alpha
```

Apply the egress policy now:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring-egress-to-beta
  namespace: team-alpha
spec:
  podSelector:
    matchLabels:
      role: monitoring
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: team-alpha
      podSelector:
        matchLabels:
          app: web
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: team-beta
      podSelector:
        matchLabels:
          app: web
    ports:
    - port: 80
      protocol: TCP
  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
EOF

Then retest:

kubectl exec -n team-alpha $ALPHA_POD -- wget -qO- --timeout=5 http://$BETA_IP

```

### Step 8 — Test the ResourceQuota ceiling

```bash
# Scale team-alpha to 11 pods — quota allows max 10
kubectl scale deployment web -n team-alpha --replicas=11

kubectl get pods -n team-alpha
# 10 will be Running, 1 will be Pending

kubectl describe replicaset -n team-alpha | grep -A5 "Warning"
# You'll see: "exceeded quota: team-alpha-quota, requested: pods=1, used: pods=10, limited: pods=10"
```