.PHONY: help preflight vm-up vm-provision vm-halt vm-suspend vm-destroy get-kubeconfig install-client-tools

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

vm-up: preflight ## Provision and start the Vagrant VM
	vagrant up --provider=parallels

vm-provision: ## Re-run VM provisioning (install/update tools)
	vagrant provision

vm-halt: ## Gracefully shut down the Vagrant VM
	vagrant halt

vm-suspend: ## Suspend the Vagrant VM to disk
	vagrant suspend

vm-destroy: ## Destroy the Vagrant VM and all its resources
	vagrant destroy -f

get-kubeconfig: ## Downloads the kubeconfig from the VM for host access
	@vagrant ssh -c 'kind get kubeconfig --name k8s-wizard' 2>/dev/null | sed 's|https://0\.0\.0\.0:|https://127.0.0.1:|' > kubeconfig.yaml
	@echo "Kubeconfig written to kubeconfig.yaml"
	@echo "Usage: export KUBECONFIG=$$(pwd)/kubeconfig.yaml"

install-client-tools: ## Install kubectl, helm and k9s on the host via Homebrew
	brew install kubectl helm k9s
