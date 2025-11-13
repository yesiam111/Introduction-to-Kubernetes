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
kubectl get pods -l app=web
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
```
- Sample output
```bash
NAME        MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
nginx-pdb   2               N/A               1                     30s
```

## VII. Cleanup
### 1. Remove all created resources
```bash
kubectl delete deployment nginx-deployment
kubectl delete statefulset web
kubectl delete daemonset fluentd-elasticsearch
kubectl delete pdb nginx-pdb
```
- Sample output
```bash
deployment.apps "nginx-deployment" deleted
statefulset.apps "web" deleted
daemonset.apps "fluentd-elasticsearch" deleted
poddisruptionbudget.policy "nginx-pdb" deleted
```
