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
kubectl -n <namespace> get pods
kubectl -n <namespace> describe pod <pod-name>
kubectl -n <namespace> delete pod <pod-name>
kubectl -n <namespace> apply -f deployment.yaml
```
- Sample output
```
NAME            READY   STATUS    RESTARTS   AGE
web-5c8d6d7b7c  1/1     Running   0          3m
```

## III. Pod Interaction
### 1. Execute commands, view logs, copy files, and port-forward
```
kubectl exec -it <pod-name> -- bash
kubectl logs <pod-name>
kubectl cp <pod-name>:/app/config.yaml ./config.yaml
kubectl port-forward <pod-name> 8080:80
```
- Sample output
```
root@pod:/# ls /app
config.yaml  main.py
```

## IV. Management Tools
### 1. Use additional management utilities
```
# CLI-based
k9s
kubectl get pods -A

# GUI-based
# Lens or Rancher UI
```
- Sample output
```
Context: dev-cluster
Namespace: default
Pods: 4 running
```

## V. High Availability (HA)
### 1. Understand HA deployment models
```
# Control Plane and etcd on same node
# Control Plane and etcd on separate nodes
```
- Sample output
```
ClusterStatus:
  controlPlaneNodes: 3
  etcdNodes: 3
  status: Healthy
```

## VI. Node Maintenance - Drain and Uncordon
### 1. Safely remove a node from scheduling and restore it later
```
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node-name>
```
- Sample output
```
node/ip-172-31-20-150 cordoned
evicting pod "web-5d6bdfc8c5-9vjgk"
node/ip-172-31-20-150 uncordoned
```

## VII. Add and Remove Nodes
### 1. Add a new node to the cluster
```
# On control plane node
kubeadm token create
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | \
openssl rsa -pubin -outform der 2>/dev/null | \
openssl dgst -sha256 -hex | sed 's/^.* //'

# On the new node
kubeadm join <control-plane-host>:6443 --token <token> \
--discovery-token-ca-cert-hash sha256:<hash>
```
- Sample output
```
This node has joined the cluster
Successfully established connection to control-plane
```

### 2. Remove a node from the cluster
```
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
kubectl delete node <node-name>
```
- Sample output
```
node/ip-172-31-25-220 drained
node "ip-172-31-25-220" deleted
```

## VIII. Backup and Restore etcd
### 1. Backup etcd using snapshot
```
ETCDCTL_API=3 etcdctl --endpoints <ENDPOINT> snapshot save /backup/etcd-snapshot.db
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

### 2. Restore etcd from snapshot
```
systemctl stop kube-apiserver kube-controller-manager kube-scheduler etcd
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir /var/lib/etcd-restored
mv /var/lib/etcd /var/lib/etcd-old && mv /var/lib/etcd-restored /var/lib/etcd
systemctl start etcd kube-apiserver kube-controller-manager kube-scheduler
```
- Sample output
```
Snapshot restored successfully
Cluster bootstrapped with revision 2384
```
