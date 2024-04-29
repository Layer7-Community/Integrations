# This Makefile drives the quickstart sections of all examples.
wait:
ifeq ($(OS),Windows_NT)
	TIMEOUT /T $(t)
else
	sleep $(t)
endif

.PHONY: grafana-stack prometheus otel

install: prometheus grafana-stack

grafana-stack:
	helm repo add grafana https://grafana.github.io/helm-charts
	helm upgrade --install --values ./grafana-stack/loki-overrides.yaml loki grafana/loki -n grafana-loki --create-namespace
	helm upgrade --install --values ./grafana-stack/promtail-overrides.yaml promtail grafana/promtail -n grafana-loki
	helm upgrade --install --values ./grafana-stack/tempo-overrides.yaml tempo grafana/tempo -n grafana-loki
	helm upgrade --install --values ./grafana-stack/mimir-distributed-overrides.yaml mimir grafana/mimir-distributed -n grafana-loki

prometheus:
	helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
	helm repo update
	kubectl apply -k ./prometheus/grafana-dashboard/
	helm upgrade -i prometheus -f ./prometheus/prometheus-values.yaml prometheus-community/kube-prometheus-stack -n monitoring

nginx-kind:
	kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
	@$(MAKE) --silent t=10 wait
	kubectl wait --for=condition=ready --timeout=600s pod -l app.kubernetes.io/name=ingress-nginx,app.kubernetes.io/component=controller -n ingress-nginx

nginx:
	kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
	@$(MAKE) --silent t=10 wait
	kubectl wait --for=condition=ready --timeout=600s pod -l app.kubernetes.io/name=ingress-nginx,app.kubernetes.io/component=controller -n ingress-nginx

otel:
	kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.5/cert-manager.yaml
	@$(MAKE) --silent t=10 wait
	kubectl wait --for=condition=ready --timeout=600s pod -l app=cert-manager -n cert-manager
	kubectl wait --for=condition=ready --timeout=600s pod -l app=cainjector -n cert-manager
	kubectl wait --for=condition=ready --timeout=600s pod -l app=webhook -n cert-manager
	kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/download/v0.97.1/opentelemetry-operator.yaml
	@$(MAKE) --silent t=10 wait
	kubectl wait --for=condition=ready --timeout=600s pod -l app.kubernetes.io/name=opentelemetry-operator -n opentelemetry-operator-system
	kubectl apply -f ./otel/

kind-cluster:
	kind create cluster --name layer7 --config ./kind-config.yaml

uninstall-kind:
	kind delete cluster --name layer7

uninstall:
	helm uninstall loki -n grafana-loki
	helm uninstall promtail -n grafana-loki
	helm uninstall tempo -n grafana-loki
	helm uninstall mimir -n grafana-loki
	helm uninstall prometheus -n monitoring
	kubectl delete -k ./prometheus/grafana-dashboard/
	kubectl delete pvc export-0-loki-minio-0 export-1-loki-minio-0 storage-mimir-alertmanager-0 storage-mimir-compactor-0 storage-mimir-ingester-0 storage-mimir-ingester-1 storage-mimir-store-gateway-0 -n grafana-loki

uninstall-otel:
	kubectl delete -f ./otel/
	kubectl delete -f https://github.com/open-telemetry/opentelemetry-operator/releases/download/v0.97.1/opentelemetry-operator.yaml
	kubectl delete -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.5/cert-manager.yaml

uninstall-nginx-kind:
	kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

uninstall-nginx:
	kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
