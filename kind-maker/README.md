# kind-maker

This workspace bootstraps a control-plane-only `kind` cluster with:

- a pinned `kindest/node` image based on Kubernetes `v1.35.0`
- Gateway API CRDs
- `cloud-provider-kind`
- `local-path-provisioner` as the default `StorageClass`
- `cert-manager`
- `metrics-server`

Optional extras:

- an opt-in shared HTTPS cluster `Gateway`
- `kube-prometheus-stack`
- Argo CD
- an opt-in local self-signed `ClusterIssuer`

## Core targets

- `make kind-preflight`
- `make kind-bootstrap-base`
- `make kind-install-gateway-api-gateway`
- `make kind-expose-grafana`
- `make kind-print-grafana-access`
- `make kind-print-grafana-admin-password`
- `make kind-install-observability`
- `make kind-install-argocd`
- `make kind-install-cert-manager-issuer`
- `make kind-bootstrap-full`
- `make kind-delete`
- `make kind-reset`
- `make kind-create-debug`

## Image workflow

The local image workflow uses a host-side OCI registry container, not an in-cluster registry.

- Default registry container name: `kind-registry`
- Default registry endpoint on the host: `localhost:5001`
- Both can be overridden with `REGISTRY_NAME=...` and `REGISTRY_PORT=...`

Typical flow:

```sh
docker build -t localhost:5001/my-app:dev .
docker push localhost:5001/my-app:dev
kubectl apply -f k8s/
```

After cluster creation, the bootstrap writes a `hosts.toml` entry into the kind node so containerd can resolve `localhost:5001/...` to the attached registry container.

## Notes

- `k9s` is treated as a core client dependency alongside `kind`, `kubectl`, `helm`, and `cloud-provider-kind`.
- `cloud-provider-kind` is treated as a shared host service. 
- `make kind-delete` removes the cluster but does not stop the shared controller.
- The effective `kind` config is rendered to `.state/kind-cluster.yaml`.
- The workspace keeps its own kubeconfig at `.state/kubeconfig` instead of relying on your global `kubectl` context.
- Run `eval "$(make kind-shell-env)"` if you want your interactive shell to point `kubectl` and `helm` at the workspace kubeconfig after cluster creation.
- `make kind-install-gateway-api` installs Gateway API CRDs only; use `make kind-install-gateway-api-gateway` to create the shared HTTPS cluster `Gateway` and wildcard `.local` certificate.
- `cert-manager` is installed without a default issuer; use `make kind-install-cert-manager-issuer` if you want an opt-in self-signed `ClusterIssuer` named `local-self-signed`.
- `make kind-expose-grafana` creates an `HTTPRoute` for `https://grafana.local` through the shared Gateway.
- `make kind-print-grafana-access` prints the `/etc/hosts` entry and current practical Grafana URL based on the mapped `kindccm` proxy port.
- `make kind-print-grafana-admin-password` prints the current Grafana admin password from the generated Kubernetes secret.
- If cluster creation fails before bootstrap, use `make kind-create-debug` so the failed node is retained for inspection.
- `make kind-sync-state` clears stale workspace state if a cluster was deleted out of band.

## Troubleshooting

- Run `make kind-preflight` before a fresh bootstrap if you want a quick check of tools, Docker, registry, and local state.
- After `make kind-expose-grafana`, add `127.0.0.1 grafana.local` to `/etc/hosts`, then inspect the mapped gateway proxy port with `docker ps --format 'table {{.Names}}\t{{.Ports}}' --filter name=kindccm-`.
- On Docker Desktop, the shared Gateway is typically reachable through that mapped host port, so the practical URL is usually `https://grafana.local:<mapped-port>`.
- The shared Gateway uses a self-signed wildcard `*.local` certificate by default, so browsers will warn until you trust the issuing CA or switch to a different issuer flow.
- If cluster creation fails early, run `make kind-create-debug` so the failed node is retained for inspection.
- If the tracked cluster was removed outside the Makefile, run `make kind-sync-state`.
- `make kind-status` gives a compact view of node health, storage, Gateway resources, certificates, core add-ons, registry state, and metrics availability.
- If plain `kubectl` is not talking to the cluster, run `export KUBECONFIG=$PWD/.state/kubeconfig` in the repo before using `kubectl` directly.

## Known Limitations On macOS

- The Kubernetes Gateway address is not always directly reachable from the host when using Docker Desktop on macOS.
- `cloud-provider-kind` may expose the shared Gateway through a dynamically mapped host port instead of a stable host port like `443`.
- Recreating the shared Gateway can change the mapped host port, so local browser access is not yet a stable `https://grafana.local` experience.
- A `Programmed=True` Gateway and an attached `HTTPRoute` confirm the in-cluster configuration, but they do not guarantee frictionless host access on macOS.
