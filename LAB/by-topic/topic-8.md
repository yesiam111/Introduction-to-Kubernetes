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
