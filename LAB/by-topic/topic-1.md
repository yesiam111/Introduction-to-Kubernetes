# Topic 1 - Kubernetes Introduction

## I. Deploy a Simple Kubernetes Cluster
### 1. Prepare Environment
```bash
sudo swapoff -a
sudo apt update && sudo apt install -y apt-transport-https ca-certificates curl
```
- Sample output
```bash
Swap off successful
Packages installed successfully
```

### 2. Install Docker Engine
```bash
sudo apt-get install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
```
- Sample output
```bash
Docker version 24.0.5, build ced0996
```

### 3. Install kubectl
```bash
sudo apt-get update
sudo apt-get install -y kubectl
kubectl version --client
```
- Sample output
```bash
Client Version: v1.30.0
```

### 4. Install kubeadm and kubelet
```bash
sudo apt-get install -y kubelet kubeadm
sudo systemctl enable kubelet
```
- Sample output
```bash
kubeadm version: &version.Info{Major:"1", Minor:"30", GitVersion:"v1.30.0"}
```

### 5. Initialize Control Plane Node
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
- Sample output
```bash
Your Kubernetes control-plane has initialized successfully!
```

### 6. Configure kubectl Access
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
- Sample output
```bash
Configuration copied to ~/.kube/config
```

### 7. Install Pod Network Add-on (Flannel)
```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
- Sample output
```bash
podsecuritypolicy.policy/psp.flannel.unprivileged created
clusterrole.rbac.authorization.k8s.io/flannel created
```

### 8. Join Worker Node (on worker)
```bash
sudo kubeadm join <control-plane-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```
- Sample output
```bash
This node has joined the cluster!
```

### 9. Verify Cluster Status
```bash
kubectl get nodes
kubectl get pods -A
```
- Sample output
```bash
NAME     STATUS   ROLES           AGE   VERSION
master   Ready    control-plane   5m    v1.30.0
worker   Ready    <none>          3m    v1.30.0
```

---

## II. Labels and Selectors
### 1. Apply Labels to a Pod
```bash
kubectl run nginx --image=nginx --labels="env=staging,tier=frontend"
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
NAME    READY   STATUS
nginx   1/1     Running
```

### 3. Use Label Selector (Set-based)
```bash
kubectl get pods -l 'tier in (frontend,backend)'
```
- Sample output
```bash
NAME    READY   STATUS
nginx   1/1     Running
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
