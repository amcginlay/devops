# macos-k8s-wizard

Local Kubernetes development environment running a kind cluster inside a Vagrant VM on macOS Apple Silicon.

## Architecture

- **Host (macOS):** Runs Vagrant, Parallels, and all cluster interaction (kubectl, helm, k9s) via an exported kubeconfig.
- **VM (Ubuntu 24.04 on Parallels):** Runs Docker and kind only. The cluster lives here but is managed from the host.
- **Kind cluster (`k8s-wizard`):** Single control-plane node. Gateway API with Nginx Gateway Fabric, cert-manager with a self-signed ClusterIssuer.

Traffic from the host reaches the cluster via Vagrant port forwarding:
- Port 6443: Kubernetes API server
- Port 443 → NodePort 30443: Nginx Gateway Fabric (HTTPS, TLS termination for `*.vm.local`)

## Prerequisites

- macOS on Apple Silicon (arm64)
- Homebrew
- Vagrant
- Parallels Desktop

Run `make preflight` to verify.

## Quick start

```
make cluster-up    # create VM, kind cluster, and install platform components
kubectl get nodes  # KUBECONFIG is set automatically by the Makefile
```

## Make targets

Run `make help` for the full list.

| Target | Description |
|---|---|
| `preflight` | Verify host meets requirements |
| `install-client-tools` | Install kubectl, helm, k9s via Homebrew |
| `cluster-up` | Create VM, kind cluster, and install platform |
| `cluster-down` | Destroy the VM and all resources |
| `get-kubeconfig` | Download kubeconfig for host access |
| `install-platform` | Install cert-manager, Gateway API, Nginx Gateway Fabric |

## Key files

- `Makefile` — Host-side targets for managing the VM, cluster, and platform
- `Vagrantfile` — VM definition and provisioning (Docker and kind only)
- `manifests/kind-cluster.yaml` — Kind cluster config (API server binding, port mappings, cert SANs)
- `manifests/selfsigned-clusterissuer.yaml` — cert-manager self-signed ClusterIssuer
- `manifests/default-gateway.yaml` — Gateway resource with wildcard TLS for `*.vm.local`

## Conventions

- The Vagrantfile provisions only Docker and kind. Everything else is installed from the host via `install-platform`.
- Kubernetes manifests go in `manifests/`.
- The Makefile sets `KUBECONFIG` automatically — no need to export it manually.
- `cluster-up` chains `install-platform`, which chains `install-client-tools` and `get-kubeconfig`.
- Helm installs use `helm upgrade --install` for idempotency.
