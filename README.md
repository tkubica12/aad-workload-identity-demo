# AAD workload identity demo
This repo contains simple demos for AAD workload identity federation with identity providers such as internal Kubernetes, GitHub or custom.

## Kubernetes
Use case is to federate internal Kubernetes identity which is behing Service Accounts with Azure AD allowing for AAD identity to be mapped to Service Account in Kubernetes. Then application can get Kubernetes token and exchange it for AAD token which than can be used to access services such as Azure, Azure Key Vault, Azure SQL etc. This allows to retrieve AAD tokens WITHOUT any need for keeping password or certificate. Get token from Kubernetes by injecting token Pod, then exchange it for AAD token - no passwords involved.

[Guide](./Kubernetes.md)