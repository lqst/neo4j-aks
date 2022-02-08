# Neo4j on Azure AKS

## Prerequisites

Verify Microsoft.OperationsManagement and Microsoft.OperationalInsights are registered on your subscription. To check the registration status:

```shell
az provider show -n Microsoft.OperationsManagement -o table
az provider show -n Microsoft.OperationalInsights -o table
```

If they are not registered, register Microsoft.OperationsManagement and Microsoft.OperationalInsights using:

```shell
az provider register --namespace Microsoft.OperationsManagement
az provider register --namespace Microsoft.OperationalInsights
```


## Create Resource group and aks cluster

```shell
az group create --name rg_aks --location northeurope
az aks create \
    --resource-group rg_aks \
    --name akscluster \
    --node-count 1 \
    --enable-addons monitoring \
    --generate-ssh-keys
```

## Add role assignment
```shell
AKS_PRINCIPAL_ID="$(az aks show --resource-group rg_aks --name akscluster \
   --query 'identity.principalId' --output tsv)"

az role assignment create --resource-group rg_aks --role "Virtual Machine Contributor" \
                          --assignee-object-id "${AKS_PRINCIPAL_ID}"

az role assignment create --resource-group rg_aks --role "Network Contributor" \
    --assignee-object-id "${AKS_PRINCIPAL_ID}"
```

## Connect to cluster
```shell
az aks get-credentials --resource-group rg_aks --name akscluster
```


## Clean up/delete resource group
```shell
az group delete --name rg_aks --yes --no-wait
```


## Helm for standalone

Create a persistent volume
```shell
az disk create \
    --resource-group rg_aks \
    --name "my-neo4j-disk" \
    --size-gb "64" \
    --sku Premium_LRS \
    --max-shares 1
# Disk id needed for volume mount later on
az disk show --resource-group rg_aks --name "my-neo4j-disk" --query id
```

Create a static public ip address
az network public-ip create -g rg_aks -n myPublicIp --dns-name neopublic --allocation-method Static

```shell
helm install single neo4j/neo4j-standalone -f single.yaml
helm uninstall single
```

### Cypher shell
```shell
kubectl run --rm -it --image "neo4j:4.4.3-enterprise" cypher-shell \
     -- cypher-shell -a "neo4j://single.default.svc.cluster.local:7687" -u neo4j -p "test1234"
```

### Port forward
```shell
kubectl port-forward service/single 7470:7474 7680:7687
```


## Azure AD APP
http://localhost:7470/browser/?idp_id=myIdp&auth_flow_step=redirect_uri


AD Group Object ID 16903965-5ac8-4522-abac-851469103279



dbms.security.authorization_providers=oidc-azure-oidc,native
dbms.security.authentication_providers=oidc-azure-oidc,native
## For the Discovery API:
dbms.security.oidc.azure-oidc.display_name=Azure SSO
dbms.security.oidc.azure-oidc.auth_flow=pkce
dbms.security.oidc.azure-oidc.auth_endpoint=https://login.microsoftonline.com/54e85725-ed2a-49a4-a19e-11c8d29f9a0f/oauth2/v2.0/authorize
dbms.security.oidc.azure-oidc.token_endpoint=https://login.microsoftonline.com/54e85725-ed2a-49a4-a19e-11c8d29f9a0f/oauth2/v2.0/token
dbms.security.oidc.azure-oidc.redirect_uri=http://localhost:7470/browser/?idp_id=azure-oidc&auth_flow_step=redirect_uri
dbms.security.oidc.azure-oidc.params=client_id=774d9fcd-ff80-4b5c-87cc-9fa4615c0871;response_type=code;scope=openid profile email
dbms.security.oidc.azure-oidc.config=principal=unique_name;code_challenge_method=S256
dbms.security.oidc.azure-oidc.authorization.group_to_role_mapping=16903965-5ac8-4522-abac-851469103279=admin
## Settings relevant for OIDC auth:
dbms.security.oidc.azure-oidc.well_known_discovery_uri=https://login.microsoftonline.com/54e85725-ed2a-49a4-a19e-11c8d29f9a0f/v2.0/.well-known/openid-configuration
dbms.security.oidc.azure-oidc.audience=774d9fcd-ff80-4b5c-87cc-9fa4615c0871
dbms.security.oidc.azure-oidc.claims.username=preferred_username
dbms.security.oidc.azure-oidc.claims.groups=groups
dbms.security.oidc.azure-oidc.claims.groups=roles;









