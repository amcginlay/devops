export KUBECONFIG := $(CURDIR)/kubeconfig.yaml

.PHONY: help preflight cluster-up cluster-down get-kubeconfig install-client-tools install-platform get-grafana-admin-password

help: ## Show available targets
	@awk 'BEGIN { FS = ":.*## "; dollar = sprintf("%c", 36); printf "Available targets:\n" } /^[a-zA-Z0-9_.-]+:.*## / { desc = $$2; while ((pos = index(desc, dollar dollar)) > 0) desc = substr(desc, 1, pos - 1) dollar substr(desc, pos + 2); targets[++count] = $$1; descriptions[count] = desc; if (length($$1) > width) width = length($$1) } END { for (i = 1; i <= count; i++) printf "  make %-" width "s  %s\n", targets[i], descriptions[i] }' $(MAKEFILE_LIST)

preflight: ## Verify system meets requirements (macOS, Apple Silicon, Parallels)
	@echo "Running preflight checks..."
	@[ "$$(uname)" = "Darwin" ] || { echo "FAIL: macOS is required"; exit 1; }
	@[ "$$(uname -m)" = "arm64" ] || { echo "FAIL: Apple Silicon (arm64) is required"; exit 1; }
	@command -v brew >/dev/null 2>&1 || { echo "FAIL: Homebrew is not installed"; exit 1; }
	@command -v vagrant >/dev/null 2>&1 || { echo "FAIL: Vagrant is not installed"; exit 1; }
	@command -v prlctl >/dev/null 2>&1 || { echo "FAIL: Parallels Desktop is not installed"; exit 1; }
	@echo "All preflight checks passed."

cluster-up: preflight ## Start the VM, create the K8s cluster and provision a standard platform
	vagrant up --provider=parallels
	$(MAKE) install-platform

cluster-down: ## Destroy the VM and all its resources
	vagrant destroy -f

get-kubeconfig: ## Download the kubeconfig from the VM for host access
	@vagrant ssh -c 'kind get kubeconfig --name k8s-wizard' 2>/dev/null | sed 's|https://0\.0\.0\.0:|https://127.0.0.1:|' > kubeconfig.yaml
	@echo "Kubeconfig written to kubeconfig.yaml"
	@echo "Usage: export KUBECONFIG=$$(pwd)/kubeconfig.yaml"

install-client-tools: ## Install kubectl, helm and k9s on the host via Homebrew
	brew install kubectl helm k9s

get-grafana-admin-password: ## Print the Grafana admin password
	@kubectl get secret kube-prometheus-stack-grafana -n monitoring -o jsonpath='{.data.admin-password}' | base64 -d && echo

install-platform: install-client-tools get-kubeconfig ## Install platform components (cert-manager, Gateway API, Nginx Gateway Fabric, metrics-server, Prometheus stack)
	@echo "==> Installing cert-manager with self-signed cluster-wide issuer..."
	helm repo add jetstack https://charts.jetstack.io --force-update
	helm upgrade --install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --set crds.enabled=true --wait
	kubectl apply -f manifests/selfsigned-clusterissuer.yaml
	@echo "==> Installing Gateway API CRDs..."
	kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/latest/download/standard-install.yaml
	@echo "==> Installing Nginx Gateway Fabric..."
	helm upgrade --install nginx-gateway oci://ghcr.io/nginx/charts/nginx-gateway-fabric --namespace nginx-gateway --create-namespace --set nginx.service.type=NodePort --set "nginx.service.nodePorts[0].port=30443" --set "nginx.service.nodePorts[0].listenerPort=443" --wait
	@echo "==> Creating default Gateway with TLS..."
	kubectl apply -f manifests/default-gateway.yaml
	@echo "==> Installing metrics-server..."
	helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/ --force-update
	helm upgrade --install metrics-server metrics-server/metrics-server --namespace kube-system --set args[0]=--kubelet-insecure-tls --wait
	@echo "==> Installing Prometheus stack (Prometheus, Grafana, Alertmanager)..."
	helm repo add prometheus-community https://prometheus-community.github.io/helm-charts --force-update
	helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace --wait
	kubectl apply -f manifests/grafana-httproute.yaml
	kubectl apply -f manifests/prometheus-httproute.yaml
	kubectl apply -f manifests/alertmanager-httproute.yaml
