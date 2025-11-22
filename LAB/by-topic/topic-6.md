# Topic 6 – Kubernetes Networking (Integrated Full Lab)

## I. Pod-to-Pod Networking
### 1. Verify basic Pod communication in the cluster
```bash
kubectl create ns net-lab

kubectl run pod-a --image=busybox -n net-lab -- sleep 3600
kubectl run pod-b --image=busybox -n net-lab -- sleep 3600

kubectl get pod -n net-lab -o wide
kubectl exec -n net-lab pod-a -- ping -c 4 <POD-B-IP>
```
- Sample output
```bash
4 packets transmitted, 4 received, 0% packet loss
```

### 2. Inspect Pod network namespace, routing, DNS configuration
```bash
kubectl exec -n net-lab pod-a -- ip a
kubectl exec -n net-lab pod-a -- ip route
kubectl exec -n net-lab pod-a -- cat /etc/resolv.conf
```
- Sample output
```bash
default via 10.244.1.1 dev eth0
nameserver 10.96.0.10
```

### 3. Validate cross-node Pod connectivity (overlay routing)
```bash
kubectl get pod -n kube-system -o wide | grep -E 'flannel|calico'

kubectl get pod -n net-lab -o wide

kubectl exec -n net-lab pod-a -- ping -c 4 <POD-B-IP>
```
- Sample output
```bash
4 packets transmitted, 4 received
```

### 4. Use netshoot for deeper Pod-level inspection
```bash
kubectl run netshoot --rm -it --image=nicolaka/netshoot -n net-lab -- bash

ip a
ip route
dig kubernetes.default.svc.cluster.local
tcpdump -ni eth0
```
- Sample output
```bash
GET / HTTP/1.1
200 OK
```

---

## II. Service Networking
### 1. Deploy a workload and expose it with ClusterIP
```bash
kubectl create deployment nginx --image=nginx -n net-lab
kubectl expose deployment nginx --port=80 --target-port=80 -n net-lab

kubectl get svc -n net-lab
kubectl describe svc nginx -n net-lab
```
- Sample output
```bash
nginx   ClusterIP   10.96.120.15   80/TCP
```

### 2. Compare NodePort and LoadBalancer behaviors
```bash
kubectl expose deployment nginx --name=nginx-nodeport --port=80 --type=NodePort -n net-lab
kubectl expose deployment nginx --name=nginx-lb --port=80 --type=LoadBalancer -n net-lab
kubectl get svc -n net-lab
```
- Sample output
```bash
nginx-nodeport   NodePort   80:32245/TCP
```

---

## III. DNS for Kubernetes
### 1. Validate DNS service discovery
```bash
kubectl run dns-test --image=busybox -n net-lab -- sleep 3600
kubectl exec -n net-lab dns-test -- nslookup nginx.net-lab.svc.cluster.local
```
- Sample output
```bash
Address: 10.96.120.15
```

### 2. Enable and inspect CoreDNS logs
```bash
kubectl -n kube-system edit configmap coredns
# add: `log` after `reload`, `loadbalance`, ...

kubectl -n kube-system rollout restart deployment coredns

kubectl logs -n kube-system -l k8s-app=kube-dns -f
```
- Sample output
```bash
INFO [nginx.net-lab.svc.cluster.local] query from 10.244.1.12
```

---

## IV. Network Policy Basics
### 1. Apply a default deny-all ingress policy
```bash
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

kubectl exec -n net-lab dns-test -- wget -qO- nginx.net-lab.svc.cluster.local
```
- Sample output
```bash
wget: download timed out
```

### 2. Allow specific communication to a target app
```bash
kubectl label pod -n net-lab dns-test role=frontend --overwrite

cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-nginx
  namespace: net-lab
spec:
  podSelector:
    matchLabels:
      app: nginx
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 80
EOF

kubectl exec -n net-lab dns-test -- wget -qO- nginx.net-lab.svc.cluster.local
```
- Sample output
```bash
<html><h1>Welcome to nginx!</h1></html>
```

---

## V. Advanced Network Policy and Traffic Control
### 1. Restrict external egress for all Pods
```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-external
  namespace: net-lab
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/8
EOF

kubectl exec -n net-lab dns-test -- wget -qO- http://1.1.1.1
```
- Sample output
```bash
wget: bad address
```

### 2. Multi-namespace policy: allow only app namespace to reach backend
```bash
kubectl create ns backend
kubectl label namespace net-lab env=app
kubectl run db --image=busybox -n backend -- sleep 3600
kubectl label pod -n backend db role=db

cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-to-db
  namespace: backend
spec:
  podSelector:
    matchLabels:
      role: db
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          env: app
EOF
```
- Sample output
```bash
networkpolicy.networking.k8s.io/allow-app-to-db created
```

---

## VI. Headless Service and Stateful Lookup
### 1. Validate direct Pod DNS resolution for stateful workloads
```bash
cat <<EOF | kubectl apply -n net-lab -f -
apiVersion: v1
kind: Service
metadata:
  name: demo
spec:
  clusterIP: None
  selector:
    app: demo
  ports:
  - port: 80
    name: web
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: demo
spec:
  serviceName: demo
  replicas: 2
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
EOF

kubectl exec -n net-lab dns-test -- nslookup demo-0.demo.net-lab.svc.cluster.local
```
- Sample output
```bash
Name: demo-0.demo.net-lab.svc.cluster.local
Address: 10.244.2.30
```

---

## VII. Cleanup
### 1. Remove all created resources
```bash
kubectl delete ns net-lab
kubectl delete ns backend
```
- Sample output
```bash
namespace "net-lab" deleted
namespace "backend" deleted
```
