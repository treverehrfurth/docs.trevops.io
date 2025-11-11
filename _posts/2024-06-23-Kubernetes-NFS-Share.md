---
title: Kubernetes NFS Mount Setup for Persistent Storage
description: Documentation on how to setup an NFS mount for persistent storage in any kubernetes deployment.
date: 2024-06-23 19:13:00 -0500
categories: [Kubernetes, NFS]
tags: [kubernetes, nfs, persistent storage]     # TAG names should always be lowercase
---

## Requirements

1. Install `nfs-common` on all Kubernetes nodes
   ```bash
   sudo apt install -y nfs-common
   ```

2. Install the **NFS CSI** driver (follow upstream for your distro/cluster)
   - https://github.com/kubernetes-csi/csi-driver-nfs

3. An existing NFS share from TrueNAS/Synology/etc.

---

## Setup

### 1) PersistentVolume — `nfs-pv.yaml`

> Static PV that points directly at your NFS export.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs
spec:
  capacity:
    storage: 200Gi         # adjust size for your needs
  accessModes:
    - ReadWriteMany
  storageClassName: nfs
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: 192.168.1.24            # IP of NFS Server
    path: "/mnt/POOL/SHARE"         # NFS mount path
```

**Apply:**
```bash
kubectl apply -f nfs-pv.yaml
```

---

### 2) PersistentVolumeClaim — `nfs-pvc.yaml`

> Binds to the `nfs` StorageClass and requests capacity from the PV above.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs
  namespace: default
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs
  resources:
    requests:
      storage: 20Gi
```

**Apply:**
```bash
kubectl apply -f nfs-pvc.yaml
```

---

### 3) Test with NGINX web server — `nfs-web.yaml`

> Mounts the PVC at `/usr/share/nginx/html` in an `nginx` Deployment (same as your flow).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-web
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-web
  template:
    metadata:
      labels:
        app: nfs-web
    spec:
      containers:
        - name: nfs-web
          image: nginx
          ports:
            - containerPort: 80
          volumeMounts:
            - name: site
              mountPath: /usr/share/nginx/html
      volumes:
        - name: site
          persistentVolumeClaim:
            claimName: nfs
```

**Apply:**
```bash
kubectl apply -f nfs-web.yaml
```

---

## Validation

1) Get the pod name (e.g. `nfs-web-xxxxxxxxxx-xxxxx`):
```bash
kubectl get pods
```

2) Exec into the pod:
```bash
kubectl exec -it <your-pod-name> -- /bin/bash
```

3) Check the mounted path and verify the NFS mount and files are available:
```bash
cd /usr/share/nginx/html
ls -la
```

---

## Cleanup (Optional)

```bash
kubectl delete deployment nfs-web -n default
kubectl delete pvc nfs -n default
kubectl delete pv nfs
```