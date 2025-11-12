# Topic 1 - Kubernetes Service
## I. NodePort Service
### 1. Expose application using NodePort Service
---
```bash
# Step 1: Create a simple deployment
kubectl create deployment webapp --image=nginx --replicas=2

# Step 2: Expose the deployment as a NodePort service
kubectl expose deployment webapp --port=80 --type=NodePort

# Step 3: Verify the service
kubectl get svc webapp
```
---
- Sample output
---
```bash
NAME      TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
webapp    NodePort   10.98.103.45   <none>        80:31000/TCP   10s
```

## II. LoadBalancer Service
### 1. Expose application using LoadBalancer Service
---
```bash
# Step 1: Create a new deployment
kubectl create deployment myapp --image=nginx

# Step 2: Expose it as LoadBalancer service
kubectl expose deployment myapp --port=80 --type=LoadBalancer

# Step 3: Check the external IP assigned
kubectl get svc myapp
```
---
- Sample output
---
```bash
NAME      TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)        AGE
myapp     LoadBalancer   10.100.87.12     203.0.113.10     80:30981/TCP   30s
```

## III. Ingress Controller Installation (NGINX + MetalLB)
### 1. Install MetalLB for bare-metal load balancing
---
```bash
# Step 1: Apply MetalLB namespace and components
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.10/config/manifests/metallb-native.yaml

# Step 2: Create address pool configuration (adjust IP range to match your network)
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  namespace: metallb-system
  name: default-address-pool
spec:
  addresses:
  - 192.168.1.240-192.168.1.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  namespace: metallb-system
  name: default
EOF
```
---
- Sample output
---
```bash
namespace/metallb-system unchanged
customresourcedefinition.apiextensions.k8s.io/addresspools.metallb.io created
deployment.apps/controller created
deployment.apps/speaker created
ipaddresspool.metallb.io/default-address-pool created
l2advertisement.metallb.io/default created
```

## IV. Deploy NGINX Ingress Controller
### 1. Install NGINX Ingress using Helm
---
```bash
# Step 1: Add ingress-nginx repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

# Step 2: Update repository
helm repo update

# Step 3: Install ingress controller
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```
---
- Sample output
---
```bash
NAME: ingress-nginx
LAST DEPLOYED: Wed Nov 12 10:20:00 2025
NAMESPACE: ingress-nginx
STATUS: deployed
REVISION: 1
```

## V. Ingress Resource Configuration
### 1. Configure Ingress rules for multiple services
---
```bash
# Step 1: Create two sample deployments and services
kubectl create deployment api-v1 --image=nginx
kubectl expose deployment api-v1 --port=80

kubectl create deployment api-v2 --image=nginx
kubectl expose deployment api-v2 --port=80

# Step 2: Create an Ingress resource routing requests based on path
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: example.local
    http:
      paths:
      - path: /api/v1
        pathType: Prefix
        backend:
          service:
            name: api-v1
            port:
              number: 80
      - path: /api/v2
        pathType: Prefix
        backend:
          service:
            name: api-v2
            port:
              number: 80
EOF
```
---
- Sample output
---
```bash
deployment.apps/api-v1 created
service/api-v1 created
deployment.apps/api-v2 created
service/api-v2 created
ingress.networking.k8s.io/api-ingress created
```

## VI. Verify Ingress Routing
### 1. Test routing functionality
---
```bash
# Get the external IP of ingress controller
kubectl get svc -n ingress-nginx

# Test access from local machine (replace IP with your ingress external IP)
curl http://<EXTERNAL-IP>/api/v1
curl http://<EXTERNAL-IP>/api/v2
```
---
- Sample output
---
```bash
<!DOCTYPE html>
<html>
<head><title>Welcome to nginx!</title></head>
<body>
<h1>Welcome to nginx!</h1>
</body>
</html>
```
