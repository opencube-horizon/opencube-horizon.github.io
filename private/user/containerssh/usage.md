# Login Containers via ContainerSSH

Access to the OpenCUBE platform is provided via login containers, provided via ContainerSSH, as opposed to a classical set of login nodes. When logging in via SSH, you access a private login container instance, which you can freely configure to your needs.

## Login

Simply SSH into the system as usual:
```bash
ssh -p $PORT -i /path/to/key $NODE
```

where:

- `$PORT` refers to the port specified in `nodePort` at `containerssh/configs/service.yaml` (default: 32223), and
- `$NODE` refers to one node of the kubernetes cluster where the ContainerSSH Kubernetes Service is running.

Example:

```bash
$ ssh -p 32223 -i ~/.ssh/id_ed25519 cn04
user@containerssh-user-abcde:~>
```

> Note: The first login may take some seconds. This is due to ContainerSSH first fetching your container from the registry, creating a new container, and finally forwarding your SSH session to the newly created pod. See [Multi-Session](#multi-session) for re-using existing login container instances.

Within your login container, you will be given a unique UID/GID and placed in your home directory.

## Persistence

Login containers are ephemeral. Any state is only preserved per SSH connection. You may modify your container as you wish, but note that only your home directory at `/home/$username` persists across SSH connections. Any other modifications will not persist. Refer to [Customization](../customization) if you want to modify your container image.

Your home directory is mounted as a Persistent Volume. You can re-use that volume by utilising the Persistent Volume Claim `pvc-home-directory` associated with it and provided within your default namespace.

You can check it via:

```bash
$ kubectl get pvc
NAME                STATUS   VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS    VOLUMEATTRIBUTESCLASS   AGE
pvc-home-directory  Bound    pvc-$VOLUME   64Gi       RWX            csi-cephfs-sc   <unset>                 
```

You can use it within your pods as follows:

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: pod
    image: image
    volumeMounts:
    - name: home
      mountPath: /home/$USERNAME
  volumes:
  - name: home
    persistentVolumeClaim: 
      claimName: pvc-home-directory
```

## Multi-Session

To gain multiple shells into one session, use SSH Connection Multiplexing via [SSH Control Masters](https://manpages.ubuntu.com/manpages/noble/man5/ssh_config.5.html#ControlMaster). The easiest way is to use an appropriate SSH config entry:

```
Host opencube
  HostName $NODE
  Port $PORT
  User $USER
  ControlPath ~/.ssh/controlmasters/%r@%h:%p
  ControlMaster auto
  ControlPersist yes
```

> Note: Directory `~/.ssh/controlmasters` needs to exist.

The first SSH command will request a login container. Subsequent commands will re-use the connection and land in the same container.
You can check the SSH sessions by listing entries in the `ControlPath`:

```bash
% ls -lAh ~/.ssh/controlmasters
srw------- 2 user user 0 May 18 10:50 user@cn04:3222
```

## Security

Within your login container, you cannot escalate your privileges and cannot run as root. Please refer to [Customization](../customization) for information on modifying your image during image build-time, which includes full freedom on modifying components.

## Debugging

In some cases, a login container may get stuck or refuse to start. You can check the status of your login containers using `kubectl`:

```bash
$ kubectl get pods
NAME                         READY   STATUS      RESTARTS      AGE
containerssh-user-wcfm2   1/1     Running     0             2m4s
```

Refer to [Access](../../system-access/overview.md) for information on `kubectl` usage.