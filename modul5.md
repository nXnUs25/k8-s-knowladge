# Storage

## Volumens  

### Volumens ephemeral 

- the data is stored as long as the Pod exists
- in other words: when Pod is removed volumen too
- most often these are local volumes (data stored on the node where Pod is running)
    - EmptyDir
    - ConfigMap
    - Secret
    - CSI ephemeral volumes

- configMap
    - mounted to Pod as a file
    - SSD/HDD storage values

- secret
    - mounted to Pod as a file, but...
    - their values are stored in RAM (tmpfs)

- emptyDir
    - we allocate an empty volume - most often so that containers within one Pod can save data there
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: test-pd
    spec:
      containers: 
      - image: k8smaestro/web-app:1.0
        name: webapp-ctr
        volumeMounts:
        - mountPath: /cache
          name: cache-volume
      volumes:
      - name: cache-volume
        emptyDir: 
          medium: Memory
          sizeLimit: "1Gi"
    ```
    - data is usually saved on disk (HDD/SSD), but we can also use RAM (tmpfs)

- downwardAPI
    - we use information from `ApiServer {}` for the container's file system
        - information about Pod
        - information about yourself (container)
    - Pod labels will be saved in the `/etc/podinfo` file
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: test-pd
      labels:
        env: prod
        cluster: test1
    spec:
      containers:
      - image: k8smaestro/web-app:1.0
        name: webapp-ctr
        volumeMounts:
        - name: podinfo
          mountPath: /etc/pod-info
    volumes:
        - name: podinfo
          downwardAPI:
            items:
              - path: "labels"
                fileRef:
                  fieldPath: metadata.labels
    ```
- CSI ephemeral volumes
    - CSI specific driver https://kubernetes-csi.github.io/docs/drivers.html
    - they work similarly to configMap, secret, downwardAPI
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: csi-pod
    spec:
      containers:
      - image: k8smaestro/web-app:1.0
        name: webapp-ctr
        volumeMounts:
        - mountPath: /data
          name: my-csi-inline-vol
      volumes:
      - name: my-csi-inline-vol
          csi:
            driver: inline.storage.kubernetes.io
    ```

- hostPath
    - mount files from the container to the Node filesystem (aka: `docker -v file_system_path:/container_path`)
    - rather anti-pattern in K8's (try to avoid usage)
    - we use it only in specific situations
        - for example, to run monitoring tools
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: monitoring-pod
    spec:
      containers: 
      - image: google/cadvisor:v0.31.0
        name: cadvisor
        volumeMounts:
        - name: rootfs
          mountPath: /rootfs
          readOnly: true
        - name: var-run
          mountPath: /var/run
          readOnly: false
    volumes:
      - name: rootfs
        hostPath: 
          path: /
      - name: var-run
        hostPath: 
          path: /var/run
    ```


### Kubernetes Storage - requirements  

- cannot be dependent on the Pod's life cycle (because pods are volatile)
- should be available to all nodes in the cluster
- must survive even if the entire cluster fails

### Persistent Volumes

- object seen for the whole cluster
- is not assigned to a given Namespace (so we can share them)
- created by Yaml
    ```yaml
    apiVersion: v1
    metadata:
      name: ps-vol
    spec:
      accessModes:
      - ReadWriteOnce
      capacity:
        storage: 10Gi
      persistentVolumeReclaimPolicy: Retain
      gcePersistentDisk:
        pdName: ps-vol
    ```
- maps physical (usually external) storage to an object in Kubernetes

```
                         ---------- NFS, GlusterFS ....
        External        /
pv ---> FileSystem ----
                        \
                         \--------- AWS EBS, Azure Files, 
                                    GCE Persistent Disk
```


