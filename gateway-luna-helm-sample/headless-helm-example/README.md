## Description

This sample helm chart requires a derived **Gateway image 11.1.2** that can connect to Luna HSM 7.

## Container GW Headless deployment in Kubernetes without Policy Manager
1. Set disklessConfig.enabled to false. Create node.properites file with node.cluster.pass value set. Sample node.properties is for Derby configuration
   Reference README for APIM Gateway charts GibHub Pages [Diskless Configuration](https://github.com/CAAPIM/apim-charts/blob/stable/charts/gateway/README.md#diskless-configuration) for MySQL database configuration

   NOTE: Note that this can also be created with tools like [kustomize](https://kustomize.io/) which will be better for CI/CD pipelines. 
   You can also take advantage of the secret [secret store csi driver](https://secrets-store-csi-driver.sigs.k8s.io/) to mount this secret from an external KMS provider.

2. create Kubernetes Secret
```
   kubectl create secret generic gateway-secret --from-file=node.properties=./node.properties
```
3. Encrypt LunaPIN by OpenSSL, password is value from node.cluster.pass
   
   NOTE: Ensure to use openssl 3.2.x version for optimal compatibility.
```
   echo -n '<LunaPIN>' | openssl enc -e -aes-256-cbc -pbkdf2 -iter 600000 -saltlen 16 -pass pass:<gateway_cluster_passphrase> -a;history -d $(history 1)
```
4. Append the following system property to config.systemProperties and place encrypted LunaPIN to com.l7tech.encryptedLunaPin in luna_headless_value.yaml

   #LUNA properties
   com.l7tech.common.security.jceProviderEngineName=luna
   com.l7tech.luna.installAsLeastPreference=true
   com.l7tech.encryptedLunaPin='<EncryptedLunaPartitionPassword>'
   com.l7tech.lunaPartitionLabel=development
   com.l7tech.server.keyStore.defaultSsl.alias=-1:<alias of the default SSL key, the default is SSL>
   com.l7tech.common.security.disableJceProviderFallback=false

5. Deploy the Gateway. The Gateway is now using Luna HSM as keystore.
   ```
   helm install --set-file "license.value=helm-example/license.xml" --set "license.accept=true" --repo https://caapim.github.io/apim-charts/ <DEPLOYMENT-NAME> gateway --values headless-helm-example/luna_headless_value.yaml --namespace <NAMESPACE>
   ```