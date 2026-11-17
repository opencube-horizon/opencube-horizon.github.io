# Scheduling Workflows

The main workflow interface of OpenCUBE is Kubernetes, which is deployed using Talos. OpenCUBE supports all standard Kubernetes workflow types in addition to some special features such as MPI support (cf. [MPI](../../mpi/mpi)). 

## Workflow Restrictions

There are some basic restrictions that are applied to all workflows. 

### User Namespaces

OpenCUBE enforces the use of [User Namespaces](https://kubernetes.io/docs/tasks/configure-pod-container/user-namespaces) for all Kubernetes pods. This requires the setting `spec.hostUsers: false`. Workflows that do not set this setting will be rejected with the following message:

```
The pods "test" is invalid: : ValidatingAdmissionPolicy 'require-user-namespaces' with binding 'require-user-namespaces' denied request: Pods must set spec.hostUsers to false.
```

The following pod description is an example on how to set this setting:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test
spec:
  hostUsers: false
  containers:
  - name: demo
    image: alpine:latest
```

### Pod Security Standard

Talos enforces the "baseline" [Pod Security Standard](https://kubernetes.io/docs/concepts/security/pod-security-standards/) profile and will warn about settings that do not match the "restricted" profile. Refer to the K8s documentation about which settings are rejected by the "baseline" profile. The majority of settings revolve around preventing sharing of host namespaces. Most existing workflows will not need to be touched.

A common warning that appears when violating the "restricted" profile is given below. This warning should be addressed, but the workloads are still scheduled.

```plaintext
Warning: would violate PodSecurity "restricted:latest":
 allowPrivilegeEscalation != false
  (container "test" must set securityContext.allowPrivilegeEscalation=false), 
 unrestricted capabilities
  (container "test" must set securityContext.capabilities.drop=["ALL"]), 
 runAsNonRoot != true
  (pod or container "test" must set securityContext.runAsNonRoot=true), 
 seccompProfile
  (pod or container "test" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
```

A full example of a pod that will not issue warnings is given below.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test
spec:
  hostUsers: false
  containers:
  - name: test
    image: alpine:latest
    command: '/bin/ash'
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
      runAsUser: 1001
      runAsGroup: 1001
      runAsNonRoot: true
      seccompProfile:
        type: Localhost
        localhostProfile: profiles/opencube.json
```

Note that the `runAs{User,Group}` settings can be set to arbitrary, non-zero IDs. Refer to [Storage](../../storage/storage) for a discussion on ID impact. Given that this pod does not run as root, you cannot perform several operations that may be usualy expected within pods, such as manually starting a SSHD or installing packages.