# Topic 5 - Kubernetes Workloads and Scheduling

## Before the lab
### 1. Install helm
```
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4 | bash
## DO NOT DO THIS ON PRODUCTION ENV
## ALWAYS CHECK THE SCRIPT CONTENT BEFORE EXECUTION
```
### 2. Install local disk provisioning
```
helm repo add openebs https://openebs.github.io/openebs
helm repo update
helm install openebs --namespace openebs openebs/openebs --set engines.replicated.mayastor.enabled=false --create-namespace
kubectl patch storageclass openebs-hostpath -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

```

### 3. Verify storage class
```
kubectl get sc
```
- Sample output:
```
NAME                         PROVISIONER        RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
openebs-hostpath (default)   openebs.io/local   Delete          WaitForFirstConsumer   false                  23m
openebs-loki-localpv         openebs.io/local   Delete          WaitForFirstConsumer   false                  23m
openebs-minio-localpv        openebs.io/local   Delete          WaitForFirstConsumer   false                  23m
```
## I. Deployment
### 1. Create a simple Nginx Deployment
```bash
kubectl create deployment nginx-deployment --image=nginx:latest
kubectl get deployments
kubectl get pods -l app=nginx-deployment
```
- Sample output
```bash
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   1/1     1            1           10s

NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-6f9c9d8f8d-abcde   1/1     Running   0          10s
```

## II. StatefulSet
### 1. Deploy a StatefulSet with storage persistence
```bash
kubectl apply -f https://k8s.io/examples/application/web/web.yaml
kubectl get statefulsets
kubectl get pods -l app=nginx
kubectl get pvc
```
- Sample output
```bash
NAME   READY   AGE
web    3/3     1m

NAME    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
www-web-0   Bound    pvc-12345a67-bc89-4e0d-b2ab-0cde12345fgh   1Gi        RWO       openebs-hostpath       1m
```

## III. DaemonSet
### 1. Run a Pod on all nodes using DaemonSet
```bash
kubectl apply -f https://k8s.io/examples/controllers/daemonset.yaml
kubectl -n kube-system get daemonsets | grep -E "fluentd|NAME"
kubectl -n kube-system get pods -o wide | grep -E "fluentd|NAME"
```
- Sample output
```bash
NAME                DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
fluentd-elasticsearch   3         3         3       3            3           <none>          2m

NAME                          READY   STATUS    NODE
fluentd-elasticsearch-abc12   1/1     Running   node1
fluentd-elasticsearch-def34   1/1     Running   node2
```

## IV. Scaling Workloads
### 1. Scale a Deployment up and down
```bash
kubectl scale deployment/nginx-deployment --replicas=5
kubectl get pods -l app=nginx-deployment
kubectl scale deployment/nginx-deployment --replicas=2
kubectl get pods -l app=nginx-deployment
```
- Sample output
```bash
deployment.apps/nginx-deployment scaled
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-6f9c9d8f8d-a1b2c   1/1     Running   0          20s
nginx-deployment-6f9c9d8f8d-d3e4f   1/1     Running   0          20s
```

### 2. Configure Horizontal Pod Autoscaler
 * Refer to [6.config-hpa.md](../6.config-hpa.md) to config HPA first
 * Add resource request and limit to deployment. HPA won't activate without resource request
```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
EOF
```

```bash
kubectl autoscale deployment nginx-deployment --min=2 --max=10 --cpu-percent=80
kubectl get hpa
```
- Sample output
```bash
NAME               REFERENCE                     TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
nginx-deployment   Deployment/nginx-deployment   0%/80%    2         10        2          15s
```

## V. Rolling Update
### 1. Perform rolling update on Deployment
```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.25.1
kubectl rollout status deployment/nginx-deployment
kubectl rollout history deployment/nginx-deployment
```
- Sample output
```bash
deployment "nginx-deployment" successfully rolled out
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl set image deployment/nginx-deployment nginx=nginx:1.25.1
```

### 2. Rollback Deployment version
```bash
kubectl rollout undo deployment/nginx-deployment --to-revision=1
kubectl rollout status deployment/nginx-deployment
```
- Sample output
```bash
deployment.apps/nginx-deployment rolled back
deployment "nginx-deployment" successfully rolled out
```

## VI. Pod Disruption Budget
### 1. Create a PDB for Deployment
```bash
cat <<EOF | kubectl apply -f -
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: nginx-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: nginx-deployment
EOF

kubectl get pdb
kubectl describe pdb nginx-pdb
```
- Sample output
```bash
NAME        MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
nginx-pdb   2               N/A               1                     30s

Name:         nginx-pdb
Namespace:    default
Min available: 2
Selector:     app=nginx-deployment
Allowed disruptions: 1
Current:      3
Desired:      2
```

### 2. Simulate disruption (drain node)
```bash
kubectl get nodes
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
kubectl get pods -l app=nginx-deployment
kubectl describe pdb nginx-pdb
```
- Sample output
```bash
node/<node-name> cordoned
evicting pod "nginx-deployment-7cdbd8f8b5-abc12"
cannot evict pod as it would violate the pod's disruption budget.

Name:         nginx-pdb
Min available: 2
Allowed disruptions: 0
Disruptions allowed: 0
Current healthy: 2
```

### 3. Restore normal state
```bash
kubectl uncordon <node-name>
kubectl get pods -l app=nginx-deployment
kubectl describe pdb nginx-pdb
```
- Sample output
```bash
node/<node-name> uncordoned
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-7cdbd8f8b5-abc12   1/1     Running   0          5m
nginx-deployment-7cdbd8f8b5-def34   1/1     Running   0          5m
nginx-deployment-7cdbd8f8b5-ghi56   1/1     Running   0          5m

Allowed disruptions: 1
Current healthy: 3
```

## VII. Cleanup
### 1. Remove all created resources
```bash
kubectl delete deployment nginx-deployment
kubectl delete statefulset web
kubectl -n kube-system delete daemonset fluentd-elasticsearch
kubectl delete pdb nginx-pdb
 kubectl delete hpa nginx-deployment
```
- Sample output
```bash
deployment.apps "nginx-deployment" deleted
statefulset.apps "web" deleted
daemonset.apps "fluentd-elasticsearch" deleted
poddisruptionbudget.policy "nginx-pdb" deleted
horizontalpodautoscaler.autoscaling "nginx-deployment" deleted
```
----
# Extra lab
## VIII. Taints and Tolerations
### 1. Add a taint to a specific node
```bash
kubectl get nodes
kubectl taint nodes <node-name> key1=value1:NoSchedule
kubectl describe node <node-name> | grep Taint
```
- Sample output
```bash
NAME       STATUS   ROLES    AGE   VERSION
node1      Ready    worker   10d   v1.30.2
node2      Ready    worker   10d   v1.30.2

Taints: key1=value1:NoSchedule
```

### 2. Deploy a **Deployment** without toleration
```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-no-toleration
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-no-toleration
  template:
    metadata:
      labels:
        app: nginx-no-toleration
    spec:
      containers:
      - name: nginx
        image: nginx
EOF

kubectl get pods -o wide
```
- Sample output
```bash
NAME                                   READY   STATUS    RESTARTS   AGE   NODE
nginx-no-toleration-7cdbd8f8b5-abc12   0/1     Pending   0          15s   <none>
```

### 3. Deploy a **Deployment** with matching toleration
```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-with-toleration
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-with-toleration
  template:
    metadata:
      labels:
        app: nginx-with-toleration
    spec:
      tolerations:
      - key: "key1"
        operator: "Equal"
        value: "value1"
        effect: "NoSchedule"
      containers:
      - name: nginx
        image: nginx
EOF

kubectl get pods -o wide
```
- Sample output
```bash
NAME                                   READY   STATUS    RESTARTS   AGE   NODE
nginx-with-toleration-7cdbd8f8b5-abc1  1/1     Running   0          20s   node1
```

### 4. Remove taint from node
```bash
kubectl taint nodes <node-name> key1=value1:NoSchedule-
kubectl describe node <node-name> | grep Taint
```
- Sample output
```bash
Taints: <none>
```

---

## IX. Assigning Workloads to Nodes

### 1. Node Selector (using Deployment)
```bash
kubectl label nodes <node-name> disktype=ssd
kubectl get nodes --show-labels
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-node-selector
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-node-selector
  template:
    metadata:
      labels:
        app: nginx-node-selector
    spec:
      nodeSelector:
        disktype: ssd
      containers:
      - name: nginx
        image: nginx
EOF

kubectl get pods -o wide
```
- Sample output
```bash
NAME                                   READY   STATUS    RESTARTS   AGE   NODE
nginx-node-selector-7cdbd8f8b5-abc12   1/1     Running   0          25s   node1
```

---

### 2. Node Affinity (using StatefulSet)
```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web-affinity
spec:
  serviceName: "web-affinity"
  replicas: 3
  selector:
    matchLabels:
      app: web-affinity
  template:
    metadata:
      labels:
        app: web-affinity
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: disktype
                operator: In
                values:
                - ssd
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
EOF

kubectl get pods -o wide
```
- Sample output
```bash
NAME             READY   STATUS    RESTARTS   AGE   NODE
web-affinity-0   1/1     Running   0          15s   node1
web-affinity-1   1/1     Running   0          15s   node1
web-affinity-2   1/1     Running   0          15s   node1
```

---

### 3. Pod Affinity (using Deployment)
```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-pod-affinity
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: nginx
        image: nginx
EOF

kubectl get pods -o wide
```
- Sample output
```bash
NAME                                   READY   STATUS    RESTARTS   AGE   NODE
nginx-pod-affinity-7cdbd8f8b5-abc1     1/1     Running   0          20s   node1
nginx-pod-affinity-7cdbd8f8b5-def2     1/1     Running   0          20s   node1
nginx-pod-affinity-7cdbd8f8b5-ghi3     1/1     Running   0          20s   node1
```

---

### 4. Pod Anti-Affinity (using Deployment)
```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-anti-affinity
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-aa
  template:
    metadata:
      labels:
        app: nginx-aa
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - nginx-aa
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: nginx
        image: nginx
EOF

kubectl get pods -o wide
```
- Sample output
```bash
NAME                                 READY   STATUS    RESTARTS   AGE   NODE
nginx-anti-affinity-75f8d6d77c-abc1  1/1     Running   0          20s   node1
nginx-anti-affinity-75f8d6d77c-def2  1/1     Running   0          20s   node2
nginx-anti-affinity-75f8d6d77c-ghi3  1/1     Running   0          20s   node3
```

---

## X. Cleanup
```bash
kubectl delete deployment nginx-no-toleration nginx-with-toleration nginx-node-selector nginx-pod-affinity nginx-anti-affinity
kubectl delete statefulset web-affinity
kubectl label nodes <node-name> disktype-
```
- Sample output
```bash
deployment.apps "nginx-no-toleration" deleted
deployment.apps "nginx-with-toleration" deleted
deployment.apps "nginx-node-selector" deleted
deployment.apps "nginx-pod-affinity" deleted
deployment.apps "nginx-anti-affinity" deleted
statefulset.apps "web-affinity" deleted
node/<node-name> unlabeled
```
