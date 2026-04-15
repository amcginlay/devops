# CLAUDE.md

## Purpose

This repository bootstraps a control-plane-only `kind` cluster for local platform work.

The base platform includes:

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

## Working rules

- Favor hyphens over underscores when naming new targets, helpers, and docs.
- Keep the base bootstrap lean. Observability and GitOps stay optional unless explicitly requested otherwise.
- Keep the shared Gateway separate from API installation. Installing Gateway API CRDs should not silently create the cluster Gateway resource.
- Keep certificate issuer policy opt-in. Installing `cert-manager` should not silently create a default issuer.
- Treat `cloud-provider-kind` as a shared host service. Do not make cluster deletion automatically shut it down.
- Keep registry workflows host-side, not in-cluster, unless there is a deliberate reason to change that.
- Treat `k9s` as a core client tool, not an optional extra.
- Pin versions for external dependencies wherever practical.

## Important files

- `Makefile`: primary orchestration entrypoint
- `kind/cluster.yaml.tpl`: template for the rendered kind config
- `README.md`: operator-facing usage notes
- `.state/`: rendered config and ephemeral local state
- `.state/kubeconfig`: workspace-scoped kubeconfig for this cluster

## Expected workflow

Typical sequence:

```sh
make kind-preflight
make tools-check
make kind-bootstrap-base
make kind-install-gateway-api-gateway
make kind-install-cert-manager-issuer
make kind-install-observability
make kind-install-argocd
```

If cluster creation is failing early, use:

```sh
make kind-create-debug
```

Common image flow:

```sh
docker build -t localhost:5001/my-app:dev .
docker push localhost:5001/my-app:dev
kubectl apply -f k8s/
```

## Editing guidance

- Prefer updating the Makefile and template-driven config together so behavior and docs stay aligned.
- If you change registry-related variables, ensure the rendered kind config, docs, and workflow examples still make sense.
- Follow the current kind local-registry pattern based on `hosts.toml`; do not reintroduce the older CRI mirror patch approach.
- If you change bootstrap dependencies, update both `tools-check` and `README.md`.
- Prefer the workspace-scoped kubeconfig in `.state/kubeconfig`; do not rely on the user’s global current-context being set correctly.
- Do not add Docker Compose unless the host-side support stack grows beyond the current simple registry/service model.
- Do not replace `local-path-provisioner` with a heavier storage stack without a clear use case.

## Validation

When changing bootstrap behavior:

- run `make kind-sync-state`
- run `make kind-preflight`
- run `make tools-check`
- run `make kind-render-config`
- use `make -n <target>` for dry checks when live bootstrap is not possible
- use `make kind-create-debug` when kind fails before the cluster is up and you need the failed node preserved

If `cloud-provider-kind` is unavailable, fail clearly rather than silently degrading the Gateway API story.
