# Using OpenFAM

[OpenFAM](https://openfam.github.io/) is an API providing fabric-attached memory (FAM). 


OpenCUBE provides a base container image which ships with a pre-installed OpenFAM installation. It is currently located at [harbor.pt.horizon-opencube.eu/openfam-public/openfam-client:latest](https://harbor.pt.horizon-opencube.eu/openfam-public/openfam-client). This document describes the usage of this base container image with K8s deployment of OpenFAM at OpenCUBE. For information of the OpenFAM cluster, refer to the [Deployment Guide](../deployment.md).

## Terminology

- "OpenFAM cluster" refers to a running OpenFAM stack as deployed by the Deployment Guide.
- "OpenFAM installation" refers to the OpenFAM installation, including `libopenfam` and the OpenFAM headers.

## Description

The base image is built ontop of OpenSUSE Leap 16. It includes:

- OpenFAM 3.1 at `/usr/local`
- Python 3.13
- OpenSSH
- OpenSSL

## Building custom Apps

You can use the base image as you would use any other container image. The base image notably does not contain any compiler. If you want to build applications, consider using a multi-stage container:

```Dockerfile
FROM harbor.pt.horizon-opencube.eu/baseimages/openfam-client:latest as builder
RUN zypper --non-interactive refresh \
 && zypper --non-interactive install \
    <dependencies>
RUN <build-your-app>

FROM harbor.pt.horizon-opencube.eu/baseimages/openfam-client:latest 
COPY --from builder \
  /path/to/files-in-builder /path/to/files-in-final-image

RUN zypper --non-interactive refresh \
 && zypper --non-interactive install \
    <dependency-libs>

ENV OPENFAM_INSTALL_DIR=/usr/local/
ENV OPENFAM_ROOT=/usr/local/
```

Given that the base image `openfam-client` has OpenFAM installed at `/usr/local`, make sure to instruct your applications depending on OpenFAM to search at this location, for example via a `./configure --with-openfam=/usr/local`. 

Please make sure that you export the `OPENFAM_INSTALL_DIR` and `OPENFAM_ROOT` environment variables, as these are required by `libopenfam` to find for example the configuration files.

Using multi-stage containers can help reducing the size of your final container image by not including large programs like gcc or large source trees.

## Running Image

The following steps assume a running OpenFAM cluster deployment in your default namespace as deployed in the Deployment Guide. Adapt as necessary. 


### libopenfam
If your application is built on top of the OpenFAM base image and uses `libopenfam`, it will require the OpenFAM configuration files to be located at `/usr/local/config`. Within the used namespace of the OpenFAM cluster, there should be ConfigMap resources that contain these configurations. 
If not, then please consult the Deployment Guide linked above.

Mount the ConfigMap entries into your workload. An example using a pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: myimage
    volumeMounts:
    - name: cis
      mountPath: "/usr/local/config/fam_client_interface_config.yaml"
      subPath: "fam_client_interface_config.yaml"
    - name: meta
      mountPath: "/usr/local/config/fam_metadata_config.yaml"
      subPath: "fam_metadata_config.yaml"
    - name: mem
      mountPath: "/usr/local/config/fam_memoryserver_config.yaml"
      subPath: "fam_memoryserver_config.yaml"
    - name: pe
      mountPath: "/usr/local/config/fam_pe_config.yaml"
      subPath: "fam_pe_config.yaml"
  volumes:
  - name: cis
    configMap:
      name: openfam-cis
  - name: meta
    configMap:
      name: openfam-meta
  - name: mem
    configMap:
      name: openfam-mem
  - name: pe
    configMap:
      name: openfam-pe
```

If you want to run your application in a different namespace than the OpenFAM cluster, then please refer to the deployment guide on how to adjust the OpenFAM cluster deployment to accept traffic from non-local namespaces.

Should you want to deploy your own OpenFAM cluster, the please also refer to the Deployment Guide linked above.

### Custom FAM implementation

If you use a custom FAM implementation that is not built upon `libopenfam`, then you can access the cluster via the Client Interface Server (CIS). You can extract the IP and port from the logs of the OpenFAM Deployment launcher:

```bash
% kubectl logs openfam-server-launcher-0
[...]
Service             Id  Host           RPC Port
----------------  ----  -----------  ----------
[...]
CIS                  0  10.42.X.Y          8080
```

Alternatively, you can extract the CIS IP via Kubernetes DNS. Assuming you have `bind-utils` installed run the following command from within a pod running in the same namespace the OpenFAM cluster is deployed:

```bash
% dig +short +search openfam-server-openfam-cis-0.openfam-server
10.42.X.Y
```

You can also use this FQDN within applications.

> The format of the DNS search string is: `<pod-name>.<vc-jobname>`. Search for the CIS pod name via `kubectl get pods`. Search for the Volcano (VC) Jobname via `kubectl get vcjob`. If the Deployment Guide has been used, the pod name and Volcano Job Name will be as stated above.

The default CIS port, if the Deployment Guide has been used to set up the cluster, will be 8080.