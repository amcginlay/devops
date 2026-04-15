# kind-maker

This workspace bootstraps a control-plane-only `kind` cluster with:

- a pinned `kindest/node` image based on Kubernetes `v1.35.0`
- Gateway API CRDs
- `cloud-provider-kind`
- `local-path-provisioner` as the default `StorageClass`
- `cert-manager`
- `metrics-server`

Optional extras:

- an opt-in shared cluster `Gateway`
- `kube-prometheus-stack`
- Argo CD
- an opt-in local self-signed `ClusterIssuer`

## Core targets

- `make kind-preflight`
- `make kind-bootstrap-base`
- `make kind-install-gateway-api-gateway`
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
- `make kind-install-gateway-api` installs Gateway API CRDs only; use `make kind-install-gateway-api-gateway` to create the shared cluster `Gateway` resource.
- `cert-manager` is installed without a default issuer; use `make kind-install-cert-manager-issuer` if you want an opt-in self-signed `ClusterIssuer` named `local-self-signed`.
- If cluster creation fails before bootstrap, use `make kind-create-debug` so the failed node is retained for inspection.
- `make kind-sync-state` clears stale workspace state if a cluster was deleted out of band.

## Troubleshooting

- Run `make kind-preflight` before a fresh bootstrap if you want a quick check of tools, Docker, registry, and local state.
- If cluster creation fails early, run `make kind-create-debug` so the failed node is retained for inspection.
- If the tracked cluster was removed outside the Makefile, run `make kind-sync-state`.
- `make kind-status` gives a compact view of node health, storage, core add-ons, registry state, and metrics availability.
- If plain `kubectl` is not talking to the cluster, run `export KUBECONFIG=$PWD/.state/kubeconfig` in the repo before using `kubectl` directly.
