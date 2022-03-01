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

# Work in progress/static ip + fqdn
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
```shell
# Work in progress/static ip + fqdn
az network public-ip create -g rg_aks -n myPublicIp --dns-name neopublic --allocation-method Static
```

Update helm values file (single.yaml) with volume URL/Disk id

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
URL: http://localhost:7470/browser/?idp_id=myIdp&auth_flow_step=redirect_uri

AD Group Object ID and tenant/subscription id:s are needed in neo4j.conf


# Todo
 - [ ] Azure cli section for creating App/AD Group
 - [ ] Domain of some sort + https
 - [ ] Reduce copy paste