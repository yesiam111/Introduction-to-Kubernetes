# Topic 6 - Kubernetes Networking

## I. Understanding Pod-to-Pod Networking
### 1. Verify Pod network communication within the cluster
```bash
# Create a namespace for networking tests
kubectl create ns net-lab

# Deploy two pods in the same namespace
kubectl run pod-a --image=busybox -n net-lab -- sleep 3600
kubectl run pod-b --image=busybox -n net-lab -- sleep 3600

# Get pod IPs
kubectl get pod -n net-lab -o wide

# From pod-a, ping pod-b using its IP
kubectl exec -n net-lab pod-a -- ping -c 4 <POD-B-IP>
```
- Sample output
```bash
PING <POD-B-IP> (<POD-B-IP>): 56 data bytes
64 bytes from <POD-B-IP>: seq=0 ttl=64 time=0.425 ms
64 bytes from <POD-B-IP>: seq=1 ttl=64 time=0.350 ms
64 bytes from <POD-B-IP>: seq=2 ttl=64 time=0.310 ms
--- <POD-B-IP> ping statistics ---
4 packets transmitted, 4 received, 0% packet loss
```

## II. Service Networking
### 1. Expose Deployment via ClusterIP Service
```bash
# Create a Deployment with nginx
kubectl create deployment nginx --image=nginx -n net-lab

# Expose the Deployment as ClusterIP
kubectl expose deployment nginx --port=80 --target-port=80 -n net-lab

# Verify Service and Endpoints
kubectl get svc -n net-lab
kubectl describe svc nginx -n net-lab
```
- Sample output
```bash
NAME    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
nginx   ClusterIP   10.96.120.15   <none>        80/TCP    2m

Name:              nginx
Namespace:         net-lab
Selector:          app=nginx
Type:              ClusterIP
IP Families:       <none>
IP:                10.96.120.15
Port:              80/TCP
Endpoints:         10.244.0.12:80
```

## III. DNS Resolution in Kubernetes
### 1. Validate DNS service discovery between Pods
```bash
# Create another Pod to test DNS
kubectl run dns-test --image=busybox:1.28 -n net-lab -- sleep 3600

# Use nslookup to resolve the nginx service name
kubectl exec -n net-lab dns-test -- nslookup nginx.net-lab.svc.cluster.local
```
- Sample output
```bash
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      nginx.net-lab.svc.cluster.local
Address 1: 10.96.120.15 nginx.net-lab.svc.cluster.local
```

## IV. Network Policy Basics
### 1. Restrict traffic between Pods using NetworkPolicy
```bash
# Apply default deny-all ingress policy
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: net-lab
spec:
  podSelector: {}
  policyTypes:
  - Ingress
EOF

# Attempt to ping nginx service again from dns-test pod
kubectl exec -n net-lab dns-test -- wget -qO- nginx.net-lab.svc.cluster.local
```
- Sample output
```bash
wget: download timed out
```

## V. Allow Specific Traffic
### 1. Create policy to allow DNS and specific app communication
```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-nginx
  namespace: net-lab
spec:
  podSelector:
    matchLabels:
      app: nginx
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 80
EOF

# Test connectivity again
kubectl exec -n net-lab dns-test -- wget -qO- nginx.net-lab.svc.cluster.local
```
- Sample output
```bash
<!DOCTYPE html>
<html>
<head><title>Welcome to nginx!</title></head>
<body>
<h1>Welcome to nginx!</h1>
</body>
</html>
```

## VI. Cleanup
### 1. Delete created resources
```bash
kubectl delete ns net-lab
```
- Sample output
```bash
namespace "net-lab" deleted
```
