SHELL := /bin/sh

STATE_DIR := .state
ABS_STATE_DIR := $(CURDIR)/$(STATE_DIR)
CLUSTER_NAME_FILE := $(STATE_DIR)/cluster-name
CLOUD_KUBECONFIG_FILE := $(STATE_DIR)/kubeconfig
CLOUD_PROVIDER_PID_FILE := $(STATE_DIR)/cloud-provider-kind.pid
CLOUD_PROVIDER_LOG_FILE := $(STATE_DIR)/cloud-provider-kind.log

export KUBECONFIG := $(CURDIR)/$(CLOUD_KUBECONFIG_FILE)

KIND_CLUSTER_CONFIG_TEMPLATE := kind/cluster.yaml.tpl
KIND_CLUSTER_CONFIG := $(STATE_DIR)/kind-cluster.yaml
GATEWAY_API_CERTIFICATE_MANIFEST := kind/gateway-api-certificate.yaml
GATEWAY_API_GATEWAY_MANIFEST := kind/gateway-api-gateway.yaml
ALERTMANAGER_HTTPROUTE_MANIFEST := kind/alertmanager-httproute.yaml
GRAFANA_HTTPROUTE_MANIFEST := kind/grafana-httproute.yaml
LOCAL_SELF_SIGNED_CLUSTER_ISSUER_MANIFEST := kind/local-self-signed-clusterissuer.yaml
PROMETHEUS_HTTPROUTE_MANIFEST := kind/prometheus-httproute.yaml
REGISTRY_NAME ?= kind-registry
REGISTRY_PORT ?= 5001
REGISTRY_CONTAINER_PORT ?= 5000
REGISTRY_IMAGE ?= registry:2.8.3
KIND_NODE_IMAGE ?= kindest/node:v1.35.0@sha256:452d707d4862f52530247495d180205e029056831160e22870e37e3f6c1ac31f
KIND_CREATE_FLAGS ?=

GATEWAY_API_VERSION ?= v1.4.0
CERT_MANAGER_VERSION ?= v1.19.1
METRICS_SERVER_CHART_VERSION ?= 3.13.0
LOCAL_PATH_PROVISIONER_VERSION ?= v0.0.32
KUBE_PROMETHEUS_STACK_CHART_VERSION ?= 80.13.3
ARGO_CD_VERSION ?= v3.2.1

CERT_MANAGER_NAMESPACE ?= cert-manager
MONITORING_NAMESPACE ?= monitoring
ARGO_CD_NAMESPACE ?= argocd
GATEWAY_API_GATEWAY_NAMESPACE ?= gateway-system
GATEWAY_API_GATEWAY_NAME ?= shared
GATEWAY_API_GATEWAY_CLASS_NAME ?= cloud-provider-kind
GATEWAY_API_GATEWAY_LISTENER_NAME ?= https
GATEWAY_API_GATEWAY_TLS_HOSTNAME ?= *.local
GATEWAY_API_GATEWAY_TLS_SECRET_NAME ?= local-wildcard-tls
GATEWAY_API_GATEWAY_CERTIFICATE_NAME ?= local-wildcard
LOCAL_SELF_SIGNED_CLUSTER_ISSUER_NAME ?= local-self-signed
GRAFANA_HOSTNAME ?= grafana.local
GRAFANA_ROUTE_NAME ?= grafana
GRAFANA_SERVICE_NAME ?= kube-prometheus-stack-grafana
GRAFANA_ADMIN_SECRET_NAME ?= kube-prometheus-stack-grafana
GRAFANA_HOSTS_ADDRESS ?= 127.0.0.1
ALERTMANAGER_HOSTNAME ?= alertmanager.local
ALERTMANAGER_ROUTE_NAME ?= alertmanager
ALERTMANAGER_SERVICE_NAME ?= kube-prometheus-stack-alertmanager
ALERTMANAGER_SERVICE_PORT ?= 9093
ALERTMANAGER_HOSTS_ADDRESS ?= 127.0.0.1
PROMETHEUS_HOSTNAME ?= prometheus.local
PROMETHEUS_ROUTE_NAME ?= prometheus
PROMETHEUS_SERVICE_NAME ?= kube-prometheus-stack-prometheus
PROMETHEUS_SERVICE_PORT ?= 9090
PROMETHEUS_HOSTS_ADDRESS ?= 127.0.0.1

.PHONY: help tools-check tools-install tools-install-argocd registry-start registry-stop registry-status kind-sync-state kind-export-kubeconfig kind-shell-env kind-print-grafana-access kind-print-grafana-admin-password kind-print-alertmanager-access kind-print-prometheus-access kind-preflight kind-render-config kind-create kind-create-debug kind-delete kind-reset kind-bootstrap-base kind-bootstrap-full kind-install-base kind-install-gateway-api kind-install-gateway-api-gateway kind-expose-grafana kind-expose-alertmanager kind-expose-prometheus kind-install-storage kind-install-cert-manager kind-install-cert-manager-issuer kind-install-metrics-server kind-install-observability kind-install-argocd kind-start-cloud-provider kind-check-cloud-provider kind-stop-cloud-provider kind-untaint-control-plane kind-configure-local-registry kind-status

help: ## Show available targets
	@awk 'BEGIN { FS = ":.*## "; dollar = sprintf("%c", 36); printf "Available targets:\n" } /^[a-zA-Z0-9_.-]+:.*## / { desc = $$2; while ((pos = index(desc, dollar dollar)) > 0) desc = substr(desc, 1, pos - 1) dollar substr(desc, pos + 2); targets[++count] = $$1; descriptions[count] = desc; if (length($$1) > width) width = length($$1) } END { for (i = 1; i <= count; i++) printf "  make %-" width "s  %s\n", targets[i], descriptions[i] }' $(MAKEFILE_LIST)

tools-check: ## Verify required client tools are installed
	@for tool in docker kind kubectl helm k9s; do \
		if ! command -v "$$tool" >/dev/null 2>&1; then \
			printf 'Missing required tool: %s\n' "$$tool" >&2; \
			exit 1; \
		fi; \
	done
	@if ! command -v cloud-provider-kind >/dev/null 2>&1; then \
		printf '%s\n' \
			'Missing required tool: cloud-provider-kind' \
			'This bootstrap uses Gateway API support via cloud-provider-kind.' \
			'Install it with: make tools-install' >&2; \
		exit 1; \
	fi
	@printf 'Required tools are present: docker kind kubectl helm k9s cloud-provider-kind\n'

tools-install: ## Install kind/kubectl/helm/cloud-provider-kind/k9s with Homebrew
	@command -v brew >/dev/null 2>&1 || { printf 'Homebrew is required for tools-install\n' >&2; exit 1; }
	@brew install kind kubectl helm cloud-provider-kind k9s

tools-install-argocd: ## Install the optional Argo CD CLI with Homebrew
	@command -v brew >/dev/null 2>&1 || { printf 'Homebrew is required for tools-install-argocd\n' >&2; exit 1; }
	@brew install argocd

registry-start: ## Start the local registry used by kind image pulls
	@if docker inspect "$(REGISTRY_NAME)" >/dev/null 2>&1; then \
		if [ "$$(docker inspect -f '{{.State.Running}}' "$(REGISTRY_NAME)")" = "true" ]; then \
			printf 'Local registry %s is already running on localhost:%s\n' "$(REGISTRY_NAME)" "$(REGISTRY_PORT)"; \
		else \
			printf 'Starting existing local registry %s\n' "$(REGISTRY_NAME)"; \
			docker start "$(REGISTRY_NAME)" >/dev/null; \
		fi; \
	else \
		printf 'Creating local registry %s on localhost:%s using %s\n' "$(REGISTRY_NAME)" "$(REGISTRY_PORT)" "$(REGISTRY_IMAGE)"; \
		docker run -d --restart=always -p "127.0.0.1:$(REGISTRY_PORT):$(REGISTRY_CONTAINER_PORT)" --name "$(REGISTRY_NAME)" "$(REGISTRY_IMAGE)" >/dev/null; \
	fi

registry-stop: ## Stop and remove the local registry container
	@if docker inspect "$(REGISTRY_NAME)" >/dev/null 2>&1; then \
		printf 'Stopping and removing local registry %s\n' "$(REGISTRY_NAME)"; \
		docker rm -f "$(REGISTRY_NAME)" >/dev/null; \
	else \
		printf 'Local registry %s is not present\n' "$(REGISTRY_NAME)"; \
	fi

registry-status: ## Show the local registry container status
	@docker ps -a --filter "name=^/$(REGISTRY_NAME)$$" --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'

kind-sync-state: ## Clear stale local state if the cluster was deleted out of band
	@mkdir -p "$(STATE_DIR)"
	@if [ -f "$(CLUSTER_NAME_FILE)" ]; then \
		cluster_name="$$(cat "$(CLUSTER_NAME_FILE)")"; \
		if ! kind get clusters | grep -Fx "$$cluster_name" >/dev/null 2>&1; then \
			printf 'Clearing stale cluster state for %s\n' "$$cluster_name"; \
			rm -f "$(CLUSTER_NAME_FILE)" "$(CLOUD_KUBECONFIG_FILE)"; \
		fi; \
	fi
	@if [ ! -f "$(CLUSTER_NAME_FILE)" ] && [ -f "$(CLOUD_KUBECONFIG_FILE)" ]; then \
		printf 'Removing workspace kubeconfig with no tracked cluster\n'; \
		rm -f "$(CLOUD_KUBECONFIG_FILE)"; \
	fi
	@if [ -f "$(CLOUD_PROVIDER_PID_FILE)" ] && ! sudo kill -0 "$$(cat "$(CLOUD_PROVIDER_PID_FILE)")" >/dev/null 2>&1; then \
		printf 'Removing stale cloud-provider-kind PID file\n'; \
		rm -f "$(CLOUD_PROVIDER_PID_FILE)"; \
	fi

kind-export-kubeconfig: kind-sync-state ## Refresh the workspace kubeconfig for the tracked cluster
	@cluster_name="$${CLUSTER_NAME:-$$(cat "$(CLUSTER_NAME_FILE)" 2>/dev/null || true)}"; \
	[ -n "$$cluster_name" ] || { printf 'Cluster name not found. Run make kind-create first.\n' >&2; exit 1; }; \
	kind get clusters | grep -Fx "$$cluster_name" >/dev/null 2>&1 || { \
		printf 'Tracked cluster %s does not exist. Run make kind-sync-state or make kind-create.\n' "$$cluster_name" >&2; \
		exit 1; \
	}; \
	kind export kubeconfig --name "$$cluster_name" --kubeconfig "$(KUBECONFIG)" >/dev/null 2>&1; \
	printf 'Exported kubeconfig for %s to %s\n' "$$cluster_name" "$(CLOUD_KUBECONFIG_FILE)"

kind-shell-env: ## Print shell exports for direct kubectl and helm use via eval "$$(make kind-shell-env)"
	@$(MAKE) kind-export-kubeconfig >/dev/null
	@printf 'export KUBECONFIG=%s\n' "$(KUBECONFIG)"

kind-print-grafana-access: kind-export-kubeconfig ## Print the current Grafana hosts entry and practical URL
	@mapped_port="$$(docker ps --format '{{.Ports}}' --filter "name=kindccm-" | sed -n 's/.*0\.0\.0\.0:\([0-9][0-9]*\)->443\/tcp.*/\1/p' | head -n 1)"; \
	if [ -z "$$mapped_port" ]; then \
		mapped_port="$$(docker ps --format '{{.Ports}}' --filter "name=kindccm-" | sed -n 's/.*\[\:\:\]:\([0-9][0-9]*\)->443\/tcp.*/\1/p' | head -n 1)"; \
	fi; \
	printf 'Hosts entry: %s %s\n' "$(GRAFANA_HOSTS_ADDRESS)" "$(GRAFANA_HOSTNAME)"; \
	if [ -n "$$mapped_port" ]; then \
		printf 'Grafana URL: https://%s:%s\n' "$(GRAFANA_HOSTNAME)" "$$mapped_port"; \
	else \
		printf 'Grafana URL: <mapped gateway port not found; inspect docker ps --filter name=kindccm- >\n'; \
	fi

kind-print-grafana-admin-password: ## Print the Grafana admin password from the current secret
	@$(MAKE) kind-export-kubeconfig >/dev/null
	@kubectl get secret --namespace "$(MONITORING_NAMESPACE)" "$(GRAFANA_ADMIN_SECRET_NAME)" -o jsonpath='{.data.admin-password}' | base64 --decode; printf '\n'

kind-print-alertmanager-access: kind-export-kubeconfig ## Print the current Alertmanager hosts entry and practical URL
	@mapped_port="$$(docker ps --format '{{.Ports}}' --filter "name=kindccm-" | sed -n 's/.*0\.0\.0\.0:\([0-9][0-9]*\)->443\/tcp.*/\1/p' | head -n 1)"; \
	if [ -z "$$mapped_port" ]; then \
		mapped_port="$$(docker ps --format '{{.Ports}}' --filter "name=kindccm-" | sed -n 's/.*\[\:\:\]:\([0-9][0-9]*\)->443\/tcp.*/\1/p' | head -n 1)"; \
	fi; \
	printf 'Hosts entry: %s %s\n' "$(ALERTMANAGER_HOSTS_ADDRESS)" "$(ALERTMANAGER_HOSTNAME)"; \
	if [ -n "$$mapped_port" ]; then \
		printf 'Alertmanager URL: https://%s:%s\n' "$(ALERTMANAGER_HOSTNAME)" "$$mapped_port"; \
	else \
		printf 'Alertmanager URL: <mapped gateway port not found; inspect docker ps --filter name=kindccm- >\n'; \
	fi

kind-print-prometheus-access: kind-export-kubeconfig ## Print the current Prometheus hosts entry and practical URL
	@mapped_port="$$(docker ps --format '{{.Ports}}' --filter "name=kindccm-" | sed -n 's/.*0\.0\.0\.0:\([0-9][0-9]*\)->443\/tcp.*/\1/p' | head -n 1)"; \
	if [ -z "$$mapped_port" ]; then \
		mapped_port="$$(docker ps --format '{{.Ports}}' --filter "name=kindccm-" | sed -n 's/.*\[\:\:\]:\([0-9][0-9]*\)->443\/tcp.*/\1/p' | head -n 1)"; \
	fi; \
	printf 'Hosts entry: %s %s\n' "$(PROMETHEUS_HOSTS_ADDRESS)" "$(PROMETHEUS_HOSTNAME)"; \
	if [ -n "$$mapped_port" ]; then \
		printf 'Prometheus URL: https://%s:%s\n' "$(PROMETHEUS_HOSTNAME)" "$$mapped_port"; \
	else \
		printf 'Prometheus URL: <mapped gateway port not found; inspect docker ps --filter name=kindccm- >\n'; \
	fi

kind-preflight: tools-check kind-sync-state registry-start ## Check tools, Docker, state, registry, and shared services
	@docker info >/dev/null
	@printf 'Docker connectivity check passed\n'
	@if [ -f "$(CLUSTER_NAME_FILE)" ]; then \
		cluster_name="$$(cat "$(CLUSTER_NAME_FILE)")"; \
		printf 'Tracked cluster: %s\n' "$$cluster_name"; \
		kind get clusters | grep -Fx "$$cluster_name" >/dev/null 2>&1 || { \
			printf 'Tracked cluster %s is stale\n' "$$cluster_name" >&2; \
			exit 1; \
		}; \
		$(MAKE) kind-export-kubeconfig >/dev/null; \
		kubectl get nodes >/dev/null; \
		printf 'Workspace kubeconfig is usable\n'; \
	else \
		printf 'Tracked cluster: <none>\n'; \
	fi
	@if [ -f "$(CLOUD_PROVIDER_PID_FILE)" ] && sudo kill -0 "$$(cat "$(CLOUD_PROVIDER_PID_FILE)")" >/dev/null 2>&1; then \
		printf 'cloud-provider-kind appears to be running\n'; \
	else \
		printf 'cloud-provider-kind is not currently running\n'; \
	fi

kind-render-config:
	@mkdir -p "$(STATE_DIR)"
	@sed \
		-e 's|__REGISTRY_NAME__|$(REGISTRY_NAME)|g' \
		-e 's|__REGISTRY_PORT__|$(REGISTRY_PORT)|g' \
		-e 's|__KIND_NODE_IMAGE__|$(KIND_NODE_IMAGE)|g' \
		"$(KIND_CLUSTER_CONFIG_TEMPLATE)" > "$(KIND_CLUSTER_CONFIG)"
	@printf 'Rendered %s\n' "$(KIND_CLUSTER_CONFIG)"

kind-create: tools-check kind-sync-state registry-start kind-render-config ## Create a control-plane-only kind cluster
	@mkdir -p "$(STATE_DIR)"
	@cluster_name="$${CLUSTER_NAME:-k8s-$$(date +"%y%m%d%H%M")}"; \
	kind get clusters | grep -Fx "$$cluster_name" >/dev/null 2>&1 && { \
		printf 'Cluster %s already exists. Use make kind-install-base or make kind-delete first.\n' "$$cluster_name" >&2; \
		exit 1; \
	} || true; \
	printf 'Creating kind cluster %s\n' "$$cluster_name"; \
	kind create cluster $(KIND_CREATE_FLAGS) --name "$$cluster_name" --config "$(KIND_CLUSTER_CONFIG)" && \
	printf '%s\n' "$$cluster_name" > "$(CLUSTER_NAME_FILE)" && \
	kind export kubeconfig --name "$$cluster_name" --kubeconfig "$(KUBECONFIG)"
	@$(MAKE) kind-configure-local-registry >/dev/null

kind-create-debug: ## Create the cluster with kind retention enabled for debugging
	@$(MAKE) kind-create KIND_CREATE_FLAGS='--retain'

kind-delete: kind-sync-state ## Delete the cluster without stopping shared services
	@cluster_name="$${CLUSTER_NAME:-$$(cat "$(CLUSTER_NAME_FILE)" 2>/dev/null || true)}"; \
	if [ -n "$$cluster_name" ]; then \
		printf 'Deleting kind cluster %s\n' "$$cluster_name"; \
		kind delete cluster --name "$$cluster_name"; \
	else \
		printf 'No cluster name recorded in %s\n' "$(CLUSTER_NAME_FILE)" >&2; \
	fi
	@rm -f "$(CLUSTER_NAME_FILE)" "$(CLOUD_KUBECONFIG_FILE)"

kind-reset: kind-delete kind-bootstrap-base ## Delete and fully recreate the base cluster

kind-bootstrap-base: kind-create kind-install-base ## Create the cluster and install the base platform

kind-bootstrap-full: kind-bootstrap-base kind-install-observability kind-install-argocd ## Bootstrap base + observability + Argo CD

kind-install-base: kind-export-kubeconfig kind-untaint-control-plane kind-start-cloud-provider kind-install-gateway-api kind-install-storage kind-install-cert-manager kind-install-metrics-server ## Install Gateway API CRDs, storage, cert-manager, and metrics-server

kind-configure-local-registry: ## Connect the local registry to the cluster and publish registry hosting info
	@cluster_name="$${CLUSTER_NAME:-$$(cat "$(CLUSTER_NAME_FILE)" 2>/dev/null)}"; \
	[ -n "$$cluster_name" ] || { printf 'Cluster name not found. Run make kind-create first.\n' >&2; exit 1; }; \
	printf 'Connecting %s to the kind network\n' "$(REGISTRY_NAME)"; \
	if [ "$$(docker inspect -f='{{json .NetworkSettings.Networks.kind}}' "$(REGISTRY_NAME)")" = "null" ]; then \
		docker network connect kind "$(REGISTRY_NAME)"; \
	fi; \
	registry_dir="/etc/containerd/certs.d/localhost:$(REGISTRY_PORT)"; \
	node_name="$${cluster_name}-control-plane"; \
	printf 'Configuring containerd registry hosts for %s\n' "$$node_name"; \
	docker exec "$$node_name" mkdir -p "$$registry_dir"; \
	printf '%s\n' \
		'[host."http://$(REGISTRY_NAME):$(REGISTRY_CONTAINER_PORT)"]' | docker exec -i "$$node_name" cp /dev/stdin "$$registry_dir/hosts.toml"; \
	printf 'Publishing local registry details inside the cluster\n'; \
	printf '%s\n' \
		'apiVersion: v1' \
		'kind: ConfigMap' \
		'metadata:' \
		'  name: local-registry-hosting' \
		'  namespace: kube-public' \
		'data:' \
		'  localRegistryHosting.v1: |' \
		'    host: "localhost:$(REGISTRY_PORT)"' \
		'    help: "https://kind.sigs.k8s.io/docs/user/local-registry/"' | kubectl apply -f -

kind-untaint-control-plane: ## Allow workloads and load balancers on the control-plane node
	@cluster_name="$${CLUSTER_NAME:-$$(cat "$(CLUSTER_NAME_FILE)" 2>/dev/null)}"; \
	[ -n "$$cluster_name" ] || { printf 'Cluster name not found. Run make kind-create first.\n' >&2; exit 1; }; \
	node_name="$${cluster_name}-control-plane"; \
	printf 'Allowing workloads and load balancers on %s\n' "$$node_name"; \
	kubectl taint nodes "$$node_name" node-role.kubernetes.io/control-plane- >/dev/null 2>&1 || true; \
	kubectl label node "$$node_name" node.kubernetes.io/exclude-from-external-load-balancers- >/dev/null 2>&1 || true

kind-start-cloud-provider: ## Start the shared cloud-provider-kind process
	@mkdir -p "$(STATE_DIR)"
	@if [ -f "$(CLOUD_PROVIDER_PID_FILE)" ] && sudo kill -0 "$$(cat "$(CLOUD_PROVIDER_PID_FILE)")" >/dev/null 2>&1; then \
		printf 'cloud-provider-kind already running with PID %s\n' "$$(cat "$(CLOUD_PROVIDER_PID_FILE)")"; \
	else \
		printf 'Starting shared cloud-provider-kind in the background\n'; \
		sudo -v && \
		sudo /bin/sh -c 'nohup cloud-provider-kind --gateway-channel standard --enable-lb-port-mapping >"$(ABS_STATE_DIR)/cloud-provider-kind.log" 2>&1 & echo $$! >"$(ABS_STATE_DIR)/cloud-provider-kind.pid"'; \
	fi
	@printf 'cloud-provider-kind log: %s\n' "$(CLOUD_PROVIDER_LOG_FILE)"
	@$(MAKE) kind-check-cloud-provider >/dev/null

kind-check-cloud-provider: ## Verify that cloud-provider-kind is running
	@if [ ! -f "$(CLOUD_PROVIDER_PID_FILE)" ]; then \
		printf 'cloud-provider-kind PID file not found\n' >&2; \
		exit 1; \
	fi
	@sleep 2
	@if ! sudo kill -0 "$$(cat "$(CLOUD_PROVIDER_PID_FILE)")" >/dev/null 2>&1; then \
		printf 'cloud-provider-kind failed to stay running\n' >&2; \
		[ -f "$(CLOUD_PROVIDER_LOG_FILE)" ] && tail -n 20 "$(CLOUD_PROVIDER_LOG_FILE)" >&2 || true; \
		exit 1; \
	fi
	@printf 'cloud-provider-kind is running (PID %s)\n' "$$(cat "$(CLOUD_PROVIDER_PID_FILE)")"

kind-stop-cloud-provider: ## Stop the shared cloud-provider-kind process
	@if [ -f "$(CLOUD_PROVIDER_PID_FILE)" ]; then \
		pid="$$(cat "$(CLOUD_PROVIDER_PID_FILE)")"; \
		if sudo kill -0 "$$pid" >/dev/null 2>&1; then \
			printf 'Stopping shared cloud-provider-kind (PID %s)\n' "$$pid"; \
			sudo kill "$$pid"; \
		fi; \
		rm -f "$(CLOUD_PROVIDER_PID_FILE)"; \
	fi

kind-install-gateway-api: ## Install Gateway API CRDs
	@printf 'Installing Gateway API CRDs %s\n' "$(GATEWAY_API_VERSION)"
	@kubectl apply --server-side=true -f "https://github.com/kubernetes-sigs/gateway-api/releases/download/$(GATEWAY_API_VERSION)/standard-install.yaml"
	@kubectl wait --for=condition=Established --timeout=120s crd/gatewayclasses.gateway.networking.k8s.io
	@kubectl wait --for=condition=Established --timeout=120s crd/gateways.gateway.networking.k8s.io
	@kubectl wait --for=condition=Established --timeout=120s crd/httproutes.gateway.networking.k8s.io

kind-install-gateway-api-gateway: kind-export-kubeconfig kind-start-cloud-provider kind-install-gateway-api kind-install-cert-manager-issuer ## Install a shared HTTPS Gateway resource and wildcard certificate for Gateway API routes
	@printf 'Installing wildcard certificate %s/%s for %s\n' "$(GATEWAY_API_GATEWAY_NAMESPACE)" "$(GATEWAY_API_GATEWAY_CERTIFICATE_NAME)" "$(GATEWAY_API_GATEWAY_TLS_HOSTNAME)"
	@kubectl create namespace "$(GATEWAY_API_GATEWAY_NAMESPACE)" --dry-run=client -o yaml | kubectl apply -f -
	@sed \
		-e 's|__GATEWAY_API_GATEWAY_NAMESPACE__|$(GATEWAY_API_GATEWAY_NAMESPACE)|g' \
		-e 's|__GATEWAY_API_GATEWAY_CERTIFICATE_NAME__|$(GATEWAY_API_GATEWAY_CERTIFICATE_NAME)|g' \
		-e 's|__GATEWAY_API_GATEWAY_TLS_SECRET_NAME__|$(GATEWAY_API_GATEWAY_TLS_SECRET_NAME)|g' \
		-e 's|__GATEWAY_API_GATEWAY_TLS_HOSTNAME__|$(GATEWAY_API_GATEWAY_TLS_HOSTNAME)|g' \
		-e 's|__LOCAL_SELF_SIGNED_CLUSTER_ISSUER_NAME__|$(LOCAL_SELF_SIGNED_CLUSTER_ISSUER_NAME)|g' \
		"$(GATEWAY_API_CERTIFICATE_MANIFEST)" | kubectl apply -f -
	@kubectl wait --namespace "$(GATEWAY_API_GATEWAY_NAMESPACE)" --for=condition=Ready --timeout=180s certificate/"$(GATEWAY_API_GATEWAY_CERTIFICATE_NAME)"
	@if kubectl get gateway --namespace "$(GATEWAY_API_GATEWAY_NAMESPACE)" "$(GATEWAY_API_GATEWAY_NAME)" >/dev/null 2>&1; then \
		printf 'Recreating Gateway %s/%s to refresh listener and proxy state\n' "$(GATEWAY_API_GATEWAY_NAMESPACE)" "$(GATEWAY_API_GATEWAY_NAME)"; \
		kubectl delete gateway --namespace "$(GATEWAY_API_GATEWAY_NAMESPACE)" "$(GATEWAY_API_GATEWAY_NAME)" --wait=true; \
	fi
	@printf 'Installing shared HTTPS Gateway %s/%s with GatewayClass %s\n' "$(GATEWAY_API_GATEWAY_NAMESPACE)" "$(GATEWAY_API_GATEWAY_NAME)" "$(GATEWAY_API_GATEWAY_CLASS_NAME)"
	@sed \
		-e 's|__GATEWAY_API_GATEWAY_NAMESPACE__|$(GATEWAY_API_GATEWAY_NAMESPACE)|g' \
		-e 's|__GATEWAY_API_GATEWAY_NAME__|$(GATEWAY_API_GATEWAY_NAME)|g' \
		-e 's|__GATEWAY_API_GATEWAY_CLASS_NAME__|$(GATEWAY_API_GATEWAY_CLASS_NAME)|g' \
		-e 's|__GATEWAY_API_GATEWAY_LISTENER_NAME__|$(GATEWAY_API_GATEWAY_LISTENER_NAME)|g' \
		-e 's|__GATEWAY_API_GATEWAY_TLS_HOSTNAME__|$(GATEWAY_API_GATEWAY_TLS_HOSTNAME)|g' \
		-e 's|__GATEWAY_API_GATEWAY_TLS_SECRET_NAME__|$(GATEWAY_API_GATEWAY_TLS_SECRET_NAME)|g' \
		"$(GATEWAY_API_GATEWAY_MANIFEST)" | kubectl apply -f -
	@kubectl wait --namespace "$(GATEWAY_API_GATEWAY_NAMESPACE)" --for=condition=Programmed --timeout=180s gateway/"$(GATEWAY_API_GATEWAY_NAME)"

kind-expose-grafana: kind-export-kubeconfig kind-install-observability kind-install-gateway-api-gateway ## Expose Grafana through the shared HTTPS Gateway
	@printf 'Installing Grafana HTTPRoute %s for host %s\n' "$(GRAFANA_ROUTE_NAME)" "$(GRAFANA_HOSTNAME)"
	@sed \
		-e 's|__MONITORING_NAMESPACE__|$(MONITORING_NAMESPACE)|g' \
		-e 's|__GATEWAY_API_GATEWAY_NAMESPACE__|$(GATEWAY_API_GATEWAY_NAMESPACE)|g' \
		-e 's|__GATEWAY_API_GATEWAY_NAME__|$(GATEWAY_API_GATEWAY_NAME)|g' \
		-e 's|__GATEWAY_API_GATEWAY_LISTENER_NAME__|$(GATEWAY_API_GATEWAY_LISTENER_NAME)|g' \
		-e 's|__GRAFANA_ROUTE_NAME__|$(GRAFANA_ROUTE_NAME)|g' \
		-e 's|__GRAFANA_HOSTNAME__|$(GRAFANA_HOSTNAME)|g' \
		-e 's|__GRAFANA_SERVICE_NAME__|$(GRAFANA_SERVICE_NAME)|g' \
		"$(GRAFANA_HTTPROUTE_MANIFEST)" | kubectl apply -f -
	@kubectl wait --namespace "$(MONITORING_NAMESPACE)" --for=jsonpath='{.status.parents[0].conditions[?(@.type=="Accepted")].status}'=True --timeout=180s httproute/"$(GRAFANA_ROUTE_NAME)"
	@printf '\nGrafana access:\n'
	@$(MAKE) kind-print-grafana-access

kind-expose-alertmanager: kind-export-kubeconfig kind-install-observability kind-install-gateway-api-gateway ## Expose Alertmanager through the shared HTTPS Gateway
	@printf 'Installing Alertmanager HTTPRoute %s for host %s\n' "$(ALERTMANAGER_ROUTE_NAME)" "$(ALERTMANAGER_HOSTNAME)"
	@sed \
		-e 's|__MONITORING_NAMESPACE__|$(MONITORING_NAMESPACE)|g' \
		-e 's|__GATEWAY_API_GATEWAY_NAMESPACE__|$(GATEWAY_API_GATEWAY_NAMESPACE)|g' \
		-e 's|__GATEWAY_API_GATEWAY_NAME__|$(GATEWAY_API_GATEWAY_NAME)|g' \
		-e 's|__GATEWAY_API_GATEWAY_LISTENER_NAME__|$(GATEWAY_API_GATEWAY_LISTENER_NAME)|g' \
		-e 's|__ALERTMANAGER_ROUTE_NAME__|$(ALERTMANAGER_ROUTE_NAME)|g' \
		-e 's|__ALERTMANAGER_HOSTNAME__|$(ALERTMANAGER_HOSTNAME)|g' \
		-e 's|__ALERTMANAGER_SERVICE_NAME__|$(ALERTMANAGER_SERVICE_NAME)|g' \
		-e 's|__ALERTMANAGER_SERVICE_PORT__|$(ALERTMANAGER_SERVICE_PORT)|g' \
		"$(ALERTMANAGER_HTTPROUTE_MANIFEST)" | kubectl apply -f -
	@kubectl wait --namespace "$(MONITORING_NAMESPACE)" --for=jsonpath='{.status.parents[0].conditions[?(@.type=="Accepted")].status}'=True --timeout=180s httproute/"$(ALERTMANAGER_ROUTE_NAME)"
	@printf '\nAlertmanager access:\n'
	@$(MAKE) kind-print-alertmanager-access

kind-expose-prometheus: kind-export-kubeconfig kind-install-observability kind-install-gateway-api-gateway ## Expose Prometheus through the shared HTTPS Gateway
	@printf 'Installing Prometheus HTTPRoute %s for host %s\n' "$(PROMETHEUS_ROUTE_NAME)" "$(PROMETHEUS_HOSTNAME)"
	@sed \
		-e 's|__MONITORING_NAMESPACE__|$(MONITORING_NAMESPACE)|g' \
		-e 's|__GATEWAY_API_GATEWAY_NAMESPACE__|$(GATEWAY_API_GATEWAY_NAMESPACE)|g' \
		-e 's|__GATEWAY_API_GATEWAY_NAME__|$(GATEWAY_API_GATEWAY_NAME)|g' \
		-e 's|__GATEWAY_API_GATEWAY_LISTENER_NAME__|$(GATEWAY_API_GATEWAY_LISTENER_NAME)|g' \
		-e 's|__PROMETHEUS_ROUTE_NAME__|$(PROMETHEUS_ROUTE_NAME)|g' \
		-e 's|__PROMETHEUS_HOSTNAME__|$(PROMETHEUS_HOSTNAME)|g' \
		-e 's|__PROMETHEUS_SERVICE_NAME__|$(PROMETHEUS_SERVICE_NAME)|g' \
		-e 's|__PROMETHEUS_SERVICE_PORT__|$(PROMETHEUS_SERVICE_PORT)|g' \
		"$(PROMETHEUS_HTTPROUTE_MANIFEST)" | kubectl apply -f -
	@kubectl wait --namespace "$(MONITORING_NAMESPACE)" --for=jsonpath='{.status.parents[0].conditions[?(@.type=="Accepted")].status}'=True --timeout=180s httproute/"$(PROMETHEUS_ROUTE_NAME)"
	@printf '\nPrometheus access:\n'
	@$(MAKE) kind-print-prometheus-access

kind-install-storage: ## Install local-path-provisioner and make it the default storage class
	@printf 'Installing local-path-provisioner %s\n' "$(LOCAL_PATH_PROVISIONER_VERSION)"
	@kubectl apply -f "https://raw.githubusercontent.com/rancher/local-path-provisioner/$(LOCAL_PATH_PROVISIONER_VERSION)/deploy/local-path-storage.yaml"
	@kubectl rollout status --namespace local-path-storage deployment/local-path-provisioner --timeout=180s
	@kubectl patch storageclass standard -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}' >/dev/null 2>&1 || true
	@kubectl patch storageclass local-path -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

kind-install-cert-manager: ## Install cert-manager
	@printf 'Installing cert-manager %s\n' "$(CERT_MANAGER_VERSION)"
	@helm upgrade --install cert-manager oci://quay.io/jetstack/charts/cert-manager \
		--namespace "$(CERT_MANAGER_NAMESPACE)" \
		--create-namespace \
		--version "$(CERT_MANAGER_VERSION)" \
		--set crds.enabled=true
	@kubectl rollout status --namespace "$(CERT_MANAGER_NAMESPACE)" deployment/cert-manager --timeout=180s
	@kubectl rollout status --namespace "$(CERT_MANAGER_NAMESPACE)" deployment/cert-manager-webhook --timeout=180s
	@kubectl rollout status --namespace "$(CERT_MANAGER_NAMESPACE)" deployment/cert-manager-cainjector --timeout=180s

kind-install-cert-manager-issuer: kind-export-kubeconfig ## Install an opt-in self-signed ClusterIssuer for cert-manager development flows
	@printf 'Installing local self-signed ClusterIssuer %s\n' "$(LOCAL_SELF_SIGNED_CLUSTER_ISSUER_NAME)"
	@sed \
		-e 's|__LOCAL_SELF_SIGNED_CLUSTER_ISSUER_NAME__|$(LOCAL_SELF_SIGNED_CLUSTER_ISSUER_NAME)|g' \
		"$(LOCAL_SELF_SIGNED_CLUSTER_ISSUER_MANIFEST)" | kubectl apply -f -

kind-install-metrics-server: ## Install metrics-server
	@printf 'Installing metrics-server chart %s\n' "$(METRICS_SERVER_CHART_VERSION)"
	@helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/ --force-update >/dev/null
	@helm repo update >/dev/null
	@helm upgrade --install metrics-server metrics-server/metrics-server \
		--namespace kube-system \
		--version "$(METRICS_SERVER_CHART_VERSION)" \
		--set args[0]=--kubelet-insecure-tls
	@kubectl rollout status --namespace kube-system deployment/metrics-server --timeout=180s

kind-install-observability: ## Install kube-prometheus-stack
	@printf 'Installing kube-prometheus-stack chart %s\n' "$(KUBE_PROMETHEUS_STACK_CHART_VERSION)"
	@helm repo add prometheus-community https://prometheus-community.github.io/helm-charts --force-update >/dev/null
	@helm repo update >/dev/null
	@helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
		--namespace "$(MONITORING_NAMESPACE)" \
		--create-namespace \
		--version "$(KUBE_PROMETHEUS_STACK_CHART_VERSION)"
	@kubectl rollout status --namespace "$(MONITORING_NAMESPACE)" deployment/kube-prometheus-stack-operator --timeout=300s

kind-install-argocd: ## Install Argo CD into the cluster
	@printf 'Installing Argo CD %s\n' "$(ARGO_CD_VERSION)"
	@kubectl create namespace "$(ARGO_CD_NAMESPACE)" --dry-run=client -o yaml | kubectl apply -f -
	@kubectl apply -n "$(ARGO_CD_NAMESPACE)" -f "https://raw.githubusercontent.com/argoproj/argo-cd/$(ARGO_CD_VERSION)/manifests/install.yaml"
	@kubectl rollout status --namespace "$(ARGO_CD_NAMESPACE)" deployment/argocd-server --timeout=300s
	@kubectl rollout status --namespace "$(ARGO_CD_NAMESPACE)" deployment/argocd-repo-server --timeout=300s
	@kubectl rollout status --namespace "$(ARGO_CD_NAMESPACE)" statefulset/argocd-application-controller --timeout=300s

kind-status: kind-sync-state ## Show cluster and add-on status
	@cluster_name="$${CLUSTER_NAME:-$$(cat "$(CLUSTER_NAME_FILE)" 2>/dev/null || true)}"; \
	if [ -n "$$cluster_name" ]; then \
		printf 'Cluster: %s\n' "$$cluster_name"; \
	else \
		printf 'Cluster: <none recorded>\n'; \
	fi
	@if [ -n "$$(cat "$(CLUSTER_NAME_FILE)" 2>/dev/null || true)" ]; then \
		$(MAKE) kind-export-kubeconfig >/dev/null; \
		kubectl get nodes; \
	else \
		printf 'Nodes: <no tracked cluster>\n'; \
	fi
	@printf '\nStorage classes:\n'
	@if [ -n "$$(cat "$(CLUSTER_NAME_FILE)" 2>/dev/null || true)" ]; then kubectl get storageclass; else printf '<no tracked cluster>\n'; fi
	@printf '\nLocal registry:\n'
	@docker ps -a --filter "name=^/$(REGISTRY_NAME)$$" --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
	@printf '\ncloud-provider-kind:\n'
	@if [ -f "$(CLOUD_PROVIDER_PID_FILE)" ] && sudo kill -0 "$$(cat "$(CLOUD_PROVIDER_PID_FILE)")" >/dev/null 2>&1; then \
		printf 'running (PID %s)\n' "$$(cat "$(CLOUD_PROVIDER_PID_FILE)")"; \
	else \
		printf 'not running\n'; \
	fi
	@printf '\nGateway classes:\n'
	@if [ -n "$$(cat "$(CLUSTER_NAME_FILE)" 2>/dev/null || true)" ]; then kubectl get gatewayclass 2>/dev/null || true; else printf '<no tracked cluster>\n'; fi
	@printf '\nGateways:\n'
	@if [ -n "$$(cat "$(CLUSTER_NAME_FILE)" 2>/dev/null || true)" ]; then kubectl get gateway -A 2>/dev/null || true; else printf '<no tracked cluster>\n'; fi
	@printf '\nHTTPRoutes:\n'
	@if [ -n "$$(cat "$(CLUSTER_NAME_FILE)" 2>/dev/null || true)" ]; then kubectl get httproute -A 2>/dev/null || true; else printf '<no tracked cluster>\n'; fi
	@printf '\nCertificates:\n'
	@if [ -n "$$(cat "$(CLUSTER_NAME_FILE)" 2>/dev/null || true)" ]; then kubectl get certificate -A 2>/dev/null || true; else printf '<no tracked cluster>\n'; fi
	@printf '\nGateway access:\n'
	@if [ -n "$$(cat "$(CLUSTER_NAME_FILE)" 2>/dev/null || true)" ]; then \
		mapped_port="$$(docker ps --format '{{.Ports}}' --filter "name=kindccm-" | sed -n 's/.*0\.0\.0\.0:\([0-9][0-9]*\)->443\/tcp.*/\1/p' | head -n 1)"; \
		if [ -z "$$mapped_port" ]; then \
			mapped_port="$$(docker ps --format '{{.Ports}}' --filter "name=kindccm-" | sed -n 's/.*\[\:\:\]:\([0-9][0-9]*\)->443\/tcp.*/\1/p' | head -n 1)"; \
		fi; \
		if [ -n "$$mapped_port" ]; then \
			printf 'Gateway HTTPS host port: %s\n' "$$mapped_port"; \
			kubectl get httproute -A -o jsonpath='{range .items[*]}{.spec.hostnames[0]}{"\n"}{end}' 2>/dev/null | while IFS= read -r hostname; do \
				[ -n "$$hostname" ] || continue; \
				printf 'https://%s:%s\n' "$$hostname" "$$mapped_port"; \
			done; \
		else \
			printf '<mapped gateway port not found; inspect docker ps --filter name=kindccm->\n'; \
		fi; \
	else \
		printf '<no tracked cluster>\n'; \
	fi
	@printf '\nCore add-ons:\n'
	@if [ -n "$$(cat "$(CLUSTER_NAME_FILE)" 2>/dev/null || true)" ]; then \
		kubectl get pods -n "$(CERT_MANAGER_NAMESPACE)" 2>/dev/null || true; \
		kubectl get pods -n local-path-storage 2>/dev/null || true; \
		kubectl get pods -n kube-system -l app.kubernetes.io/name=metrics-server 2>/dev/null || true; \
	else \
		printf '<no tracked cluster>\n'; \
	fi
	@printf '\nNode metrics:\n'
	@if [ -n "$$(cat "$(CLUSTER_NAME_FILE)" 2>/dev/null || true)" ]; then \
		kubectl top nodes 2>/dev/null || printf 'metrics not yet available\n'; \
	else \
		printf '<no tracked cluster>\n'; \
	fi
