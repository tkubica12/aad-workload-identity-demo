# Create AAD identity
```bash
az ad sp create-for-rbac --name tomas1aad
```

# Create Resource Group and make tomas1aad Contributor of it
```bash
az group create -n githubdemo -l westeurope
az role assignment create --role "Contributor" \
  --assignee-object-id $(az ad sp list --display-name tomas1aad --query [0].objectId -o tsv) \
  -g githubdemo
```

# Estabilish trust between specific GitHub repo+environment and AAD account
```bash
export repo="tkubica12/aad-workload-identity-demo"
# export environment="azurecloud"
export APPLICATION_CLIENT_ID="$(az ad sp list --display-name tomas1aad --query '[0].appId' -otsv)"
export APPLICATION_OBJECT_ID="$(az ad app show --id ${APPLICATION_CLIENT_ID} --query objectId -otsv)"
cat <<EOF > body.json
{
  "name": "github-federated-credential",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:$repo:ref:refs/heads/main",
  "description": "GitHub federated credential",
  "audiences": [
    "api://AzureADTokenExchange"
  ]
}
EOF

az rest --method POST --uri "https://graph.microsoft.com/beta/applications/${APPLICATION_OBJECT_ID}/federatedIdentityCredentials" --body @body.json
rm body.json
```



# Clean up
az ad sp delete --id $(az ad sp list --display-name tomas1aad --query [0].objectId -o tsv)
az group delete -n aks -y

