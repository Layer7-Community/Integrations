# Deploying the Grafana Stack and Prometheus
This guide details how to deploy the [Grafana LGTM Stack](https://grafana.com/about/grafana-stack/) and the [Prometheus Stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) into your Kubernetes cluster. We use public Helm Charts in both stacks that you can refer to and configure

These tools showcase a preview feature with [OpenTelemetry](https://opentelemetry.io/) and the Layer7 API Gateway. Please refer to [Techdocs](https://techdocs.broadcom.com/us/en/ca-enterprise-software/layer7-api-management/api-gateway/11-1/install-configure-upgrade/configuring-opentelemetry-for-the-gateway.html) and the [Gateway Helm Chart](https://github.com/CAAPIM/apim-charts/tree/stable/charts/gateway) for more details on how to configure the Gateway for OpenTelemetry.

## Component Overview
As part of this guide we will deploy the following components
- ***L***oki (Logs)
  - Promtail
- ***G***rafana (UI)
- ***T***empo (Traces)
- ***M***imir (Metrics)
- Prometheus Stack (Cluster Monitoring)

***NOTE:*** This guide also includes the OpenTelemetry Operator, while the OpenTelemetry Operator is not a requirement, it makes configuring the Gateway for OpenTelemetry simpler.
- OpenTelemetry Operator
    - Certmanager (dependency)


![grafana-stack](./images/grafana-stack-diagram.png)

## Prerequisites
- Kubernetes v1.26+
- Ingress Controller (important for grafana)
  - the quickstart examples deploy the nginx ingress controller

You will also need to add the following entries to your hosts file

If you're running Kind locally
```
127.0.0.1 grafana.brcmlabs.com
```
If you're running Kind on a remote VM
```
<VIRTUAL-MACHINE-IP> grafana.brcmlabs.com
i.e.
192.168.1.40 grafana.brcmlabs.com
```
***NOTE*** If you are using an existing Kubernetes Cluster you can retrieve the correct address after the Prometheus Stack has been deployed

```
kubectl get ingress -n monitoring
```
output
```
NAME                 CLASS   HOSTS                  ADDRESS     PORTS     AGE
prometheus-grafana   nginx   grafana.brcmlabs.com   <ip-address>   80, 443   57m
```
/etc/hosts
```
<ip-address> grafana.brcmlabs.com
```

### Getting started
This example uses multiple namespaces for the additional components. Your Kubernetes user or service account must have sufficient privileges to create namespaces, deployments, configmaps, secrets, service accounts, roles, etc.

The following namespaces will be created/used if you use the [quickstart](#quickstart) option to deploy this example
- monitoring (prometheus, grafana)
- grafana-loki (loki, tempo, promtail)
- ingress-nginx (nginx ingress controller)

If you deploy the OpenTelemetry Operator and Certmanager
- cert-manager
- opentelemetry-operator-system

#### Clone this repository
```
git clone https://github.com/Layer7-Community/Integrations.git
```
Open a terminal and navigate to this examples directory
```
cd /path/to/cloned/repository/Integrations/grafana-stack-prometheus-otel
```

### Prometheus values pre-configuration

#### Ingress Configuration (Grafana)
If you wish to use a different host and or configure a signed certificate for Grafana, update the ingress configuration in [prometheus-values.yaml](./prometheus/prometheus-values.yaml)
```
grafana:
  ingress:
    enabled: true
    ingressClassName: nginx
    labels: {}
    hosts:
      - grafana.brcmlabs.com
    path: /
    tls:
    - secretName: brcmlabs
      hosts:
      - grafana.brcmlabs.com
```
#### Grafana Admin Password
The default credentials to login to Grafana are `admin/7layer` If you wish to change the default Grafana admin password update the following in [prometheus-values.yaml](./prometheus/prometheus-values.yaml)
```
grafana:
  adminPassword: 7layer
```

## Guide
* [Quickstart](#quickstart)
    * [Using an existing Kubernetes Cluster](#existing-kubernetes-cluster)
    * [Using Kind](#kind)
* [Deploy the Prometheus Stack](#deploy-the-promtheus-stack)
* [Deploy the Grafana Stack](#deploy-the-grafana-stack)
* [Deploy the OpenTelemetry Operator](#deploy-the-opentelemetry-operator)
* [Create an OpenTelemetryCollector](#create-an-opentelemetry-collector)
* [Verify Deployment](#verify-deployment)
* [Next steps](#next-steps)
* [Uninstall](#uninstall)

## Quickstart
A Makefile is included in the example directory that makes deploying this example a one step process. If you have access to a Docker Machine you can use [Kind](https://kind.sigs.k8s.io/) (Kubernetes in Docker). This example can optionally deploy a Kind Cluster for you (you just need to make sure that you've [installed Kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation))

The kind configuration is in the base of the example folder. If your docker machine is remote (you are using a VM or remote machine) then uncomment the network section and set the apiServerAddress to the address of your VM/Remote machine
```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
# networking:
#   apiServerAddress: "192.168.1.64"
#   apiServerPort: 6443
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
```

### Existing Kubernetes Cluster
```
make install
```
if you don't have an ingress controller you can deploy nginx with the following
```
make nginx
```
if you are using kind
```
make nginx-kind
```
#### OpenTelemetry
To deploy the OpenTelemetry Operator (includes cert-manager) and OpenTelemetryCollector + Instrumentation Custom resources, ***recommended*** for this guide.
```
make otel
```

### Kind
```
make kind-cluster install otel nginx-kind
```

### [Now verify everything is working as expected](#verify-deployment)

##################################################################
### Manual Deployment
##################################################################

## Deploy the Prometheus Stack
We use the following [Helm Chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) to deploy Prometheus

- Add the prometheus-community Helm Chart Repo
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```
- Create the Layer7 Dashboard

The dashboard is located in [./prometheus/grafana-dashboard/layer7-gateway-dashboard.json](./prometheus/grafana-dashboard/layer7-gateway-dashboard.json) and contains a simple grafana dashboard that gets populated with telemetry generated from the Gateway's OpenTelemetry integration. You will find more details about the integration on [Techdocs](techdocslink) and the [Gateway Helm Chart](gatewayhelmchartlink)
```
kubectl apply -k ./prometheus/grafana-dashboard/
```
- Deploy the Prometheus Stack
```
helm upgrade -i prometheus -f ./prometheus/prometheus-values.yaml prometheus-community/kube-prometheus-stack -n monitoring
```

## Deploy the Grafana Stack
The [Grafana LGTM](https://grafana.com/about/grafana-stack/) (Loki, Grafana, Tempo, Mimir) stack provides a great all in one solution for observing traces, metrics and logs from Kubernetes and other infrastructure. If you would like to manage more than Kubernetes, or more than a single cluster you can check out their cloud offering [here](https://grafana.com/products/cloud/).

In this example, Grafana is configured in the prometheus stack and we are not making use of [mimir](https://grafana.com/oss/mimir/)

- Add the grafana Helm Chart Repo
```
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

- Deploy Loki

Loki is deployed via [this Helm Chart](https://github.com/grafana/loki/tree/main/production/helm/loki)
```
helm upgrade --install --values ./grafana-stack/loki-overrides.yaml loki grafana/loki -n grafana-loki --create-namespace
```

- Deploy Promtail

Promtail is deployed via [this Helm Chart](https://github.com/grafana/helm-charts/tree/main/charts/promtail)
```
helm upgrade --install --values ./grafana-stack/promtail-overrides.yaml promtail grafana/promtail -n grafana-loki
```

- Deploy Tempo

Tempo is deployed via [this Helm Chart](https://github.com/grafana/helm-charts/tree/main/charts/tempo)
```
helm upgrade --install --values ./grafana-stack/tempo-overrides.yaml tempo grafana/tempo -n grafana-loki
```

- Deploy Mimir

Mimir is deployed via [this Helm Chart](https://github.com/grafana/helm-charts/tree/main/charts/mimir-distributed)
```
helm upgrade --install --values ./grafana-stack/mimir-distributed-overrides.yaml mimir grafana/mimir-distributed -n grafana-loki
```

## Deploy the OpenTelemetry Operator
the OpenTelemetry Operator is not a requirement, it makes configuring the Gateway for OpenTelemetry significantly simpler and works well with this example guide

- Deploy Cert-Manager (OTel Operator dependency)
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.5/cert-manager.yaml
```
verify cert-manager has been successfully deployed
```
kubectl wait --for=condition=ready --timeout=600s pod -l app=cert-manager -n cert-manager
kubectl wait --for=condition=ready --timeout=600s pod -l app=cainjector -n cert-manager
kubectl wait --for=condition=ready --timeout=600s pod -l app=webhook -n cert-manager
```

- Deploy the OpenTelemetry Operator
```
kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/download/v0.97.1/opentelemetry-operator.yaml
```
verify that the OpenTelemetry Operator has been successfully deployed
```
kubectl wait --for=condition=ready --timeout=600s pod -l app.kubernetes.io/name=opentelemetry-operator -n opentelemetry-operator-system
```

## Create an OpenTelemetry Collector
[OpenTelemetry Collectors](https://opentelemetry.io/docs/collector/) provide a vendor-agnostic way to receive, process and export telemetry data. The OpenTelemetry Operator manages a custom resource called an OpenTelemetryCollector. We define an OpenTelemetry collector for the Gateway [here](./otel/collector.yaml).

In our example we are sending telemetry to the components that we configured earlier, OpenTelemetry is [supported by 20+ vendors](https://opentelemetry.io/ecosystem/vendors/). You are not restricted to this Observability stack, it is highly likely that your existing Observability Platform supports OpenTelemetry.

The Gateway, when configured correctly will have a sidecar automatically injected by the OpenTelemetry Operator based on what we have in this configuration.
```
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: gateway-opentelemetry-collector
spec:
  ***mode: sidecar***
```

In this example we also have a file called [instrumentation.yaml](./otel/instrumentation.yaml) which facilitates auto-instrumentation of the Gateway Container, this injects an initContainer into the Gateway Deployment that inserts and configures the OpenTelemetry Agent. This is ***optional***, the Layer7 Gateway includes the OpenTelemetry SDK. Including the OpenTelemetry Agent captures additional telemetry including but not limited to JVM Metrics. Please refer to [Techdocs](https://techdocs.broadcom.com/us/en/ca-enterprise-software/layer7-api-management/api-gateway/11-1/install-configure-upgrade/configuring-opentelemetry-for-the-gateway.html) for more details

- Create OpenTelemetry Collector + Instrumentation
```
kubectl apply -f ./otel/
```

## Verify Deployment
Confirm all Grafana stack pods are up and running
```
kubectl get pods -n grafana-loki
```
output
```
NAME                                       READY   STATUS      RESTARTS   AGE
loki-0                                     1/1     Running     0          4m
loki-canary-hr5n6                          1/1     Running     0          4m
loki-chunks-cache-0                        2/2     Running     0          4m
loki-gateway-5cd888fb5b-lx68d              1/1     Running     0          4m
loki-minio-0                               1/1     Running     0          4m
loki-results-cache-0                       2/2     Running     0          4m
mimir-alertmanager-0                       1/1     Running     0          2m7s
mimir-compactor-0                          1/1     Running     0          2m7s
mimir-distributor-75db4845b5-fwgx5         1/1     Running     0          2m7s
mimir-ingester-0                           1/1     Running     0          2m7s
mimir-ingester-1                           1/1     Running     0          2m7s
mimir-make-minio-buckets-5.0.14-9ddtx      0/1     Completed   0          2m7s
mimir-minio-66c9c9446c-vr6tx               1/1     Running     0          2m7s
mimir-nginx-6c54df9bbf-xw5wm               1/1     Running     0          2m7s
mimir-overrides-exporter-75c74b879-l4vd2   1/1     Running     0          2m7s
mimir-querier-f4c7668c7-c2lkr              1/1     Running     0          2m7s
mimir-query-frontend-86f49bdd54-57njx      1/1     Running     0          2m7s
mimir-query-scheduler-9c9db55-ns8bw        1/1     Running     0          2m7s
mimir-rollout-operator-589445cccd-2gktd    1/1     Running     0          2m7s
mimir-ruler-87b575f97-xxv8z                1/1     Running     0          2m7s
mimir-store-gateway-0                      1/1     Running     0          2m7s
promtail-92scb                             1/1     Running     0          3m25s
tempo-0                                    1/1     Running     0          3m17s
```

- Confirm all datasources are configured correctly in Grafana
    - Open a browser and navigate to `https://grafana.brcmlabs.com` if you changed the grafana ingress host, navigate to the host that you configured. Accept the certificate warning and proceed to authenticate.
      
      The default username is `admin` with password `7layer`. If you updated the default grafana admin password, use the password that you configured.
    - Select the datasources tab on the left side menu and proceed to test them.
      - Loki
      - Prometheus
      - Tempo

    The animated gif below depicts the process you will need to follow.

    ![grafana-datasource-gif](./images/grafana-datasource-gif.gif)


## Next Steps
Now that you've configured the Observability stack it's time to head to the [Gateway Helm Chart](https://github.com/CAAPIM/apim-charts/tree/stable/charts/gateway) and configure your Gateway for OpenTelemetry. We have also included [values file](./gateway-example/gateway-otel-values.yaml) in this example that you shows all of the configuration overrides.

## Uninstall
If you used kind
```
make uninstall-kind
```
If you used an existing Kubernetes Cluster
```
make uninstall
```

- If you deployed nginx

If you used kind
```
make uninstall-nginx-kind
```
If you used an existing Kubernetes Cluster
```
make uninstall-nginx
```

- If you deployed the OpenTelemetry Operator
```
make uninstall-otel
```