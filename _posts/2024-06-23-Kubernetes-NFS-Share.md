---
title: Kubernetes NFS Mount Setup for Persistent Storage
description: Documentation on how to setup an NFS mount for persistent storage in any kubernetes deployment.
date: 2024-06-23 19:13:00 -0500
categories: [Kubernetes, NFS]
tags: [kubernetes, nfs, persistent storage]     # TAG names should always be lowercase
---

## Requirements
1. Install nfs-common on all kubernetes nodes
    ```
    sudo apt install -y nfs-common
    ```

2. Install [NFS CSI](https://github.com/kubernetes-csi/csi-driver-nfs) driver

3. An NFS share setup from something like TrueNAS, Synology, etc.


## Setup persistent volume
1. Create a yaml file called `nfs-pv.yaml` with the following:
    ```yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
    name: nfs
    spec:
    capacity:
        storage: 20Gi
    accessModes:
        - ReadWriteMany
    storageClassName: nfs
    nfs:
        server: 192.168.1.24            # IP of NFS Server
        path: "/mnt/TREPOOL/TRESHARE"   # NFS mount path 
    ```

2. Apply file
    ```
    kubectl apply -f nfs-pv.yaml
    ```

## Setup persistent volume claim
1. Create a yaml file called `nfs-pvc.yaml` with the following:
    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
    name: nfs
    spec:
    accessModes:
        - ReadWriteMany
    storageClassName: nfs
    resources:
        requests:
        storage: 20Gi
    ```

2. Apply file
    ```
    kubectl apply -f nfs-pvc.yaml
    ```

## Test with NGINX web server
1. Create a yaml file called `nfs-web.yaml` with the following:
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: nfs-web
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
            - name: web
                containerPort: 80
            volumeMounts:
            - name: nfs
                mountPath: /usr/share/nginx/html    # NFS path in pod
        volumes:
        - name: nfs
            persistentVolumeClaim:
            claimName: nfs
    ```

2. Apply file
    ```
    kubectl apply -f nfs-web.yaml
    ```

3. Get the name of the pod. eg: `nfs-web-7bc97bcf48-5brq6`
    ```
    kubectl get pods
    ```

4. Exec into the pod to verify nfs is mounted and files are available.
    ```
    kubectl exec -it nfs-web-7bc97bcf48-5brq6 -- /bin/bash
    ```

5. cd into NFS path from the deployment and list the directory to verify files.
    ```
    cd /usr/share/nginx/html
    ```