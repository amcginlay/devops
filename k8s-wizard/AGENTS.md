# k8s-wizard

Local Kubernetes development environment running a kind cluster inside a Vagrant VM on macOS Apple Silicon.

## Architecture

- **Host (macOS):** Runs Vagrant and Parallels. Interacts with the cluster via an exported kubeconfig.
- **VM (Ubuntu 24.04 on Parallels):** Runs Docker, kind, and all cluster workloads.
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
make install-client-tools   # install kubectl, helm, k9s on the host
make vm-up                  # create and provision the VM + cluster
make get-kubeconfig         # export kubeconfig for host access
export KUBECONFIG=$(pwd)/kubeconfig.yaml
kubectl get nodes
```

## Make targets

Run `make help` for the full list.

| Target | Description |
|---|---|
| `preflight` | Verify host meets requirements |
| `install-client-tools` | Install kubectl, helm, k9s via Homebrew |
| `vm-up` | Create and provision the VM |
| `vm-provision` | Re-run VM provisioning |
| `get-kubeconfig` | Export kubeconfig for host access |
| `vm-halt` | Gracefully shut down the VM |
| `vm-suspend` | Suspend the VM to disk |
| `vm-destroy` | Destroy the VM and all resources |

## Key files

- `Vagrantfile` — VM definition and provisioning (Docker, kubectl, helm, kind, k9s, cert-manager, Nginx Gateway Fabric, Gateway)
- `Makefile` — Host-side targets for managing the VM and cluster
- `manifests/kind-cluster.yaml` — Kind cluster config (API server binding, port mappings, cert SANs)
- `manifests/selfsigned-clusterissuer.yaml` — cert-manager self-signed ClusterIssuer
- `manifests/default-gateway.yaml` — Gateway resource with wildcard TLS for `*.vm.local`

## Conventions

- All VM provisioning logic lives in the Vagrantfile shell provisioner, not the Makefile.
- Kubernetes manifests go in `manifests/`.
- The Makefile is for host-side operations only. Targets that need to run inside the VM use `vagrant ssh -c` or `vagrant provision`.
- Provisioning should be idempotent — safe to re-run with `make vm-provision`.
- Helm installs use `helm upgrade --install` for idempotency.
