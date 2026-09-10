# Slingshot

> Note: Slingshot has not yet been enabled on the Talos-deployment. (2026-09-10)

OpenCUBE provides access to the high-speed RDMA network Slingshot, which offers 200Gbit/s line speed. In order to use Slingshot, several components are involved:

1. Slingshot network device (`cxi`)
2. Slingshot userspace tooling (`libcxi`)
3. libfabric (standard interface to `libcxi`)
4. Slingshot VNIs

## Slingshot Network Device

The Slingshot network device `cxi` (short for Cassini, the name of the NIC) is reported in Kubernetes as a resource via the `smarter-devices` system. It will appear as a character device in `/sys/dev`. 

Request a CXI device using Kubernetes [resource requests](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/):

```yaml
resources:
  requests:
    smarter-devices/cxi0: "1"
  limits:
    smarter-devices/cxi0: "1"
```

## Slingshot Userspace Tooling

The Slingshot NIC Cassini can be accessed via `libcxi`. The OpenCUBE base images include a compatible version of this by default.

## libfabric

Libfabric is the core interface to use Slingshot from applications. Applications that target Slingshot, such as MPI or OpenFAM applications, must eventually target libfabric! The OpenCUBE-provided MPI images include a compatible libfabric + MPI stack.

## Slingshot VNIs

Slingshot uses the concept of Virtual Networks and their associated Virtual Network IDs (VNIs). Similar to classical VLANs, a VNI identifies a segregated, virtual network domain that is used to securely communicate between applications and nodes. 

By default, applications using Slingshot will use the global VNI, which provides no isolation. If isolation is desired, then you can set the following resource annotation:

```yaml
metadata:
  annotations:
    vni: <"true"|"false"|$CLAIM_NAME>
```

### Automatic VNIs

When set to true, then a new unique VNI is generated for that workload. In the case of a distributed workload, such as a [Volcano MPI Job](../mpi/mpi) or a ReplicaSet, all pods within that resource will be granted access to that VNI. Usage is completely transparent, both in terms of configuration and performance.
You can track VNI usage by enabling [libfabric logging](https://ofiwg.github.io/libfabric/main/man/fabric.7.html) via the environment variable `FI_LOG_LEVEL=debug`. The `cxi` provider will provide information on which VNI is being used.

### VNI Claims

When set to a string that is neither true nor false, then that string needs to refer to the name of a VNI Claim. VNI Claims provide a way to share a VNI across multiple resources, such as multiple MPI Jobs. VNI Claims can be created using the following resource description:

```yaml
apiVersion: horizon-opencube.eu/v1
kind: VniClaim
metadata:
  name: vni-claim-$NAME
spec:
  name: $NAME
```

After creation, you can check the created VNI Claim with:

```yaml
$ kubectl get vniclaims
NAME             AGE
vni-claim-test   1h
```

Each VNI Claim is automatically associated with a corresponding VNI. VNI Claims can be redeemd within resources by setting the `vni: $NAME` annotation: Every application redeeming that claim will get access to that same VNI and can therefore communicate via Slingshot.

> Note: The `metadata.name` refers to the name of the Kubernetes-facing VniClaim resource. The `spec.name` identifies the actual VNI! Always annotate resources with the `spec.name` entry.

## Examples

TBD