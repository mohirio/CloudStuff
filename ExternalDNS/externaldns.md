# External DNS for Kubernetes

Note: this guide is incomplete.

Export the following environment variables
```bash
export HOST_NAME=""
export NODE_RESOURCE_GROUP=""
export DNS_RESOURCE_GROUP=""
```
Create a DNS Zone on the same resource group as the cluster

```bash
az network dns zone create -g $DNS_RESOURCE_GROUP -n $HOST_NAME
```

## Setting permissions

The external DNS needs permissions to make changes to the DNS server.

First, create a service principal, it might take a moment.

```bash
az ad sp create-for-rbac -n ExternalDnsServicePrincipal
```

This will output a JSON containing ```appId``` which will be used later
```json
{
  "appId": "appId GUID",
  "password": "password",
  ...
}
```

To assign the rights, we need the resource group ID and zone resource ID
```bash
# Main resource group ID 
> az group show -g $NODE_RESOURCE_GROUP
{
  "id": "/subscriptions/id/resourceGroups/externaldns",
  ...
}

# Zone group resource ID
> az network dns zone show --name $HOST_NAME -resource-group $DNS_RESOURCE_GROUP
{
  "id": "/subscriptions/.../resourceGroups/externaldns/providers/Microsoft.Network/dnszones/$HOST_NAME",
  ...
}
```

Create the roles using the above two along with ```appId``` from the previous 
step
```bash
az role assignment create --role "Reader" --assignee "appId" --scope "resource group id" 
az role assignment create --role "Contributor" --assignee "appId" --scope "dns zone resource id"
```

Finally, we need to create a JSON file that will be used to configure Kubernetes 
secret.

For tenantId and subscriptionId
```bash
az account show --query "tenantId"
az account show --query "id"
```

Populate ```azure.json``` with the following
```json
{
  "tenantId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "subscriptionId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "resourceGroup": "DNS",
  "aadClientId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "aadClientSecret": "xxxxxxxxxxxxxxx"
}
```

Use the created JSON file to add the secret to kubernetes
```bash
kubectl create secret generic azure-config-file --from-file=azure.json
```

## Deployment (Todo)

```bash
kubectl create -f externaldns.yaml
```

Deploying nginx service

```bash
kubectl create -f nginx-externaldns.yaml
```


To verify deployment
```bash
az network dns record-set a list -g $DNS_RESOURCE_GROUP -z $HOST_NAME
```
