# Azure Linux VM Workshop with Managed Identity and REST API Calls

Welcome to this mini-workshop!  
We will create a Linux VM in Azure using the Azure CLI, assign a system-assigned managed identity that has full access to the Azure API on the resource group scope, and then demonstrate how to create a Virtual Network (VNet) using a raw REST API call with `curl` from within that VM.

---

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Set Variables](#set-variables)
3. [Create a Resource Group](#create-a-resource-group)
4. [Create a Linux VM](#create-a-linux-vm)
5. [Assign Managed Identity to the VM](#assign-managed-identity-to-the-vm)
6. [Connect to the VM](#connect-to-the-vm)
7. [Use Managed Identity to Create a VNet via REST API](#use-managed-identity-to-create-a-vnet-via-rest-api)

---

## 1. Prerequisites

- [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli) installed and logged in:
  ```bash
  az login
  ```

- A valid Azure subscription where you can create resources.
- Basic knowledge of bash scripting and Azure services.

## 2. Set Variables

Let’s define a few environment variables to keep our commands concise. Adjust the values to match your desired region and resource names.

```bash
# This retrieves your current subscription ID automatically
export SUBSCRIPTION_ID=$(az account show --query id -o tsv)

# Choose a resource group name
export RESOURCE_GROUP=MyWorkshopRG

# Choose a location (e.g., eastus, westus, etc.)
export LOCATION=eastus

# Choose a VM name
export VM_NAME=MyWorkshopVM

# Choose a VNet name
export VNET_NAME=MyWorkshopVNet
```

## 3. Create a Resource Group

Create a new Resource Group where all our resources will live:

```bash
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION
```

## 4. Create a Linux VM

Create a basic Ubuntu Linux VM.
This command generates SSH keys if you don’t have them already.

```bash
az vm create \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --image Ubuntu2204 \
  --admin-username azureuser \
  --generate-ssh-keys
```

Once complete, Azure will output information including the public IP address. We’ll use that soon to connect to our VM.

## 5. Assign Managed Identity to the VM

Enable a system-assigned managed identity and grant the VM Contributor (or Owner) rights on the resource group.
Contributor is typically sufficient for all resource management tasks at the RG scope.

```bash
az vm identity assign \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --role Contributor \
  --scope /subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP
```

This will automatically create a system-assigned identity and assign the Contributor role.

## 6. Connect to the VM

Retrieve the public IP of the VM:

```bash
VM_PUBLIC_IP=$(az vm show \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --show-details \
  --query publicIps \
  -o tsv)

echo "VM Public IP: $VM_PUBLIC_IP"
```

Use SSH to connect:

```bash
ssh azureuser@$VM_PUBLIC_IP
```

Inside the VM, install jq (for JSON parsing) if it’s not already available:

```bash
sudo apt-get update && sudo apt-get install -y jq
```

## 7. Use Managed Identity to Create a VNet via REST API

Once inside the VM, we can leverage the system-assigned managed identity to call Azure Resource Manager (ARM) APIs directly. The process is:

1. Acquire an access token from the Azure Instance Metadata Service (IMDS).
2. Use curl with the Bearer token to call the ARM REST API.

### 7.1 Get the Access Token

```bash
TOKEN=$(curl -H "Metadata: true" \
  "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2019-08-01&resource=https://management.azure.com" \
  -s | jq -r '.access_token')

echo $TOKEN
```

### 7.2 Create a VNet in the Resource Group using REST

We’ll do a PUT call to create (or update) a Virtual Network in our resource group. Adjust the api-version to the latest if desired:

```bash
curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.Network/virtualNetworks/$VNET_NAME?api-version=2023-02-01" \
  -d '{
        "location": "'"$LOCATION"'",
        "properties": {
          "addressSpace": {
            "addressPrefixes": ["10.0.0.0/16"]
          }
        }
      }'
```

If successful, the API response will include details of your newly created VNet.

## Clean Up Resources

If you want to remove these resources after the workshop:

```bash
az group delete --name $RESOURCE_GROUP --yes --no-wait
```

## Summary

In this workshop, we:

- Created a resource group and a Linux VM with Azure CLI.
- Assigned the VM a system-assigned managed identity with Contributor access.
- Retrieved an OAuth2 token from Azure IMDS inside the VM.
- Used curl to demonstrate calling Azure Resource Manager directly to create a Virtual Network.

This pattern lets you manage Azure resources securely and programmatically from a VM without storing credentials in code.

Happy exploring!
