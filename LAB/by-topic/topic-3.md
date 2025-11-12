# Topic 3 - Kubernetes Object Management

## I. Understanding Kubernetes Objects
### 1. Inspect Kubernetes object structure
```bash
kubectl run nginx --image=nginx
kubectl get pod nginx -o yaml
```
- Explain key section in kubernetes object

## II. Working with Kubernetes RBAC
### 1. Create a Role for specific resource access
```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
EOF
```
- Sample output
```bash
role.rbac.authorization.k8s.io/pod-reader created
```

### 2. Create a ClusterRole for cluster-wide permissions
```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
EOF
```
- Sample output
```bash
clusterrole.rbac.authorization.k8s.io/secret-reader created
```

### 3. Create RoleBinding to assign Role to a user
```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EOF
```
- Sample output
```bash
rolebinding.rbac.authorization.k8s.io/read-pods created
```

### 4. Create RoleBinding linked to a ClusterRole
```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-secrets
  namespace: demo-ns
subjects:
- kind: User
  name: dave
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
EOF
```
- Sample output
```bash
rolebinding.rbac.authorization.k8s.io/read-secrets created
```

### 5. Create ClusterRoleBinding for a user group
```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: manager
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
EOF
```
- Sample output
```bash
clusterrolebinding.rbac.authorization.k8s.io/read-secrets-global created
```

---

## III. Testing and Demonstrating Authorization

### 1. Check permissions of a specific user
```bash
kubectl auth can-i list pods --as=jane --namespace=default
kubectl auth can-i delete pods --as=jane --namespace=default
```
- Sample output
```bash
yes
no
```

### 2. Verify ClusterRole scope for another user
```bash
kubectl auth can-i get secrets --as=dave --namespace=demo-ns
kubectl auth can-i get secrets --as=dave --namespace=default
```
- Sample output
```bash
yes
no
```

### 3. Test global access for a user group
```bash
kubectl auth can-i get secrets --as=testuser --as-group=manager --namespace=default
kubectl auth can-i get secrets --as=testuser --as-group=manager --namespace=dev
```
- Sample output
```bash
yes
yes
```

### 4. Try unauthorized action
```bash
kubectl auth can-i delete secrets --as=jane --namespace=default
```
- Sample output
```bash
no
```

---

## IV. Service Account Creation and Binding
### 1. Create a ServiceAccount
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-robot
EOF
```
- Sample output
```bash
serviceaccount/build-robot created
```

### 2. Bind ClusterRole to ServiceAccount
```bash
kubectl create rolebinding my-sa-view \
  --clusterrole=view \
  --serviceaccount=demo-ns:build-robot \
  --namespace=demo-ns
```
- Sample output
```bash
rolebinding.rbac.authorization.k8s.io/my-sa-view created
```

### 3. Assign ServiceAccount to a Pod
```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      serviceAccountName: build-robot
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
EOF
```
- Sample output
```bash
deployment.apps/nginx-deployment created
```

### 4. Verify service account token usage
```bash
kubectl describe $(kubectl get pod -l app=nginx -o name)
```
- Sample output
```yaml
Name:             nginx-deployment-58c877d9fd-vgc4q
Namespace:        default
Priority:         0
Service Account:  build-robot
Node:             k8s-02/192.168.100.61
```

---

## V. ServiceAccount Authorization Test
### 1. Check permissions of ServiceAccount
```bash
kubectl auth can-i list pods --as=system:serviceaccount:demo-ns:build-robot --namespace=demo-ns
kubectl auth can-i get secrets --as=system:serviceaccount:demo-ns:build-robot --namespace=demo-ns
```
- Sample output
```bash
yes
no
```

### 2. Cleanup environment
```bash
kubectl delete role pod-reader -n default
kubectl delete clusterrole secret-reader
kubectl delete rolebinding read-pods -n default
kubectl delete rolebinding read-secrets -n demo-ns
kubectl delete clusterrolebinding read-secrets-global
kubectl delete serviceaccount build-robot
kubectl delete deployment nginx-deployment
```
- Sample output
```bash
role.rbac.authorization.k8s.io "pod-reader" deleted
clusterrole.rbac.authorization.k8s.io "secret-reader" deleted
rolebinding.rbac.authorization.k8s.io "read-pods" deleted
rolebinding.rbac.authorization.k8s.io "read-secrets" deleted
clusterrolebinding.rbac.authorization.k8s.io "read-secrets-global" deleted
serviceaccount "build-robot" deleted
deployment.apps "nginx-deployment" deleted
```

deployment.apps/nginx-deployment created
```
