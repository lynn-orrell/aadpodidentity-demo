# AAD Pod Identity Demo

Note: Variable values that begin with __ (two underscores) should be populated from the previous command's output. Other values can be named whatever makes sense.

## Create AKS Cluster

export AKS_RESOURCE_GROUP=[AKS_RESOURCE_GROUP]
export LOCATION=westus
export AKS_CLUSTER_NAME=[AKS_CLUSTER_NAME]
export ACR_NAME=[ACR_NAME]
export KEYVAULT_NAME=[KEYVAULT_NAME]

az ad sp create-for-rbac --skip-assignment
export SP_APP_ID=[__APP_ID]
export SP_PASSWORD=[__PASSWORD]

az group create -n $AKS_RESOURCE_GROUP -l $LOCATION
az aks get-versions -l $LOCATION
az aks create -n $AKS_CLUSTER_NAME -g $AKS_RESOURCE_GROUP --kubernetes-version [__K8S_VERSION] --service-principal $SP_APP_ID --client-secret $SP_PASSWORD --generate-ssh-keys -l $LOCATION --node-count 3 --enable-addons monitoring --no-wait
az aks list -o table
az aks get-credentials -n $AKS_CLUSTER_NAME -g $AKS_RESOURCE_GROUP
k get nodes

## Create ACR

az acr create --resource-group $AKS_RESOURCE_GROUP --name $ACR_NAME --sku basic
az role assignment create --assignee $SP_APP_ID --role Contributer --scope $(az acr show --name $ACR_NAME --resource-group $AKS_RESOURCE_GROUP --query "id" --output tsv)

## Create Azure Key Vault

az keyvault create -n $KEYVAULT_NAME -g $AKS_RESOURCE_GROUP -l $LOCATION
az keyvault secret set --vault-name $KEYVAULT_NAME -n Secret1 --value MySuparSecretValue1
az keyvault secret set --vault-name $KEYVAULT_NAME -n Secret2 --value MySuparSecretValue2

## Install AAD Pod Identity CRD (RBAC version)

k create -f https://raw.githubusercontent.com/Azure/aad-pod-identity/master/deploy/infra/deployment-rbac.yaml

## Create Azure Managed Identity

export MANAGED_IDENTITY_NAME=[MANAGED_IDENTITY_NAME]
az identity create -n SampleAppIdentity -g $AKS_RESOURCE_GROUP

## Grant the Managed Identity "Reader" role on the Key Vault

az keyvault assignment create --role "Reader" --assignee $(az identity show --resource-group $RESOURCE_GROUP_NAME --name $MANAGED_IDENTITY_NAME --query "principalId" --output tsv) --scope $(az keyvault show --resource-group $RESOURCE_GROUP_NAME --name $KEYVAULT_NAME --query "id" --output tsv)

## Grant "Get" and "List" permissions on secrets in the Key Vault

az keyvault set-policy -n $KEYVAULT_NAME --secret-permissions get list --spn $(az identity show --resource-group $RESOURCE_GROUP_NAME --name $MANAGED_IDENTITY_NAME --query "clientId" --output tsv)

## Assign the AKS Service Principal to the "Managed Identity Operator" role for the Managed Identity create in the previous steps

az role assignment create --role "Managed Identity Operator" --assignee $SP_APP_ID --scope $(az identity show --resource-group $RESOURCE_GROUP_NAME --name $MANAGED_IDENTITY_NAME --query "id" --output tsv)

## Create the K8s AzureIdentity and AzureIdentityBinding

awk -v MANAGED_IDENTITY_ID=`az identity show --resource-group kubernetes-hackfest --name deletemeidentity --query \"id\" --output tsv` -v CLIENT_ID="az identity show --resource-group $RESOURCE_GROUP_NAME --name $MANAGED_IDENTITY_NAME --query \"clientId\" --output tsv" '{sub(/\$MANAGED_IDENTITY_ID/,MANAGED_IDENTITY_ID);sub(/\$CLIENT_ID/,CLIENT_ID);print}' aad-pod-identity.template.yaml > aad-pod-identity.yaml

k apply -f aad-pod-identity.yaml

awk -v MANAGED_IDENTITY_NAME=`$MANAGED_IDENTITY_NAME` -v SELECTOR="aad-demo-app" '{sub(/\$MANAGED_IDENTITY_NAME/,MANAGED_IDENTITY_NAME);sub(/\$SELECTOR/,SELECTOR);print}' aad-identity-binding.template.yaml > aad-identity-binding.yaml

k apply -f aad-identity-binding.yaml

## OPTIONAL - Test Locally

dotnet user-secrets set "Secret1" "SuparSecretValue1-Development"
dotnet user-secrets set "Secret2" "SuparSecretValue2-Development"

## ACR build the sample app

awk -v ACR_NAME=`$ACR_NAME` '{sub(/\$ACR_NAME/,ACR_NAME);print}' keyvault-demo.template.yaml > keyvault-demo.yaml

az acr build --registry $ACR_NAME --image keyvault-demo:latest .

## Deploy app to K8s

k apply -f ./keyvault-demo.yaml
