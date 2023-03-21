# Gateway Container With Luna HSM Integration

## Description
This sample project is based on apim-charts Gateway Helm chart tag [gateway-3.0.5](https://github.com/CAAPIM/apim-charts/tree/gateway-3.0.5)
to build and run a derived Gateway container with embedded Luna HSM client software.

## Prerequisites/Dependencies
* For building the derived Gateway container image:
   * Luna HSM 7 server and the Luna Minimal Client software (must be obtained from the vendor)
   * Network access from the build machine to Luna HSM server
* For running the Gateway image, refer to the apim-charts/gateway [prerequisites](https://github.com/CAAPIM/apim-charts/tree/gateway-3.0.5/charts/gateway#prerequisites)

## Build the derived Gateway image
Follow the instructions in [dockerfile/README.md](dockerfile/README.md) to build the derived Gateway container
with the Luna client software installed.

## Deploying the Gateway
The commands in this example are run from the `gateway-luna-helm-sample` directory.

1. Follow the apim-charts/gateway [documentation](https://github.com/CAAPIM/apim-charts/blob/gateway-3.0.5/charts/gateway/README.md)
   for general configuration of the Gateway Helm chart. Instructions for installing/upgrading/uninstalling the chart are also provided there.
   User-supplied Helm chart values can be included in the [luna-values.yaml](helm-example/luna-values.yaml)
   such as the derived Gateway image with Luna client installed.

1. Set the namespace in [kustomization.yaml](helm-example/kustomization.yaml).

1. Create the ssg.security secret.
   ```
   kubectl apply -k helm-example
   ```

   Other methods of setting the ssg.security file such as ConfigMap and Init Containers are
   demonstrated at [Layer7-Community/Utilities gateway-init-container-examples)](https://github.com/Layer7-Community/Utilities/tree/main/gateway-init-container-examples)).

1. Place your Gateway license file at `helm-example/license.xml` and if you agree, you will need to set `license.accept` to true when deploying the chart. 
To run this example, you must accept the license agreement.

1. Deploy the Gateway. 
   ```
   helm install --set-file "license.value=helm-example/license.xml" --set "license.accept=true" --repo https://caapim.github.io/apim-charts/ <DEPLOYMENT-NAME> gateway --version 3.0.5 --values helm-example/luna-values.yaml --namespace <NAMESPACE>
   ```

1. Enable Luna via Policy Manager following Step 5, tasks 2 to 5 from [TechDocs: Configure the SafeNet Luna HSM Client v10.2](https://techdocs.broadcom.com/us/en/ca-enterprise-software/layer7-api-management/api-gateway/10-1/install-configure-upgrade/configure-the-appliance-gateway/configure-hardware-security-modules-hsm/configure-safenet-luna-sa-hsm-parent/configure-the-safenet-luna-hsm-client-v102.html).

1. Once Luna is enabled, scale Gateway container cluster size to 0. Wait for all containers to be terminated before proceeding to the next step.
   ```
   kubectl scale deployments --namespace <NAMESPACE> <DEPLOYMENT-NAME>-gateway --replicas 0
   ```

1. Once all containers are removed, scale Gateway container cluster size to 1. Wait for container to be 
ready and connect using Policy Manager to verify Luna HSM has been successfully enabled on Gateway.
Then scale up as desired.
   ```
   kubectl scale deployments --namespace <NAMESPACE> <DEPLOYMENT-NAME>-gateway --replicas 1
   ```