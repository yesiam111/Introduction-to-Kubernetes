# Topic 3 - Kubernetes Object Management

## I. Understanding Kubernetes Objects
### 1. Inspect Kubernetes object structure
```bash
kubectl get pod nginx -o yaml
```
- Sample output
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:latest
status:
  phase: Running
```

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
  namespace: development
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

## III. Service Account Creation and Binding
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
  --serviceaccount=my-namespace:build-robot \
  --namespace=my-namespace
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
