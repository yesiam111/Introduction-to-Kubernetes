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

# Access service using curl from local or another host in same network
curl http://<NODE_IP>:<NODEPORT>
```  
---  
- Sample output  
---  
```bash
NAME      TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
webapp    NodePort   10.98.103.45   <none>        80:31000/TCP   10s
```  
---  

## II. Install MetalLB + NGINX Ingress Controller  
### 1. Install and configure MetalLB (bare-metal LoadBalancer)  
---  
```bash
# Confirm kubectl access
kubectl cluster-info

# (If using kube-proxy in IPVS mode) enable strictARP:
# within the config, set strictARP: true
kubectl get configmap kube-proxy -n kube-system -o yaml | \
  sed -e "s/strictARP: false/strictARP: true/" | \
  kubectl apply -n kube-system -f -

# Download the latest MetalLB manifest
MetalLB_RTAG=$(curl -s https://api.github.com/repos/metallb/metallb/releases/latest | \
             grep tag_name | cut -d '"' -f 4 | sed 's/v//')
mkdir ~/metallb && cd ~/metallb
wget https://raw.githubusercontent.com/metallb/metallb/v$MetalLB_RTAG/config/manifests/metallb-native.yaml

# Apply MetalLB components
kubectl apply -f metallb-native.yaml

# Create address-pool configuration (adjust range to your network)
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: production
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.30-192.168.1.50
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-advert
  namespace: metallb-system
spec:
  ipAddressPools:
  - production
EOF
```  
---  
- Sample output  
---  
```bash
namespace/metallb-system created
customresourcedefinition.apiextensions.k8s.io/addresspools.metallb.io created
customresourcedefinition.apiextensions.k8s.io/l2advertisements.metallb.io created
serviceaccount/controller created
serviceaccount/speaker created
deployment.apps/controller created
daemonset.apps/speaker created

kubectl get ipaddresspools.metallb.io -n metallb-system
NAME         AGE
production   23s

kubectl get l2advertisements.metallb.io -n metallb-system
NAME        AGE
l2-advert   49s
```  
---  

### 2. Install NGINX Ingress Controller using Helm  
---  
```bash
# Add repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install ingress controller into its namespace
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
---  

## III. Ingress Resource Configuration  
### 1. Deploy two services and configure Ingress rules  
---  
```bash
# Create two sample deployments and services
kubectl create deployment api-v1 --image=nginx
kubectl expose deployment api-v1 --port=80

kubectl create deployment api-v2 --image=nginx
kubectl expose deployment api-v2 --port=80

# Apply Ingress resource
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
### 2. Test ingress routing  

```bash
# Get external IP of ingress controller
kubectl get svc -n ingress-nginx

# Test via curl or browser, add /etc/hosts for example.local
curl http://example.local/api/v1
curl http://example.local/api/v2
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
---  

## IV. Deploy LoadBalancer Service  
### 1. Expose application using LoadBalancer Service  
---  
```bash
# Create a new deployment
kubectl create deployment myapp --image=nginx

# Expose it as a LoadBalancer service
kubectl expose deployment myapp --port=80 --type=LoadBalancer

# Check the external IP assigned
kubectl get svc myapp
```  
---  
- Sample output  
---  
```bash
NAME      TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)        AGE
myapp     LoadBalancer   10.100.87.12     192.168.1.30     80:30981/TCP   30s
```

### 2. Test LoadBalancer access  
---
```bash
curl http://<EXTERNAL-IP>
```

---

## V. Cleanup  
### 1. Remove all created resources  
---
```bash
# Delete apps and services
kubectl delete deployment webapp api-v1 api-v2 myapp
kubectl delete svc webapp api-v1 api-v2 myapp

# Delete ingress and MetalLB/Ingress controller
kubectl delete ingress api-ingress
helm uninstall ingress-nginx -n ingress-nginx
kubectl delete ns ingress-nginx metallb-system
```
