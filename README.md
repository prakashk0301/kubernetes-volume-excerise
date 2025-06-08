# Using AWS EBS (Elastic Block Store) Volumes in Kubernetes

## ✅ Use Case
**Stateful workloads** (e.g., databases) that need persistent storage.

- Kubernetes can dynamically provision EBS volumes using the **aws-ebs-csi-driver**.
- These volumes are attached to the node where the pod runs, and move with the pod if rescheduled.
- You use a **StorageClass + PVC**, and Kubernetes handles the rest.

---

## 🧱 Static Provisioning with Pre-created EBS Volume

If you want to create the EBS volume manually (e.g., via AWS Console), note the **volume ID** and **availability zone**.

### 📝 `pv-static.yaml`
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ebs-pv-static
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  csi:
    driver: ebs.csi.aws.com
    volumeHandle: vol-xxxxxxxx  # Replace with actual EBS volume ID
    fsType: ext4
```

### 📝 `pvc-static.yaml`
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-pvc-static
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  volumeName: ebs-pv-static
  storageClassName: manual
```

### 📝 `pod.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "while true; do sleep 10; done"]
    volumeMounts:
    - mountPath: /data
      name: ebs-volume
  volumes:
  - name: ebs-volume
    persistentVolumeClaim:
      claimName: ebs-pvc
```

---

## ⚙️ Dynamic Provisioning of Volume

Install the EBS CSI driver:  
🔗 [aws-ebs-csi-driver GitHub](https://github.com/kubernetes-sigs/aws-ebs-csi-driver)

### 📝 `storageclass.yaml`
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
```

### 📝 `pvc.yaml`
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: ebs-sc
```

### 📝 `pod.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "while true; do sleep 10; done"]
    volumeMounts:
    - mountPath: /data
      name: ebs-volume
  volumes:
  - name: ebs-volume
    persistentVolumeClaim:
      claimName: ebs-pvc
```

---

## ✅ Summary

| Type             | Provisioning | Manual EBS Volume | Automation Level | Access Mode  |
|------------------|--------------|-------------------|------------------|--------------|
| Static           | Manual       | Yes               | Low              | ReadWriteOnce |
| Dynamic          | Automatic    | No                | High             | ReadWriteOnce |



# Using Amazon EFS with Kubernetes for ReadWriteMany Access

Amazon EFS (Elastic File System) provides support for the `ReadWriteMany` (RWX) access mode, making it ideal for workloads where multiple pods across nodes need to read and write concurrently to a shared volume. Unlike AWS EBS, which only supports `ReadWriteOnce`, EFS is well-suited for shared access.

## ✅ Prerequisites

Before using EFS volumes with Kubernetes, you must install the EFS CSI Driver:

```bash
kubectl apply -k "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/ecr/?ref=release-1.7"
```

Ensure you have an EFS filesystem created in AWS and that it's accessible from your Kubernetes worker nodes (VPC, subnets, and security groups configured correctly).

---

## 📄 YAML Manifests

### `pv.yaml` – PersistentVolume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: efs-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: efs-sc
  csi:
    driver: efs.csi.aws.com
    volumeHandle: fs-12345678  # Replace with your EFS FileSystem ID
```

---

### `pvc.yaml` – PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
```

---

### `deployment.yaml` – Deployment Using EFS Volume

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: efs-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: efs-demo
  template:
    metadata:
      labels:
        app: efs-demo
    spec:
      containers:
      - name: web
        image: busybox
        command: ["sh", "-c", "while true; do echo $(hostname) >> /data/out.txt; sleep 5; done"]
        volumeMounts:
        - name: efs-volume
          mountPath: /data
      volumes:
      - name: efs-volume
        persistentVolumeClaim:
          claimName: efs-pvc
```

---

## 🧪 Verification

Once deployed, you can verify the shared volume is working as expected by checking the contents of `/data/out.txt` from both pod replicas. They should all write to the same file due to shared access via EFS.
