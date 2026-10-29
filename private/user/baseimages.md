# OpenCUBE Base Container Images

OpenCUBE provides a set of base container images that can be used to build custom containers. Images are provided at the [OpenCUBE container registry](https://harbor.pt.horizon-opencube.eu). The container files are provided on the [OpenCUBE GitHub](https://github.com/opencube-horizon/containers) project.

Currently, the following core runtime images are provided:

- `rt-baseimage`
	- OpenSUSE Leap 16 based
	- includes kanidm integration (see [Container Image Customization](../containerssh/customization)) 
	- includes `tini` as init process
- `rt-libfabric`
	- based on `rt-baseimage`
	- includes [Slingshot](../slingshot/slingshot) aware libfabric
- `rt-mpi`
	- based on `rt-libfabric`
	- includes libfabric-aware Open MPI 5

Use and extend these images to your liking. Do note, however, that it is recommended to keep the following entrypoint and command:

```docker
ENTRYPOINT ["/usr/local/bin/tini", "-s", "--", "/usr/local/bin/start"]
CMD ["/bin/bash", "-li"]
```

This ensures that the kanidm integration is running. If you do not need this feature in your container, then you can safely overwrite this.

For an example usage of the `rt-mpi` image, refer to [MPI Jobs](../mpi/mpi).

For OpenFAM base images and usage, refer to [OpenFAM](../openfam/usage).