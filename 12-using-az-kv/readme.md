# Using Azure Key Vault to store secrets

## Set up

- Install latest version of the `aks-preview` extension or update to the latest version
  ```bash
  az extension add --name aks-preview
  or
  az extension update --name aks-preview
  ```
- Existing Azure Subscription with EnableWorkloadIdentityPreview feature enabled

Update vars.sh file to include few more variables for using Azure Key Vault

```yaml
SUBSCRIPTION_ID="__change_me__" #subscription id
UAMI="__change_me__"     #name for user assigned identity
KEYVAULT_NAME="__change_me__" #keyvault name
IDENTITY_TENANT="__change_me__" #tenant id
```

Source the file through with `source vars.sh`.

## Upgrade existing AKS cluster

Upgrade your AKS cluster with Azure Key Vaut Provider for Secrets store CSI Driver capability, enable-oidc-issuer

```bash
az aks enable-addons --addons azure-keyvault-secrets-provider --name $AKS_NAME --resource-group $RES_GROUP
az aks update -g $RES_GROUP -n $AKS_NAME --enable-oidc-issuer

```

## Create Azure KeyVault

Create an Azure Key Vault resource to store secrets. You can store secret in your key vault using the `az keyvaul secret set` command

```bash
az keyvault create -n $KEYVAULT_NAME -g $RES_GROUP -l $REGION
```

```bash
az keyvault secret set --vault-name $KEYVAULT_NAME -n admin-passsword --value kv_supersecret
```

## Configure Workload Identity

First set a specific subscription to be the current active subscription:

```bash
az account set --subscription $SUBSCRIPTION_ID
```

Create managed identity using `az identity create`:

```bash
az identity create --name $UAMI --resource-group $RES_GROUP

```

The following command outputs the user assigned client id. Save this output
in the `vars.sh` as USER_ASSIGNED_CLIENT_ID.

```bash
az identity show -g $RES_GROUP --name $UAMI --query 'clientId' -o tsv
```

In `vars.sh`, set the following variable:

```yaml
USER_ASSIGNED_CLIENT_ID="__change_me__"
```

Run `source vars.sh` before moving the next step.

You need to grant the workload identity permission to access the Key Vault
secrets, keys and certificates by setting acesss policy.

```bash
az keyvault set-policy -n $KEYVAULT_NAME --key-permissions get --spn $USER_ASSIGNED_CLIENT_ID
az keyvault set-policy -n $KEYVAULT_NAME --secret-permissions get --spn $USER_ASSIGNED_CLIENT_ID
az keyvault set-policy -n $KEYVAULT_NAME --certificate-permissions get --spn $USER_ASSIGNED_CLIENT_ID
```

```bash
export AKS_OIDC_ISSUER="$(az aks show --resource-group $RES_GROUP --name $AKS_NAME --query "oidcIssuerProfile.issuerUrl" -o tsv)"
echo $AKS_OIDC_ISSUER
```

_If URL is empty, verify you have installed the latest version of the `aks-preview` extension andd OIDC issuer is enabled._

Update the values for `name` and `namespace` with the Kubernetes service account name and its namespace in the `service.yaml` file and apply the manifest.

<details markdown="1">
<summary>Click here for the Service YAML file</summary>

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: ${USER_ASSIGNED_CLIENT_ID}
  labels:
    azure.workload.identity/use: "true"
  name: "__change_me__" # service account name
  namespace: "_change_me__" # namespace of your workload
```

</details>

```bash
kubectl apply -f service.yaml
```

Create the federated identity credential between the Managed Identity, the service account issuer, and the subject.

```bash
export federatedIdentityName="aksfederatedidentity" # can be changed as needed
az identity federated-credential create --name $federatedIdentityName --identity-name $UAMI --resource-group $RES_GROUP --issuer ${AKS_OIDC_ISSUER} --subject system:serviceaccount:${serviceAccountNamespace}:${serviceAccountName}
```

In the secret-provider-class, change `ClientID`, `keyvaultName` and `tenantID` with the worload identity, keyvault name and the tenant ID of the keyvault. Update `ObjectName` with the name of your secret. Deploy the `SecretProviderClass` after making the changes.

<details markdown="2">
<summary>Click here for the Secret Provider Class YAML</summary>

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-kvname-workload-identity # needs to be unique per namespace, can be change as needed
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "false"
    clientID: "__change_me___" # USER_ASSIGNED_CLIENT_ID
    keyvaultName: "__change_me___" # Set to the name of your key vault
    cloudName: "" # [OPTIONAL for Azure] if not provided, the Azure environment defaults to AzurePublicCloud
    objects: |
      array:
        - |
          objectName: "__change_me__"
          objectType: secret
          objectVersion: ""
    tenantId: "__change_me__" # The tenant ID of the key vault
```

</details>

```bash
kubectl apply -f secret-provider-class.yaml
```

Change the `serviceAccountName` then apply pod to the cluster.

<details markdown="2">
<summary>Click here for the Mongo Deployment YAML</summary>

```yaml
kind: Deployment
apiVersion: apps/v1

metadata:
  name: mongodb

spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      serviceAccountName: "__change_me__" # service account name
      containers:
        - name: mongodb-container

          image: mongo:5.0
          imagePullPolicy: Always

          ports:
            - containerPort: 27017

          resources:
            requests:
              cpu: 100m
              memory: 200Mi
            limits:
              cpu: 500m
              memory: 300Mi

          readinessProbe:
            exec:
              command:
                - mongo
                - --eval
                - db.adminCommand('ping')

          volumeMounts:
            - name: secrets-store01-inline
              mountPath: "/mnt/secrets-store"
              readOnly: true

      volumes:
        - name: secrets-store01-inline
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: "azure-kvname-workload-identity" # Update to match Secret Provider Class name if needed
```

</details>

```bash
kubectl apply -f mongo-deployment.yaml
```

## Validate Secrets

After the pod starts, the mounted content at the volume path that you specified in your deployment YAML is available.

```bash
## show secrets held in secrets-store
kubectl exec <pod_name> -- ls /mnt/secrets-store/

## print a test secret 'ExampleSecret' held in secrets-store
kubectl exec <pod_name> -- cat /mnt/secrets-store/<secret_name>
```

### References:

- [Use the Azure Key Vault Provider for Secrets Store CSI Driver in an AKS cluster](https://learn.microsoft.com/en-us/azure/aks/csi-secrets-store-driver)
- [Provide an identity to access the Azure Key Vault Provider for Secrets Store CSI Driver](https://learn.microsoft.com/en-us/azure/aks/csi-secrets-store-identity-access)
