# Topic 4 - Kubernetes Pod and Container

## I. ConfigMap and Secret
### 1. Create and use ConfigMap in Pod
```bash
# Create a ConfigMap from literal values
kubectl create configmap game-demo \
  --from-literal=player_initial_lives=3 \
  --from-literal=ui_properties_file_name=user-interface.properties

# Verify ConfigMap content
kubectl get configmap game-demo -o yaml
```
- Sample output
```yaml
apiVersion: v1
data:
  player_initial_lives: "3"
  ui_properties_file_name: user-interface.properties
kind: ConfigMap
metadata:
  name: game-demo
```

### 2. Mount ConfigMap into Pod as env and volume
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo-pod
spec:
  containers:
  - name: demo
    image: alpine
    command: ["sleep", "3600"]
    env:
      - name: PLAYER_INITIAL_LIVES
        valueFrom:
          configMapKeyRef:
            name: game-demo
            key: player_initial_lives
    volumeMounts:
      - name: config
        mountPath: /config
        readOnly: true
  volumes:
  - name: config
    configMap:
      name: game-demo
EOF
```
- Sample output
```bash
pod/configmap-demo-pod created
```

### 3. Create and use Secret in Pod
```bash
kubectl create secret generic test-secret \
  --from-literal=username=my-app \
  --from-literal=password=39528$vdg7Jb

# Verify Secret
kubectl get secret test-secret -o yaml
```
- Sample output
```yaml
apiVersion: v1
data:
  password: Mzk1MjgkdmRnN0pi
  username: bXktYXBw
kind: Secret
metadata:
  name: test-secret
type: Opaque
```

### 4. Mount Secret as volume
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-test-pod
spec:
  containers:
  - name: test-container
    image: nginx
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secret-volume
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: test-secret
EOF
```
- Sample output
```bash
pod/secret-test-pod created
```

---

## II. Resource Requests and Limits
### 1. Define CPU and Memory limits for Pod
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
EOF
```
- Sample output
```bash
pod/frontend created
```

### 2. Observe pod scheduling based on resource request
```bash
kubectl describe pod frontend
```
- Sample output
```bash
Status: Running
Requests:
  cpu: 250m
  memory: 64Mi
Limits:
  cpu: 500m
  memory: 128Mi
```

---

## III. Container Probes and Restart Policy
### 1. Liveness probe using exec
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -f /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
EOF
```
- Sample output
```bash
pod/liveness-exec created
```

### 2. Liveness probe using HTTP
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: liveness-http
spec:
  containers:
  - name: liveness
    image: registry.k8s.io/liveness
    args: ["/server"]
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 3
      periodSeconds: 3
EOF
```
- Sample output
```bash
pod/liveness-http created
```

---

## IV. Multi-Container Pod
### 1. Create Pod with sidecar container
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  containers:
  - name: web
    image: alpine
    command: ['sh', '-c', 'echo "logging" > /opt/logs.txt; sleep 3600']
    volumeMounts:
      - name: data
        mountPath: /opt
  - name: logshipper
    image: alpine
    command: ['sh', '-c', 'tail -f /opt/logs.txt']
    volumeMounts:
      - name: data
        mountPath: /opt
  volumes:
    - name: data
      emptyDir: {}
EOF
```
- Sample output
```bash
pod/multi-container-pod created
```

---

## V. Init Container
### 1. Create Pod with Init Containers
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  initContainers:
  - name: init-myservice
    image: busybox
    command: ['sh', '-c', "until nslookup myservice.default.svc.cluster.local; do echo waiting for myservice; sleep 2; done"]
  - name: init-mydb
    image: busybox
    command: ['sh', '-c', "until nslookup mydb.default.svc.cluster.local; do echo waiting for mydb; sleep 2; done"]
  containers:
  - name: myapp
    image: busybox
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
EOF
```
- Sample output
```bash
pod/myapp-pod created
```

---

## VI. Practice Summary
### 1. Combined practice task
```bash
# Create ConfigMap and Secret
kubectl create configmap demo-config --from-literal=app_mode=production
kubectl create secret generic demo-secret --from-literal=db_user=admin --from-literal=db_pass=12345

# Apply Pod with resource limit and probe
kubectl apply -f frontend.yaml
kubectl apply -f liveness-exec.yaml
```
- Sample output
```bash
configmap/demo-config created
secret/demo-secret created
pod/frontend created
pod/liveness-exec created
```
