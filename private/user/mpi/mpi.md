# MPI Jobs

OpenCUBE uses Containers as its software packaging system and Kubernetes as its orchestrator. MPI jobs expect to be run in a multi-node cluster either using a resource manager (flux, Slurm et al.) or directly using an MPI launcher. Within Kubernetes, the user is responsible for providing a "cluster" and launching the desired MPI application manually.

In order to keep this process as simple as possible, two approaches can be provided:

1. Volcano
2. MPI-Operator

## Volcano

[Volcano](https://volcano.sh/) describes itself as a "cloud native batch scheduling system" and has support for launching MPI applications. 

Conceptually, Volcano provides a means to automatically:

1. launch a set of identical worker pods
2. create an overlay network to provide basic connectivity
3. create Kubernetes network services to enable DNS-based discovery
4. create an MPI hostfile

What is left to the user:

1. provide a means of launching an MPI app (such as an SSH daemon)
2. lauch the MPI app
3. data management

Refer to [job.yaml](../job.yaml) for an example Volcano MPI Job.

### MPI Base Image

OpenCUBE provides an MPI base image with all relevant components at `harbor.pt.horizon-opencube.eu/baseimages/rt-mpi:latest` ([Containerfile](https://github.com/opencube-horizon/containers/tree/main/baseimages/rt-mpi/)).

This image contains:

- Open MPI v5
- libfabric
	- HPE Slingshot-aware
- SSH server and client tools
- `sshuser` user
	- UID/GID: 1001
	- can launch rootless sshd on port 2022

To launch MPI applications, use the following container description:

```yaml
containers:
- image: harbor.pt.horizon-opencube.eu/baseimages/rt-mpi:latest
  securityContext:
    runAsUser: 1001
    runAsGroup: 1001
    args:
    - /usr/bin/bash
    - -c
    - |
     /usr/sbin/sshd -f /opt/ssh/sshd_config \
     && sleep 1 \
     && mpirun \
      --prtemca plm_ssh_args "-p 2022" \
      <params> \
      <application>
```

This will: 

- launch `sshd` on port 2022
- instruct `mpirun` to use SSH at port 2022

### Volcano Job

A Volcano Job mimics a regular Kubernetes [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/). It executes a set of tasks, each consisting of a set of replicas of one Pod. Using Volcano for MPI involves two tasks: (1) the MPI launcher, and (2) the MPI workers. 

Both the launcher and the workers need to locally start the SSH daemon. The launcher additionally invokes `mpirun`. The `mpi` plugin of Volcano automatically creates an MPI hostfile at `/etc/volcano/$TASKNAME.host`, which you can pass to MPI for launch. 

#### Scheduling 
The number of MPI worker replicas determines the number of "nodes" the MPI job is running on. Please note that by default, pods are scheduled onto any free node matching the resource requirements. The Kubernetes scheduling system processes scheduling based on which [resources](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#resource-units-in-kubernetes) are requested. In specific, you can request/limit:

- CPU core time
- memory

Example, requesting a minimum of 64MiB of storage and 0.25 CPU time, with a promised maximum of 128MiB of memory and 0.5 CPU time:
```yaml
resources:
  requests:
    memory: "64Mi"
    cpu: "250m"
  limits:
    memory: "128Mi"
    cpu: "500m"
```

Setting resource requests guarantees that the resources are available upon scheduling.
CPU time limits are enforced via cgroups as a hard ceiling. Memory limits are enforced using the Linux out-of-memory subsystem as a soft ceiling; if the provided limits are exceeded, the OoM-killer _may_ kill the pod, but might allow it to continue. 

In addition, you can set scheduling constraints such as [topology spread constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/). 
In order to spread your pods across all possible nodes, you can for example limit scheduling of only a single pod per node with the following constraint:

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule
    matchLabelKeys:
      - pod-template-hash
    labelSelector:
      matchLabels:
        app: mpiworker
```
These constraints forces at most one `mpiworker` pod per node as identified by the `kubernetes.io/hostname` label. 

Refer to [job.yaml](../job.yaml) for an example Volcano Job description.

#### MPI Launcher Tuning

MPI launchers such as the Open MPI-provided `mpirun` contain many options for tuning how applications are launched onto the available hosts/nodes. The example application listed in [job.yaml](../job.yaml) uses the launch option `--map-by ppr:1:node`, which launches one MPI process on each available node. This is useful for MPI+OpenMP-style workloads.  Workloads using raw MPI may want to omit this option to spawn as many MPI processes as each node has processors.
Refer to the Open MPI [`mpirun` documentation](https://docs.open-mpi.org/en/v5.0.x/man-openmpi/man1/mpirun.1.html) for more information.

### Storage

By default, no persistent storage is attached to Pods in general and Volcano Jobs in specific. For durable data export, you can either: 

1. copy data off your pods interactively via `kubectl cp`, or
2. mount a persistent volume.

You can mount your home directory mounted into your login container using the Persistent Volume Claim associated with it. Refer to [Login Containers/Usage](../../containerssh/usage#persistence) for more information. 

Refer to [job-storage.yaml](../job-storage.yaml) for an example Volcano Job description template that mounts a PVC. This file requires modification prior to launch! The relevant lines for storage are:

```yaml
spec:
  volumes:
    - name: home
      persistentVolumeClaim:
        claimName: pvc-home-directory
  securityContext:
    fsGroup: 1001
  containers:
    - image: harbor.pt.horizon-opencube.eu/baseimages/rt-mpi:latest
      name: mpilauncher
      args:
        - /usr/bin/bash
        - -c
        - |
         /usr/sbin/sshd -f /opt/ssh/sshd_config \
         && TMPDIR=$(mktemp -d) \
         && mpirun --prtemca plm_ssh_args "-p 2022" \
              --mca pml cm --mca mtl ofi --mca mtl_ofi_provider_include tcp \
              --map-by 'ppr:1:node' \
              --hostfile /etc/volcano/mpiworker.host \
              /opt/osu/mpi/collective/osu_alltoall -m :512 -i 100 \
              >"$TMPDIR/job.out" 2>"$TMPDIR/job.err" \
         && cp --recursive "$TMPDIR" "/home/mounted/job-$(date +%y-%m-%dT%H%M)"
      securityContext:
        runAsUser: 1001
        runAsGroup: 1001
      volumeMounts:
        - name: home
          mountPath: /home/<user-name>
```

Several changes are included in the `mpilauncher`:
 
1. The security context *of the pod* sets the fsGroup UID to the ID of the `sshuser`, 1001.
2. After MPI application termination, output files are copied to the mounted home-directory at `volumeMounts.mountPath`, in this case `/home/mounted`. Due to the `fsGroup` entry, the mounted volume is writable by the `fsGroup`, which has been set to coincide with the `runAsGroup` entry, allowing access from that pod.
3. Your login container sets the entry `fsGroup: <your-id>`, so that you always have access from your login container.

In your login container, you should now see the following output:

```bash
user@containerssh-user-12345:~> ls -lAhd job-*
drwxrws--- 2 buildah user 2 Sep 10 12:13 job-26-09-10T1213
user@containerssh-user-12345:~> ls -lAh job-26-09-10T1213
total 512
-rw-rw-r-- 1 buildah user   0 Sep 10 12:13 job.err
-rw-rw-r-- 1 buildah user 405 Sep 10 12:13 job.out
```
> Note: If your login container has already been running when submitting the Volcano Job, then you may observe that, in your login container, all files in the home-directory are now owned by group `1001`/`buildah`. This is due to the fact that the mechanism implementing `fsGroup` performs a `chown` on the volume prior to pod launch to the ID provided by `fsGroup`. The Volcano Job pod runs with `fsGroup: 1001`, which means a `chown` gets executed there. Since the volume is shared, these changes will be visible in your login container. To re-gain access, re-run the login container, triggering a new `chown` to your UID.

## MPI-Operator

TBD
