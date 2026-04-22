# k8s-wizard

A self-contained local Kubernetes environment for macOS Apple Silicon. Spins up a kind cluster inside a Vagrant/Parallels VM with Gateway API, TLS, and cert-manager pre-configured.

## What you get

- **Ubuntu 24.04 VM** via Vagrant + Parallels (4 CPUs, 8 GB RAM)
- **Kind cluster** (single control-plane node)
- **Nginx Gateway Fabric** with HTTPS on port 443 (`*.vm.local`)
- **cert-manager** with a self-signed ClusterIssuer
- **Tools in the VM:** Docker, kubectl, helm, kind, k9s
- **Host access** to the cluster via exported kubeconfig and port forwarding

## Prerequisites

- macOS on Apple Silicon
- [Homebrew](https://brew.sh)
- [Vagrant](https://www.vagrantup.com) (`brew install vagrant`)
- [Parallels Desktop](https://www.parallels.com)
- Vagrant Parallels plugin (`vagrant plugin install vagrant-parallels`)

Run `make preflight` to verify your setup.

## Getting started

```bash
# Install client tools on the host
make install-client-tools

# Create the VM and provision the cluster
make vm-up

# Export the kubeconfig
make get-kubeconfig
export KUBECONFIG=$(pwd)/kubeconfig.yaml

# Verify
kubectl get nodes
```

## Managing the VM

```bash
make vm-halt       # shut down
make vm-suspend    # suspend to disk
make vm-up         # resume
make vm-provision  # re-run provisioning
make vm-destroy    # destroy everything
```

## Deploying an app

Create an HTTPRoute in any namespace to expose a service through the Gateway:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app
spec:
  parentRefs:
    - name: default-gateway
      namespace: nginx-gateway
  hostnames:
    - "my-app.vm.local"
  rules:
    - backendRefs:
        - name: my-app-svc
          port: 80
```

Then add `127.0.0.1 my-app.vm.local` to `/etc/hosts` and visit `https://my-app.vm.local`.

## Project structure

```
.
├── Makefile                              # Host-side targets
├── Vagrantfile                           # VM definition and provisioning
└── manifests/
    ├── kind-cluster.yaml                 # Kind cluster config
    ├── selfsigned-clusterissuer.yaml     # cert-manager ClusterIssuer
    └── default-gateway.yaml              # Gateway with wildcard TLS
```

Run `make help` for all available targets.
