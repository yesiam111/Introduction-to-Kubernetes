# Topic 2 - Kubernetes Basic Management Tasks

## I. Context Management
### 1. View and switch between Kubernetes contexts
```
kubectl config get-contexts
kubectl config current-context
kubectl config use-context <context-name>
```
- Sample output
```
CURRENT   NAME             CLUSTER          AUTHINFO           NAMESPACE
*         dev-cluster      dev-cluster      dev-user
          prod-cluster     prod-cluster     prod-user
```

## II. Namespace Operations
### 1. Manage resources within specific namespaces
```
kubectl -n demo-ns get pods
kubectl -n demo-ns describe pod <pod-name>
kubectl -n demo-ns delete pod <pod-name>
kubectl -n demo-ns apply -f deployment.yaml
```
- Sample output
```
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-647677fc66-hsbks   1/1     Running   0          14s
```

## III. Pod Interaction
### 1. Execute commands, view logs, copy files, and port-forward
- Demo with any container
```
kubectl exec -it <pod-name> -- bash
kubectl logs <pod-name>
kubectl cp <pod-name>:/app/config.yaml ./config.yaml
kubectl port-forward <pod-name> 8080:80
```


## VI. Node Maintenance - Drain and Uncordon
### 1. Safely remove a node from scheduling and restore it later
```
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```
- Sample output
```
Warning: ignoring DaemonSet-managed Pods: kube-system/calico-node-fj222, kube-system/kube-proxy-2n6cc
evicting pod demo-ns/nginx-deployment-647677fc66-hsbks
pod/nginx-deployment-647677fc66-hsbks evicted
node/k8s-02 drained
```
### 2. Allow node to take workload
```
kubectl uncordon <node-name>
```

## VII. Add and Remove Nodes
### 1. Add a new node to the cluster
```
# On control plane node
kubeadm token create
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | \
openssl rsa -pubin -outform der 2>/dev/null | \
openssl dgst -sha256 -hex | sed 's/^.* //'

# Or
kubeadm token create --print-join-commandd

###

# On the new node
kubeadm join <control-plane-host>:6443 --token <token> \
--discovery-token-ca-cert-hash sha256:<hash> 
```

### 2. Remove a node from the cluster
```
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
kubectl delete node <node-name>
```

## VIII. Backup and Restore etcd
### 1. Create pod
```
kubectl run nginx --image=nginx --labels="env=staging,tier=frontend"
kubectl run nginx-prod --image=nginx --labels="env=prod,tier=frontend"
kubectl get pods --show-labels
```

### 2. Backup etcd using snapshot
```
mkdir /backup
ETCDCTL_API=3 etcdctl --endpoints https://<etcd ip>:2379 --cert /etc/kubernetes/pki/etcd/peer.crt --key /etc/kubernetes/pki/etcd/peer.key --cacert /etc/kubernetes/pki/etcd/ca.crt snapshot save /backup/etcd-snapshot.db
ETCDCTL_API=3 etcdctl --write-out=table snapshot status /backup/etcd-snapshot.db
```
- Sample output
```
+----------+----------+------------+------------+
|  HASH    | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| 3a1d8f3f |     2384 |       1223 |     1.8 MB |
+----------+----------+------------+------------+
```

- If etcd client not found: `apt install etcd-client -y`

### 2. Delete pod
```
kubectl delete pod nginx nginx-prod
kubectl get pod
## nginx and ngix-prod are deleted
```

### 3. Restore etcd from snapshot
```
systemctl stop kubelet docker
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir /var/lib/etcd-restored
mv /var/lib/etcd /var/lib/etcd-old && mv /var/lib/etcd-restored /var/lib/etcd
systemctl start kubelet docker
```

### 4. Get pod list
```
kubectl get pod
## should see nginx and nginx-prod pod if etcd restored correctly
```

### 5. Cleanup
```
kubectl delete pod nginx nginx-prod --force
```
