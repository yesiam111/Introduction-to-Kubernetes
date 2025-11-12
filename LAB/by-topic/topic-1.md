# Topic 1 - Kubernetes Introduction

## I. Deploy a Simple Kubernetes Cluster

Refer to [1.install-kubeadm.md](../1.install-kubeadm.md)

---

## II. Labels and Selectors
### 1. Apply Labels to a Pod
```bash
kubectl run nginx --image=nginx --labels="env=staging,tier=frontend"
kubectl run nginx-prod --image=nginx --labels="env=prod,tier=frontend"
kubectl get pods --show-labels
```
- Sample output
```bash
NAME    READY   STATUS    LABELS
nginx   1/1     Running   env=staging,tier=frontend
```

### 2. Use Label Selector (Equality-based)
```bash
kubectl get pods -l env=staging
```
- Sample output
```bash
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          2m6s
```

### 3. Use Label Selector (Set-based)
```bash
kubectl get pods -l 'tier in (frontend,backend)'
```
- Sample output
```bash
NAME         READY   STATUS    RESTARTS   AGE
nginx        1/1     Running   0          2m12s
nginx-prod   1/1     Running   0          17s
```

---

## III. Namespace
### 1. Create a Namespace
```bash
kubectl create namespace demo-ns
kubectl get namespaces
```
- Sample output
```bash
NAME              STATUS   AGE
default           Active   25m
demo-ns           Active   10s
```

### 2. Create Deployment in Namespace
```bash
cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
EOF

kubectl -n demo-ns apply -f deployment.yaml
```
- Sample output
```bash
deployment.apps/nginx-deployment created
```

### 3. Inspect Deployment and Pods
```bash
kubectl -n demo-ns get pods -l app=nginx
kubectl -n demo-ns describe deployment nginx-deployment
```
- Sample output
```bash
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-6dbb85fdd8-b8mkw   1/1     Running   0          1m
```
```bash
Name:                   nginx-deployment
Selector:               app=nginx
Replicas:               1 desired | 1 updated | 1 total | 1 available
```
