# CephFS

CephFS is used for mounting a persisting home directory into each login container. This requires:

- a CephFS volume (not filesystem!)
- a CephFS subvolume
- set of specific capabilities: 

```
mgr "allow rw"
osd "allow rwx tag cephfs metadata=$FS_NAME, allow rw tag cephfs data=$FS_NAME"
mds "allow r fsname=$FS_NAME path=/volumes, allow rws fsname=$FS_NAME path=/volumes/$SUB_VOL"
mon "allow r fsname=$FS_NAME"
```

Modify the file `cephfs/helm-cephfs-config.yaml` to match your setup. In specific, adjust the `clusterID`s, the `monitors`, the `fsName` (which is the volume name / `$FS_NAME`), and the `subvolumeGroup` (which is `$SUB_VOL`). Also adjust the `{admin,user}{ID,Key}` to match the user used for the capabilities above. 

Apply the Helm chart via:

```bash 
kubectl create namespace ceph-csi-cephfs
helm repo add ceph-csi https://ceph.github.io/csi-charts
helm install \
  ceph-csi-cephfs \
  ceph-csi/ceph-csi-cephfs \
  --namespace ceph-csi-cephfs \
  -f helm-cephfs-config.yaml
```

> Note: It may happen that not all pods of this deployment will launch due to port collisions of the Prometheus Liveness endpoint. Editing the configuration with `kubectl -n ceph-csi-cephfs edit ...` is a reasonably quick way to resolve this.
