# System Access

User management at OpenCUBE is managed via the identity management tool kanidm. During onboarding, you will receive an account, for which you need to set a password, 2FA, and an SSH publickey.

The OpenCUBE system can be accessed in two ways:

1. via SSH
2. via kubectl

## SSH

SSH access uses Login Containers. SSH into the OpenCUBE system with the SSH key you provided during onboarding:

```bash
ssh -p $PORT -i /path/to/key $NODE
```

In the current configuration (2026-09-04), access is possible from the OpenCUBE login. Example:

```bash
$ ssh -p 32223 -i ~/.ssh/id_ed25519 cn04
user@containerssh-user-abcde:~>
```

Refer to the [Login Containers](../../containerssh/usage) page for more information.

## kubectl

Access to Kubernetes resources is established via the `kubectl` utility. 

### Namespaces

Every user has access to two Kubernetes namespaces by default: 

1. `ns-$USERNAME`
2. `ns-group-$GROUPNAME`

The first namespace is meant as a "private" namespace for individual users. The second namespace can be used if kubernetes resources need to be shared within a group.

### Access from Login Container

After logging into a login container, the pre-installed `kubectl` is already fully configured to provide access to the `ns-$USERNAME` namespace.
Access to group namespaces requires a full login. Refer to the [Local Usage](#local-usage) Guide on how to set up a OIDC-login.

### Access from local machine

Access from a local machine uses Open-ID Connect (OIDC) for authentication. `kubectl` supports the OAuth2-Flow, but requires an external plugin.

> Note: OIDC-based access requires either a valid token or a password+TOTP, which you receive during onboarding.

Install [`krew`](https://krew.sigs.k8s.io/) by following the official guide, which manages `kubectl` plugins.
Next, install `oidc-login` using krew:
```bash
kubectl krew install oidc-login
```

Create a user `oidc` (adapt the name as you see fit) and point it to the OpenCUBE IDM:
```bash
kubectl config set-credentials oidc \
	--exec-api-version=client.authentication.k8s.io/v1 \
	--exec-interactive-mode=Never \
	--exec-command=kubectl \
	--exec-arg=oidc-login \
	--exec-arg=get-token \
	--exec-arg="--oidc-issuer-url=https://idm.horizon-opencube.eu:8443/oauth2/openid/kubernetes" \
	--exec-arg="--oidc-client-id=kubernetes" \
	--exec-arg="--skip-open-browser"
```

Receive the cluster information from TODO.

Create a context for the OpenCUBE cluster:

```bash
kubectl config set-context oidc@virtual-talos \
	--user oidc \
	--cluster virtual-talos \
	--namespace ns-<your-username>
```

Lastly, set it as the current context:

```bash
kubectl config use-context oidc@virtual-talos
```

You should now be able to log in:

```bash
% kubectl --user oidc --cluster virtual-talos auth whoami
Please visit the following URL in your browser: http://localhost:8000/
[.. follow the steps outlined in the browser ..]
ATTRIBUTE   VALUE
Username    oidc:12346789-1234-1234-1234-1234567890AB
Groups      [oidc:group1 oidc:group2 system:authenticated]
```

When using a login container (or the OpenCUBE login node), use SSH port tunneling to tunnel the `localhost:8000` server to your machine via `ssh -L 8000:localhost:8000 [...]`.

In the current setup (2026-09-04), the `kanidm` backend is not accessible from outside the cluster. Use `sshuttle` to make it accessible: `sshuttle -vNHr login.pt.horizon-opencube.eu`

> Note: Your Username as reported by `kubectl` will _not_ be your human-readable username, but instead a UUID.

