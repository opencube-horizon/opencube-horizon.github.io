# RDMA Monitoring via libfabric

RDMA network traffic is designed to minimize crossings between user- and kernel-space in order to maximise performance. Classic network monitoring approaches therefore do not apply, given that they usually operate at the user-kernel boundary. OpenCUBE provides RDMA network monitoring that operates at the level of the network abstraction library, which for OpenCUBE is libfabric.

Applications that use RDMA communication at OpenCUBE eventually utilise libfabric, either directly or indirectly via communication libraries such as MPI or OpenFAM.
Libfabric provides a monitoring plugin that can be loaded by the application, which exposes monitoring information to a sampler. If present, this sampler periodically queries the application for monitoring data via the plugin, and exports it to a database for persistence. Overhead is at around 1% for synthetic, high-network-pressure benchmarks. 

Monitoring data comprises the amount of invocations to libfabric API invocations and the sum of the data volume that has been processed within a sample period.

OpenCUBE provides all necessary components for self-deployment of a monitoring environment.

## Deployment

A full monitoring environment comprises:

1. launching the database
2. enabling the libfabric monitoring plugin
3. launching the sampler

### Database

The sampler currently supports data export via the [Influx Line Protocol](https://docs.influxdata.com/influxdb3/core/reference/line-protocol). Any database that accepts this format is supported. The example deployment outlined here uses an InfluxDB 3.

First, create a token file for secure access. Either create a short-lived pod on the cluster:

```bash
kubectl apply -f - <<EOF
---
apiVersion: v1
kind: Pod
metadata:
  name: influxdb
spec:
  securityContext:
    runAsUser: $(id -u)
    fsGroup: $(id -u)
    runAsNonRoot: true
  containers:
  - image: influxdb:3-core
    name: influxdb
    command: ["influxdb3"]
    args: 
    - create
    - token 
    - --admin 
    - --offline 
    - --output-file
    - /output/token.json
    volumeMounts:
    - name: home
      mountPath: /output
  volumes:
  - name: home
    persistentVolumeClaim:
        claimName: pvc-home-directory
EOF
kubectl delete pod influxdb
```

Or generate a token locally using e.g. Podman:

```bash
podman run --rm -it  influxdb:3-core \
  /bin/bash -c 'influxdb3 create token --admin --offline --output-file /tmp/token.json >/dev/null && cat /tmp/token.json' \
  > token.json
```

Next, create a secret on the OpenCUBE cluster containing that token: 
```bash
kubectl create secret generic influxdb-secret \
  --from-file=token.json \
  --from-literal=token=$(jq -r '.token' token.json)
```
This creates two entries in the `influxdb-secret` entry: a `token.json` file containing a token file compatible with InfluxDB, and a `token` entry that contains the raw token for consumption of down-stream applications.

Next, launch the InfluxDB using the [`influxdb.yaml`](../influxdb.yaml) description:

```bash
kubectl apply -f influxdb.yaml
```

This will create a pod containing the database and a service for connectivity.

Verify that the database is running:

```bash
$ kubectl get pods
NAME       READY   STATUS    RESTARTS  AGE
influxdb   1/1     Running   0         1h

$ curl --header "Authorization: Token $TOKEN" influxdb-service:8181/api/v1/health
OK
```

> Note: Adjust `$TOKEN` to the token value in your `token.json` secret.

### Enable Monitoring Plugin

The libfabric monitoring plugin `ofi_hook_monitor` can be enabled by setting the environment variable `FI_HOOK=monitor`. Further configuration options can be seen on [`man fi_hook`](https://ofiwg.github.io/libfabric/v2.6.0/man/fi_hook.7.html#monitor-hooks).

Enabling this hook will create communication files at the path provided via `FI_OFI_HOOK_MONITOR_BASEPATH`, defaulting to `/dev/shm/ofi`. Point the sampler to this location.

### Launch Sampler

The libfabric monitoring sampler can be launched as a side-car container next to your application. For distributed workloads, such as MPI jobs, the sampler runs once per pod.

Refer to [job-monitoring.yaml](../job-monitoring.yaml) for an example using a Volcano MPI Job.

## Retrieve Data

Data will be stored

```bash
curl -vk \
    --header "Authorization: Bearer $TOKEN" \
    --json '{
      "format": "csv",
      "db":"ofi",
      "q": "select * from ofi_hook_monitor;"
    }' \
    influxdb-service:8181/api/v3/query_sql
```

Example output:

```csv
bucket,child_provider_ids,context_id,context_type,count,function,hostname,job_id,pid,ppid,provider,provider_id,sum,time,uid
0_64,-,0x903d6b0,cq,91121.0,mon_cq_tagged_rx,lm-mpi-job-mpiworker-0,0,55,37,tcp,0x9040380,420116.0,2026-09-06T10:31:25.011534679,1001
0_64,-,0x903d6b0,cq,683.0,mon_cq_tagged_rx,lm-mpi-job-mpiworker-0,0,55,37,tcp,0x9040380,0.0,2026-09-06T10:31:26.011810345,1001
0_64,-,0x903d6b0,cq,682.0,mon_cq_tagged_rx,lm-mpi-job-mpiworker-0,0,55,37,tcp,0x9040380,0.0,2026-09-06T10:31:27.012082331,1001
0_64,-,0x903d6b0,cq,684.0,mon_cq_tagged_rx,lm-mpi-job-mpiworker-0,0,55,37,tcp,0x9040380,0.0,2026-09-06T10:31:28.012182758,1001
0_64,-,0x903d6b0,cq,690.0,mon_cq_tagged_rx,lm-mpi-job-mpiworker-0,0,55,37,tcp,0x9040380,48.0,2026-09-06T10:31:29.012256344,1001
```

Refer to [`fi_mon_sampler(1)`](https://ofiwg.github.io/libfabric/v2.5.1/man/fi_mon_sampler.1.html) for more information on the output data. 

> Note: The libfabric version deployed at OpenCUBE currently contains a custom version of libfabric with an updated and not yet upstreamed version of the libfabric monitoring system. Refer to the output of [`man fi_mon_sampler(1)`](../fi_mon_sampler(1)) inside the `rt-libfabric` container.

## Using the Data

Output data shows the number of libfabric API invocations and the sum of handled data per data bucket. This information can be used to gain insight into when and how much data the application submits to the RDMA network. Note that this does not necessarily coincide with the actual achieved bandwidth: An application can submit hundreds of gigabytes of traffic nearly instantaneously, which is then processed by the NIC at the fastest achievable data rate.

Open MPI 5 and MPICH primarily use the libfabric functions `tsenddata` and `tinjectdata` for transmission, as well as `cq_tagged_rx` for completion queue checking. 

Assuming a decently modern Python3 with Pandas installed, you can gain some insight into the application output as follows:


Fetch `zstandard`-compressed data:
```bash
$ curl --silent --show-error \
  --header "Authorization: Bearer $INFLUX_TOKEN" \
  --json '{"db":"ofi","format":"csv","q": "select * from ofi_hook_monitor;"}' \
  "$DB/api/v3/query_sql" \
  | zstd -o "$OUTFOLDER/influxdb.csv.zstd"
```

Display overview of used functions:
```bash
$ python3 - <<EOF
import pandas as pd
df = pd.read_csv("$OUTFOLDER/influxdb.csv.zstd", compression="zstd")
print(df.function.value_counts())
EOF

function
mon_cq_tagged_rx    20474
mon_trecv           20227
mon_cq_tagged_tx    19028
mon_tsenddata       18877
mon_tinjectdata      1520
Name: count, dtype: int64
```

Display overview of submitted data packets per host and PID for function `tsenddata`:
```bash
$ python3 - <<EOF
import pandas as pd
df = pd.read_csv("$OUTFOLDER/influxdb.csv.zstd", compression="zstd")\
      .set_index(["hostname","pid","provider","time"])
print(df[df.function == "mon_tsenddata"][["sum"]].sum(axis=1).sort_index())
EOF

hostname  pid      provider  time
cn01      3509904  cxi       2026-09-10T13:13:59.663177128       8960.0
                             2026-09-10T13:13:59.673238478    3670016.0
                             2026-09-10T13:13:59.683298667    2621440.0
                             2026-09-10T13:13:59.693357457    3145728.0
                             2026-09-10T13:13:59.703416727    2621440.0
                                                                ...
cn02      3746554  cxi       2026-09-10T13:14:10.073122726    3145728.0
                             2026-09-10T13:14:10.083184527    2621440.0
                             2026-09-10T13:14:10.093246008    3145728.0
                             2026-09-10T13:14:10.103306009    3145728.0
                             2026-09-10T13:14:10.223989305     524288.0
Length: 18877, dtype: float64
```

## Debugging

The application running the monitoring plugin can be debugged by setting the environment variable `FI_LOG_LEVEL=trace`. The monitoring plugin will report which providers are monitored and which files are created. Example output for an application for one libfabric process using TCP:

```
libfabric:1756604:1788450356::tcp:fabric:hook_monitor_fabric():1423<trace> [tcp] Installing monitor hook
libfabric:1756604:1788450356::tcp:fabric:hook_monitor_fabric():1436<trace> [tcp] Installed monitor hook
libfabric:1756604:1788450412::tcp:fabric:monitor_context_close():1346<trace> [tcp] Lingering enabled, will flush file /dev/shm/ofi/12345/hostname/1756604.Ft2lCa but not delete.
libfabric:1756604:1788450412::tcp:fabric:monitor_context_close():1346<trace> [tcp] Lingering enabled, will flush file /dev/shm/ofi/12345/hostname/1756604.dlyrhJ but not delete.
```

The sampler can be debugged by adding the `-v` flag. This will report additional information such as the amount of sample targets per sample and the consumed wallclock and CPU time. Example output:

```
Starting sampler
Starting pusher
Spent 50 ms / 1000 ms (9050 left) for 4 files (1% CPU, 1Hz)
Spent 50 ms / 1000 ms (9050 left) for 4 files (1% CPU, 1Hz)
[...]
Received SIGINT, will stop
```

### Common Issues


##### `Could not stat <path>: No such file or directory`

Make sure to correctly specify the target monitoring path. This should match the path set in `FI_OFI_HOOK_MONITOR_BASEPATH`. In the container case, also make sure that this path is shared between the application and the sampler! Note that you should use a tmpfs-share for the communication files. Use an [`emptyDir medium: memory` volume](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir).

##### `DB returned 401`

Make sure that the database you pass in `-r` can be accessed and that you specify the correct token via `MON_SAMPLER_REMOTE_HEADERS="Authorization: Bearer [...]`. The provided example job script uses the `influxdb-secret` token to automatically inject the correct environment variable.