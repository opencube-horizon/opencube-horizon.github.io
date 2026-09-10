# Deploying OpenFAM

An OpenFAM cluster can be deployed per-user without special privileges. The OpenFAM cluster deployment uses a Volcano Job, which creates several types of pods:

|Number| Function 					      |
|------|--------------------------|
|1     | OpenFAM service launcher |
|1     | CIS server					      |
|N     | Memory server            |
|M     | Metadata server          |

The amount of memory and metadata service pods can be configured in the deployment script. 

The default deployment deploys:

- 1x metadata service, 16GB shared memory,
- 2x memory servers, 32GB shared memory each.

See "Adjust Cluster resources" for adjusting these defaults.

## Deployment

Retrieve the file [`server_deployment.yaml`](../server_deployment.yaml). Apply the server deployment:

```bash
kubectl apply -f server_deployment.yaml
```

This will launch a VolcanoJob resource in your default namespace, in turn launching the four pod types outlined in the Deployment Overview.
As a result, the OpenFAM launcher should eventually print the following logs:

```bash
% kubectl logs openfam-server-launcher-0
[...]
----------------------------
Details of OpenFAM Services:
----------------------------
Service             Id  Host          RPC Port
----------------  ----  ----------  ----------
memory service       0   10.42.X.Y        8793
memory service       1   10.42.X.Y        8793
metadata service     0   10.42.X.Y        8788
CIS                  0   10.42.X.Y        8080
```

### Patch Network Policy

Once the OpenFAM cluster is running, run the following:

```bash
TMPFILE_FAM=$(mktemp)
cat > "$TMPFILE_FAM" <<EOF
spec:
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: openfam
EOF
kubectl patch networkpolicy openfam-server \
    --patch-file "$TMPFILE_FAM"
rm -rf "$TMPFILE_FAM"
unset TMPFILE_FAM
```

Currently, OpenFAM is deployed as a Volcano Job using the Service plugin. By default, Volcano creates a service with an Ingress NetworkPolicy which only allows ingress traffic from pods within that Volcano Job. The patch above modifies the NetworkPolicy to allow traffic from all pods within the used namespace.
Without this patch, a client application will be unable to connect to the OpenFAM cluster.

> You may also remove the entire `namespaceSelector` and patch with an empty `ingress:` label. This will allow anyone on the cluser to connect to your OpenFAM cluster instance! Adding a second `matchLabel` will not work as these labels are ANDed together. Consider using set-based selectors. These have not been tested yet.


### Export OpenFAM Configuration

Clients using `libopenfam` will require the OpenFAM configuration files to use the OpenFAM cluster. Export these as ConfigMaps using the following commands:

```bash
TMPDIR_FAM=$(mktemp -d)
kubectl cp \
    openfam-server-launcher-0:/usr/local/config \
    "$TMPDIR_FAM"
kubectl create configmap openfam-cis   \
  --from-file=$TMPDIR_FAM/fam_client_interface_config.yaml 
kubectl create configmap openfam-meta  \
  --from-file=$TMPDIR_FAM/fam_metadata_config.yaml
kubectl create configmap openfam-mem   \
  --from-file=$TMPDIR_FAM/fam_memoryserver_config.yaml
kubectl create configmap openfam-admin \
  --from-file=$TMPDIR_FAM/openfam_admin_tool.yaml
kubectl create configmap openfam-pe    \
  --from-file=$TMPDIR_FAM/fam_pe_config.yaml
rm -rf "$TMPDIR_FAM"
unset TMPDIR_FAM
```

Verify their existence:

```bash
% kubectl get cm
NAME                 DATA   AGE
openfam-admin        1      16h
openfam-cis          1      16h
openfam-mem          1      16h
openfam-meta         1      16h
openfam-pe           1      16h
```

> Note: If you are re-deploying OpenFAM in the same namespace, then these ConfigMaps might still exist. Delete them first with a `kubectl delete configmap ..`

### Validation

Use the [`client_test_deployment.yaml`](../client_test_deployment.yaml) for quick validation of the cluster:

```bash
kubectl apply -f client_test_deployment.yaml
```

Run a FAM API example:

```bash
% kubectl exec --stdin --tty openfam-client-pod \
  -- /usr/local/bin/openfam-examples/api_fam_add
FAM initialized
fam_add successful!!
FAM finalized
```

## Adjust Cluster resources

### Increasing memory size

The `server_deployment.yaml` file currently uses shared memory at `/dev/shm` as a OpenFAM storage provider and allocates:

- 16GB of shared memory to metadata servers
- 32GB of shared memory to memory servers

Modify the deployment file to increase storage size as needed by adjusting the `sizeLimit` at path `.spec.tasks[].template.spec.volumes`.

For using a different type of memory backend, further modifications are necessary. If you want to use a different type of memory or another memory path, then you will have do adjust the [`setup_openfam.sh`](https://github.com/opencube-horizon/containers/blob/main/openfam/containerfiles/files/setup-openfam.sh) file. See [README.md](https://github.com/opencube-horizon/containers/blob/main/openfam/README.md) for guidance on rebuilding the OpenFAM server image. Alternatively bind-mount an updated `setup_openfam.sh` into your launcher.

### Scaling or Modifying Pod Counts

To change the number of pods or services of the OpeNFAM deployment, modify the `server_deployment.yaml` file by adjusting the number of replicas per Memory (`openfam-mem`) or Metadata (`openfam-meta`) replica type.

## Cluster Restart

Should the memory servers crash or you want to restart the cluster for other reasons, do the following:

```bash
kubectl exec openfam-server-launcher-0 openfam_adm --stop_service --install_path=/usr/local
kubectl exec openfam-server-launcher-0 openfam_adm --start_service --install_path=/usr/local
```

If you need to restart individual pods, then be aware that this will change the IP addresses of these pods. Since OpenFAM does not (yet) support DNS-based configurations, you have to restart the full cluster, regenerate the configuration files by running `kubectl exec openfam-server-launcher-0 sh /usr/local/bin/setup-openfam.sh`, and re-generate the config maps as outlined in [Export OpenFAM Configuration](#export-openfam-configuration).

## Errors

### Launcher errors

If the launcher reports errors, such as a lost connection, check whether all worker pods are launched by running: 

```bash
% kubectl get pods
NAME                               READY STATUS   RESTARTS AGE
openfam-server-launcher-0          1/1   Running  0        101m
openfam-server-openfam-service-0   1/1   Running  0        101m
openfam-server-openfam-service-1   1/1   Running  0        101m
openfam-server-openfam-service-2   1/1   Running  0        101m
```

The `server_deployment.yaml` currently requires all pods to run on different physical nodes via the `topologySpreadConstraints`. If there are too few nodes available for scheduling, then the OpenFAM VolcanoJob cannot start correctly. Should you want to work around this and accept overprovisioning, remove the `topologySpreadConstraints` entry from the deployment file.

### Client errors

If the client cannot connect, for example with the following error message:

```bash
% kubectl exec --stdin --tty openfam-client-pod -- \
  /usr/local/bin/openfam-examples/api_fam_add
FAM Initialization failed: Fam CIS Client: failed to connect to all 
 addresses; last error: UNKNOWN: ipv4:10.42.1.63:8080: Failed to 
 connect to remote host: Connection refused:10.42.1.63:8080
```

then you may have forgotten to patch the Network Policy as outlined above.