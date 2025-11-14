# Topic 1 - Kubernetes Storage

## I. Giới thiệu về cách lưu trữ dữ liệu của Pod trên Kubernetes
### 1. Tạo Pod sử dụng `emptyDir` volume
---
```bash
# Tạo file cấu hình pod sử dụng emptyDir volume
cat <<EOF > pod-emptydir.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-emptydir-demo
spec:
  containers:
  - name: app-container
    image: busybox
    command: ["sh", "-c", "echo Hello from container > /data/message && sleep 3600"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
  - name: sidecar
    image: busybox
    command: ["sh", "-c", "cat /data/message && sleep 3600"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
  volumes:
  - name: shared-data
    emptyDir: {}
EOF

# Áp dụng cấu hình
kubectl apply -f pod-emptydir.yaml

# Kiểm tra pod
kubectl get pods
```
## II. Ceph CSI integration sample step

### Links

* <https://docs.ceph.com/en/latest/rbd/rbd-kubernetes/>

### 1. Ceph CSI setup

#### Config preparation
```
cat <<EOF > csi-config-map.yaml
---
apiVersion: v1
kind: ConfigMap
data:
  config.json: |-
    [
      {
        "clusterID": "<ceph_cluster_id>",
        "monitors": [
          "<mon_1_ip>:6789",
          "<mon_2_ip>:6789",
          "<mon_3_ip>:6789"
        ]
      }
    ]
metadata:
  name: ceph-csi-config
  namespace: ceph-csi
EOF
```

```
kubectl apply -f csi-config-map.yaml
```

```
cat <<EOF > csi-kms-config-map.yaml
---
apiVersion: v1
kind: ConfigMap
data:
  config.json: |-
    {}
metadata:
  name: ceph-csi-encryption-kms-config
  namespace: ceph-csi
EOF

cat <<EOF > ceph-config-map.yaml
---
apiVersion: v1
kind: ConfigMap
data:
  ceph.conf: |
    [global]
    auth_cluster_required = cephx
    auth_service_required = cephx
    auth_client_required = cephx
  # keyring is a required key and its value should be empty
  keyring: |
metadata:
  name: ceph-config
  namespace: ceph-csi
EOF
```

```
kubectl apply -f csi-kms-config-map.yaml
kubectl apply -f ceph-config-map.yaml
```
#### Create CSI secret and config CEPH-CSI plugins
```
cat <<EOF > csi-rbd-secret.yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: csi-rbd-secret
  namespace: ceph-csi
stringData:
  userID: <ceph-user-id>
  userKey: <ceph-user-secret-key>
EOF
```

```
kubectl apply -f csi-rbd-secret.yaml
```

```
wget https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/csi-provisioner-rbac.yaml
sed -i.bak 's/namespace\:\ default/namespace\:\ ceph-csi/g' ./csi-provisioner-rbac.yaml
wget https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/csi-nodeplugin-rbac.yaml
sed -i.bak 's/namespace\:\ default/namespace\:\ ceph-csi/g' ./csi-nodeplugin-rbac.yaml
wget https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/csi-rbdplugin-provisioner.yaml
sed -i.bak 's/namespace\:\ default/namespace\:\ ceph-csi/g' ./csi-rbdplugin-provisioner.yaml
wget https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/csi-rbdplugin.yaml
sed -i.bak 's/namespace\:\ default/namespace\:\ ceph-csi/g' ./csi-rbdplugin.yaml
```

```
kubectl apply -f csi-provisioner-rbac.yaml
kubectl apply -f csi-nodeplugin-rbac.yaml
kubectl apply -f csi-rbdplugin-provisioner.yaml
kubectl apply -f csi-rbdplugin.yaml
```

#### Create storage class
```
cat <<EOF > csi-rbd-sc.yaml
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: csi-rbd-sc
provisioner: rbd.csi.ceph.com
parameters:
   clusterID: <ceph-cluster-id>
   pool: <ceph-rbd-pool-name>
   imageFeatures: layering
   csi.storage.k8s.io/provisioner-secret-name: csi-rbd-secret
   csi.storage.k8s.io/provisioner-secret-namespace: ceph-csi
   csi.storage.k8s.io/controller-expand-secret-name: csi-rbd-secret
   csi.storage.k8s.io/controller-expand-secret-namespace: ceph-csi
   csi.storage.k8s.io/node-stage-secret-name: csi-rbd-secret
   csi.storage.k8s.io/node-stage-secret-namespace: ceph-csi
reclaimPolicy: Delete
allowVolumeExpansion: true
mountOptions:
   - discard
EOF
```

```
kubectl apply -f csi-rbd-sc.yaml

kubectl get sc
```
### 2. Use Persistent storage
#### Create PVC
```
cat <<EOF > rbd-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rbd-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: csi-rbd-sc
  resources:
    requests:
      storage: 10Gi
EOF

kubectl apply -f rbd-pvc.yaml
kubectl get pvc
```

#### Use PVC in a pod
```
cat <<EOF > pod-rbd-pvc.yaml
apiVersion: v1
kind: Pod
metadata:
  name: rbd-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: rbd-vol
      mountPath: /data
  volumes:
  - name: rbd-vol
    persistentVolumeClaim:
      claimName: rbd-pvc
EOF

kubectl apply -f pod-rbd-pvc.yaml
kubectl get pod,pvc
kubectl exec -it rbd-pod -- df -h
```
- Sample output
```
NAME      READY   STATUS    RESTARTS   AGE
rbd-pod   1/1     Running   0          8s
NAME                                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS  VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/rbd-pvc         Bound    pvc-00c8e6da-cc31-4365-9ed0-c679dc1669ca   10Gi       RWO            csi-rbd-sc         <unset>           5m59s

Filesystem                Size      Used Available Use% Mounted on
.......
/dev/rbd0                 9.7G     24.0K      9.7G   0% /data
.......
```

#### Statefulset with dynamic provisioning
```
cat <<EOF > sts-rbd-pvc.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: rbd-ss
spec:
  serviceName: rbd-ss
  replicas: 2
  selector:
    matchLabels:
      app: rbd-ss
  template:
    metadata:
      labels:
        app: rbd-ss
    spec:
      containers:
      - name: app
        image: busybox
        command: ["sh", "-c", "sleep 3600"]
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: csi-rbd-sc
      resources:
        requests:
          storage: 10Gi
EOF

kubectl apply -f sts-rbd-pvc.yaml
kubectl get pod,pvc  ## note the number in PVC name
kubectl exec -it pod/rbd-ss-0 -- df -h
```
- Sample output
```
NAME           READY   STATUS    RESTARTS   AGE
pod/rbd-ss-0   1/1     Running   0          2m24s
pod/rbd-ss-1   1/1     Running   0          2m18s

NAME                                  STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS
persistentvolumeclaim/data-rbd-ss-0   Bound    pvc-dcb5b572-2aca-44bc-b1cb-a2b1a5f4d610   10Gi       RWO            csi-rbd-sc 
persistentvolumeclaim/data-rbd-ss-1   Bound    pvc-452b5861-6d60-49ff-8c22-2c9cba1b6292   10Gi       RWO            csi-rbd-sc
```

### 3. Ceph CSI Snapshot
#### Install additional K8S CSI component
```
git clone https://github.com/ceph/ceph-csi.git -b v3.14.1
cd ceph-csi
./scripts/install-snapshot.sh install
```

#### Create snapshot class
```
cat <<EOF > snapshotclass.yaml
---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-rbdplugin-snapclass
driver: rbd.csi.ceph.com
parameters:
  clusterID: <cluster-id>
  csi.storage.k8s.io/snapshotter-secret-name: csi-rbd-secret
  csi.storage.k8s.io/snapshotter-secret-namespace: <ceph-csi-namespace>
deletionPolicy: Delete
EOF
kubectl create -f snapshotclass.yaml
```

#### list snapshotclass
```
kubectl get volumesnapshotclass
```

#### create snapshot
```
cat <<EOF > snapshot.yaml
---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: rbd-pvc-snapshot
spec:
  volumeSnapshotClassName: csi-rbdplugin-snapclass
  source:
    persistentVolumeClaimName: rbd-pvc
EOF

kubectl create -f snapshot.yaml

kubectl get volumesnapshot
```
