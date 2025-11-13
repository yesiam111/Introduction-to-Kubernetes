# Topic 4 - Kubernetes Pod and Container

## I. ConfigMap and Secret
### 1. Prepare configuration files
```bash
# Create a folder for config files
mkdir -p ./configmap-files

# Create a user-interface.properties file
cat <<EOF > ./configmap-files/user-interface.properties
color.good=purple
color.bad=yellow
allow.textmode=true
EOF

# Create another configuration file
cat <<EOF > ./configmap-files/game.properties
enemy.types=aliens,monsters
player.maximum-lives=5
EOF

# Verify the files
ls ./configmap-files
cat ./configmap-files/user-interface.properties
```
- Sample output
```bash
game.properties  user-interface.properties
color.good=purple
color.bad=yellow
allow.textmode=true
```

### 2. Create ConfigMap from files and literals
```bash
kubectl create configmap game-demo \
  --from-literal=player_initial_lives=3 \
  --from-file=./configmap-files/game.properties \
  --from-file=./configmap-files/user-interface.properties

# Inspect ConfigMap
kubectl get configmap game-demo -o yaml
```
- Sample output
```yaml
apiVersion: v1
data:
  player_initial_lives: "3"
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
kind: ConfigMap
metadata:
  name: game-demo
```

### 3. Inject ConfigMap into Pod
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

kubectl wait --for=condition=Ready pod/configmap-demo-pod
kubectl exec -it configmap-demo-pod -- sh
```
- Sample output
```bash
pod/configmap-demo-pod created
pod/configmap-demo-pod condition met
/ # echo $PLAYER_INITIAL_LIVES
3
/ # ls /config
game.properties  user-interface.properties
/ # cat /config/user-interface.properties
color.good=purple
color.bad=yellow
allow.textmode=true
```

---

### 4. Create and use Secret
```bash
kubectl create secret generic test-secret \
  --from-literal=username=myapp \
  --from-literal=password=secretpass

kubectl get secret test-secret -o yaml
```
- Sample output
```yaml
apiVersion: v1
data:
  password: c2VjcmV0cGFzcw==
  username: bXlhcHA=
kind: Secret
metadata:
  name: test-secret
type: Opaque
```

### 5. Inject Secret into Pod
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-demo-pod
spec:
  containers:
  - name: demo
    image: alpine
    command: ["sleep", "3600"]
    env:
    - name: USERNAME
      valueFrom:
        secretKeyRef:
          name: test-secret
          key: username
    - name: PASSWORD
      valueFrom:
        secretKeyRef:
          name: test-secret
          key: password
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secret-volume
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: test-secret
EOF

kubectl wait --for=condition=Ready pod/secret-demo-pod
kubectl exec -it secret-demo-pod -- sh
```
- Sample output
```bash
/ # echo $USERNAME
myapp
/ # echo $PASSWORD
secretpass
/ # ls /etc/secret-volume
password  username
/ # cat /etc/secret-volume/username
myapp
```

---

## II. Resource Requests and Limits
### 1. Define CPU and Memory limits
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "while true; do echo running; sleep 5; done"]
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
EOF

kubectl describe pod resource-demo
```
- Sample output
```bash
Requests:
  cpu: 250m
  memory: 64Mi
Limits:
  cpu: 500m
  memory: 128Mi
Status: Running
```

---

## III. Container Probes
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

kubectl get pod liveness-exec -w
```
- Sample output
```bash
NAME             READY   STATUS    RESTARTS   AGE
liveness-exec    1/1     Running   0          10s
liveness-exec    1/1     Running   1          40s   # restarted after probe failed
```

---

## IV. Multi-Container Pod
### 1. Pod with sidecar container
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  containers:
  - name: writer
    image: busybox
    command: ['sh', '-c', 'while true; do echo $(date) >> /opt/logs.txt; sleep 2; done']
    volumeMounts:
      - name: data
        mountPath: /opt
  - name: reader
    image: busybox
    command: ['sh', '-c', 'tail -f /opt/logs.txt']
    volumeMounts:
      - name: data
        mountPath: /opt
  volumes:
    - name: data
      emptyDir: {}
EOF

kubectl exec -it multi-container-pod -c reader -- sh
```
- Sample output
```bash
/ # tail -f /opt/logs.txt
Wed Nov 12 14:01:32 UTC 2025
Wed Nov 12 14:01:34 UTC 2025
Wed Nov 12 14:01:36 UTC 2025
```

---

## V. Init Container
### 1. Pod with Init Containers
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: init-demo-pod
spec:
  initContainers:
  - name: init-wait
    image: busybox
    command: ['sh', '-c', 'echo Init container running; sleep 5']
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'echo App started; sleep 3600']
EOF

kubectl get pod init-demo-pod -w

```
- Sample output
```bash
NAME            READY   STATUS     RESTARTS   AGE
init-demo-pod   0/1     Init:0/1   0          2s
init-demo-pod   0/1     Init:Completed   0    7s
init-demo-pod   1/1     Running         0    10s
```

---

## VI. Practice Summary
### 1. Combined verification
```bash
kubectl get pod,configmap,secret
kubectl exec -it configmap-demo-pod -- sh -c "env | grep PLAYER"
kubectl exec -it secret-demo-pod -- sh -c "ls /etc/secret-volume"
```
- Sample output
```bash
NAME                  READY   STATUS    RESTARTS   AGE
configmap-demo-pod    1/1     Running   0          3m
secret-demo-pod       1/1     Running   0          2m
resource-demo         1/1     Running   0          2m
multi-container-pod   2/2     Running   0          2m
init-demo-pod         1/1     Running   0          1m
NAME            DATA   AGE
configmap/game-demo   3      3m
NAME            TYPE     DATA   AGE
secret/test-secret   Opaque   2      3m
```

---

## VII. Cleanup
### 1. Delete all created resources
```bash
kubectl delete pod configmap-demo-pod secret-demo-pod resource-demo \
  liveness-exec multi-container-pod init-demo-pod
kubectl delete configmap game-demo
kubectl delete secret test-secret
rm -rf ./configmap-files
```
- Sample output
```bash
pod "configmap-demo-pod" deleted
pod "secret-demo-pod" deleted
pod "resource-demo" deleted
pod "liveness-exec" deleted
pod "multi-container-pod" deleted
pod "init-demo-pod" deleted
configmap "game-demo" deleted
secret "test-secret" deleted
```
