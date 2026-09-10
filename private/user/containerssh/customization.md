# Container Image Customization

Login container images can be created and modified during container creation. Some components are expected to be present for normal operations, which are documented below. 

## Base Image

For convenience, an OpenCUBE base image has been created, which contains all necessary components to fully function on OpenCUBE. 
The source is available at: [https://github.com/opencube-horizon/login-containers](https://github.com/opencube-horizon/login-containers).
The image is available at: `harbor.pt.horizon-opencube.eu/containerssh/containerssh-baseimage:latest`
The base image currently uses OpenSUSE Leap 16.0 as its base image.

## Image Modification

You can use the OpenCUBE base image as a starting point and modify it to your needs.

Example:
```docker
FROM harbor.pt.horizon-opencube.eu/containerssh/containerssh-baseimage:latest

RUN ... 
```

After building the image, push it to the container image repository at harbor at repository `containerssh`. The tag must follow the format `containerssh-image-$USERNAME`, where `$USERNAME` is your username.

Using buildah as an example:
```bash
buildah build \
	--tag "containerssh-demo:latest"

buildah image push \
    localhost/containerssh-demo:latest \
    harbor.pt.horizon-opencube.eu/containerssh/containerssh-image-user:latest
```

> Note: The images uploaded to the `harbor.pt.horizon-opencube.eu/containerssh` are currently accessible by all users! OpenCUBE intends to provide private, per-user repositories soon.

### Expected Components

The base image `containerssh-baseimage` contains the following relevant components:

- `kanidm-unixd-client`: manages LDAP and NSS integration
- `tini`: init system with proper zombie reaping
- `containerssh-agent`: Agent for managing ContainerSSH session handover
- `buildah`: Container build system with support for build-in-container

Please do not remove these components from your customized image!

Refer to the [base image documentation](https://github.com/opencube-horizon/containers/tree/main/containerssh/baseimage/) for more information on the individual components. 

## Image Building

The OpenCUBE system runs on an aarch64-architecture. The container images therefore must be built for that platform. Should you not have access to an aarch64-system, you can also use your login container to build containers. In order to do so, however, you need to switch to the `buildah` user via `su`, which is integrated into the baseimage. The default password for that user is `buildah`, password-less `su` has been enabled.

Example flow for building an image and pusing it to the harbor registry:
```shell
% ssh <login-container>
user@containerssh-user-x8dxf:~> su buildah
buildah@containerssh-user-x8dxf:/home/user> buildah build -f path/to/Containerfile -t <image:tag>
buildah@containerssh-user-x8dxf:/home/user> buildah login harbor.pt.horizon-opencube.eu
Username: [...]
Password: [...]
buildah@containerssh-user-x8dxf:/home/user> buildah push localhost/<image:tag> harbor.pt.horizon-opencube.eu/<project>/<image:tag>
buildah@containerssh-user-x8dxf:/home/user> exit
user@containerssh-user-x8dxf:~>
```

> Note: All buildah-related files are ephemeral. Either push your image to a remote registry or export it before closing the login container!