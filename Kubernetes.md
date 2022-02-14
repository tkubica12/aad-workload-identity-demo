# Enable OIDC issuer feature for AKS
```bash
az feature register --name EnableOIDCIssuerPreview --namespace Microsoft.ContainerService
az feature list -o table --query "[?contains(name, 'Microsoft.ContainerService/EnableOIDCIssuerPreview')].{Name:name,State:properties.state}"  # wait for Registered
az provider register --namespace Microsoft.ContainerService
az extension add --name aks-preview --upgrade -y
```

# Create AKS
```bash
az group create -n aks -l westeurope
az aks create -n aks -g aks -c 1 -s Standard_B2s -x --enable-oidc-issuer
```

# Create AAD identity
```bash
az ad sp create-for-rbac --name tomas1aad
```

# Create Key Vault, store secret and make tomas1aad reader of secret
```bash
az keyvault create -n tomas1aadkeyvault \
  -g aks \
  --enable-rbac-authorization \
  --default-action Allow

az role assignment create --role "Key Vault Secrets Officer" \
  --assignee-object-id $(az ad signed-in-user show --query objectId -otsv) \  # Myself - to be able to create secret
  --scope $(az keyvault show -n tomas1aadkeyvault -g aks --query id -otsv)

az keyvault secret set -n mysecret --vault-name tomas1aadkeyvault --value mysecretvalue

az role assignment create --role "Key Vault Secrets User" \
  --assignee-object-id $(az ad sp list --display-name tomas1aad --query [0].objectId -o tsv) \
  --scope $(az keyvault show -n tomas1aadkeyvault -g aks --query id -otsv)
```

# Get AKS cluster credentials
```bash
az aks get-credentials -n aks -g aks --overwrite-existing
```

# Install workload identity webhook
```bash
export AZURE_TENANT_ID="$(az account show --query tenantId -otsv)"
helm repo add azure-workload-identity https://azure.github.io/azure-workload-identity/charts
helm repo update
helm install workload-identity-webhook azure-workload-identity/workload-identity-webhook \
   --namespace azure-workload-identity-system \
   --create-namespace \
   --set azureTenantID="${AZURE_TENANT_ID}"
```

# Create Service Account
```bash
export APPLICATION_CLIENT_ID="$(az ad sp list --display-name tomas1aad --query '[0].appId' -otsv)"
export SERVICE_ACCOUNT_ISSUER=$(az aks show -n aks -g aks --query "oidcIssuerProfile.issuerUrl" -o tsv)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: ${APPLICATION_CLIENT_ID}
  labels:
    azure.workload.identity/use: "true"
  name: tomas1kube
  namespace: default
EOF
```

# Estabilish trust between Kubernetes and AAD account
```bash
export APPLICATION_CLIENT_ID="$(az ad sp list --display-name tomas1aad --query '[0].appId' -otsv)"
export APPLICATION_OBJECT_ID="$(az ad app show --id ${APPLICATION_CLIENT_ID} --query objectId -otsv)"
cat <<EOF > body.json
{
  "name": "kubernetes-federated-credential",
  "issuer": "${SERVICE_ACCOUNT_ISSUER}",
  "subject": "system:serviceaccount:default:tomas1kube",
  "description": "Kubernetes service account federated credential",
  "audiences": [
    "api://AzureADTokenExchange"
  ]
}
EOF

az rest --method POST --uri "https://graph.microsoft.com/beta/applications/${APPLICATION_OBJECT_ID}/federatedIdentityCredentials" --body @body.json
rm body.json
```

# Deploy application
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: testpod
  namespace: default
spec:
  serviceAccountName: tomas1kube
  containers:
    - image: ubuntu
      name: mycontainer
      command: ['tail', '-f', '/dev/null']
EOF
```

# Enter application container a compare standard Kubernetes token with Kubernetes token scoped for 
```bash
kubectl exec -ti testpod -- bash
cat /var/run/secrets/kubernetes.io/serviceaccount/token
cat /var/run/secrets/tokens/azure-identity-token 
```

Standard Kubernetes token

```json
{
  "aud": [
    "https://oidc.prod-aks.azure.com/25da173c-6b43-4f93-9c41-6cc688dea1ff/"
  ],
  "exp": 1676374730,
  "iat": 1644838730,
  "iss": "https://oidc.prod-aks.azure.com/25da173c-6b43-4f93-9c41-6cc688dea1ff/",
  "kubernetes.io": {
    "namespace": "default",
    "pod": {
      "name": "testpod",
      "uid": "4f5edc59-911b-4d46-a534-af1c19fb1815"
    },
    "serviceaccount": {
      "name": "tomas1kube",
      "uid": "c83f100a-1443-4bec-89ad-9d6e67e844e6"
    },
    "warnafter": 1644842337
  },
  "nbf": 1644838730,
  "sub": "system:serviceaccount:default:tomas1kube"
}
```

Kubernetes token scoped for AzureADTokenExchange

```json
{
  "aud": [
    "api://AzureADTokenExchange"
  ],
  "exp": 1644842330,
  "iat": 1644838730,
  "iss": "https://oidc.prod-aks.azure.com/25da173c-6b43-4f93-9c41-6cc688dea1ff/",
  "kubernetes.io": {
    "namespace": "default",
    "pod": {
      "name": "testpod",
      "uid": "4f5edc59-911b-4d46-a534-af1c19fb1815"
    },
    "serviceaccount": {
      "name": "tomas1kube",
      "uid": "c83f100a-1443-4bec-89ad-9d6e67e844e6"
    }
  },
  "nbf": 1644838730,
  "sub": "system:serviceaccount:default:tomas1kube"
}
```

# Exchange token with AAD for Azure Key Vault scope
```bash
apt update
apt install jq -y

scope="https%3A%2F%2Fvault.azure.net%2F.default"
output=$(curl -X POST \
    "https://login.microsoftonline.com/$AZURE_TENANT_ID/oauth2/v2.0/token" \
    -d "scope=$scope&client_id=$AZURE_CLIENT_ID&client_assertion=$(cat $AZURE_FEDERATED_TOKEN_FILE)&grant_type=client_credentials&client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3Aclient-assertion-type%3Ajwt-bearer")

echo $output | jq -r .access_token | iconv -f ascii -t UTF-16LE > token
```

# See decoded AAD token after exchange
```json
{
  "aud": "https://vault.azure.net",
  "iss": "https://sts.windows.net/cac7b9c4-b5d4-4966-b6e4-39bef0bebb46/",
  "iat": 1644833800,
  "nbf": 1644833800,
  "exp": 1644837700,
  "aio": "E2ZgYFjzoyNpn/GNXrbtM9n0H3g1AgA=",
  "appid": "2ced0f74-1f9a-4456-94d9-d54a069fa687",
  "appidacr": "2",
  "idp": "https://sts.windows.net/cac7b9c4-b5d4-4966-b6e4-39bef0bebb46/",
  "oid": "7ed01f19-8c08-4802-a20c-7a351169a76c",
  "rh": "0.ATsAxLnHytS1Zkm25Dm-8L67RjmzqM-ighpHo8kPwL56QJM7AAA.",
  "sub": "7ed01f19-8c08-4802-a20c-7a351169a76c",
  "tid": "cac7b9c4-b5d4-4966-b6e4-39bef0bebb46",
  "uti": "KSqMh5G1nUOknVwujUUxAA",
  "ver": "1.0"
}
```

# Authenticate with service such as Azure Key Vault
```bash
curl https://tomas1aadkeyvault.vault.azure.net//secrets?api-version=7.2 \
  -H "Authorization: Bearer $(cat ./token)"

  {"value":[{"id":"https://tomas1aadkeyvault.vault.azure.net/secrets/mysecret","attributes":{"enabled":true,"created":1644833715,"updated":1644833715,"recoveryLevel":"Recoverable+Purgeable","recoverableDays":90},"tags":{"file-encoding":"utf-8"}}],"nextLink":null}
```

# Clean up
az ad sp delete --id $(az ad sp list --display-name tomas1aad --query [0].objectId -o tsv)
az group delete -n aks -y
