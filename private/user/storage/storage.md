# Storage

By default, no persistent storage is attached to Pods. For durable data storage, you can mount a [Persistent Volume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/).

OpenCUBE uses CephFS as a persistent storage provider. Every OpenCUBE account starts with a CephFS-based persistent volume intended for storing the users home directory. Additional persistent volumes can be requested.

You can mount your home directory to any pod using the Persistent Volume Claim (PVC) associated with it. Check it via:

```bash
$ kubectl get pvc
NAME                STATUS   VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS    VOLUMEATTRIBUTESCLASS   AGE
pvc-home-directory  Bound    pvc-$VOLUME   64Gi       RWX            csi-cephfs-sc   <unset>                 
```

You can use it within your pods as follows:

```yaml
apiVersion: v1
kind: Pod
metadata:
    name: test
spec:
  containers:
  - name: test
    image: harbor.pt.horizon-opencube.eu/baseimages/rt-baseimage:latest
    volumeMounts:
    - name: home
      mountPath: /home
  volumes:
  - name: home
    persistentVolumeClaim:
      claimName: pvc-home-directory
```

## Permissions

The `pvc-home-directory` mount contains data that is also visible in your login container. As such, file permissions need to be handled with some care. In your login container, you run under a cluster-managed UID/GID, whereas in your custom containers you may choose your UID/GID freely (assuming non-root). 

There are steps to address these permissions: (1) set an appropriate `securityContext`, and optionally (2) use the `rt-baseimage` and derivates for correctly resolving UIDs.

### Security Context

Kubernetes uses [Security Contexts](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/) which govern privilege and ACL settings. They can be set for specific containers and pod-wide. The file permission-relevant settings are:

- `runAsUser`
- `runAsGroup`
- `fsGroup`

You may set these either to an arbitrary ID (e.g. 1000), or to your assigned ID. You can determine the latter by running `id` in your login container:

```bash
user@containerssh-user-99sdw:~> id
uid=12345(user) gid=12345(user) groups=12345(user)
```

Setting `fsGroup` performs additional operations: All files created in that pod will be created with the GID specified in `fsGroup`. Additionally, all files mounted via persistent volumes will be converted to be owned by group `fsGroup`. Setting `fsGroup` to `runAsGroup` therefore ensures that no matter the ID, you can always access the files mounted to your pod.

### UID Resolving

The `rt-baseimage` image as well as the login-container image use `kanidm-unixd-client`, which integrates the IDM tool used at OpenCUBE (Kanidm). Specifically, it integrates Kanidm into [NSS](https://www.man7.org/linux/man-pages/man5/nss.5.html). As a consequence, raw UID/GIDs are resolved against the IDM. Both images come with a pre-configured setup. 

Should you decide to not use these base-images and do not want to integrate `kanidm-unixd-client` yourself, then you will only see raw UID/GIDs.