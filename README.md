# Azure Linux VM Workshop with Managed Identity and REST API Calls

Welcome to this mini-workshop!

We will create a Linux VM in Azure using the Azure CLI, assign a system-assigned managed identity that has full access to the Azure API on the resource group scope, and then demonstrate how to create and manage resources using the managed identity and the REST API.

Managed identities in Azure come in two flavors: **system-assigned and user-assigned**. Both types provide your Azure resources (like VMs or App Services) with an automatically managed identity in Azure Active Directory (Azure AD).

#### System-Assigned Managed Identity

- **Tied to a Single Resource**: The identity is enabled directly on an Azure resource (e.g., a VM).
- **Lifecycle Coupled**: When you enable the system-assigned identity, Azure automatically creates an identity in Azure AD for that resource. If you delete the resource, the identity is automatically deleted as well.
- **One-to-One Relationship**: Each resource that uses a system-assigned identity gets its own identity. You can’t share it with other resources.

**Use Case**: Great for simple scenarios where you only need credentials for a single resource, and you want that identity lifecycle to match the resource.

#### User-Assigned Managed Identity

- **Standalone Resource**: You create this identity as a separate Azure resource in a resource group or subscription.
- **Lifecycle Independent**: Once created, it exists independently of any particular resource. Deleting a VM or an App Service using this identity will not delete the identity itself.
- **Reusability**: You can attach the same user-assigned identity to multiple Azure resources. That means they all share the same Azure AD identity (same principal).

**Use Case**: Useful when you want a single identity for multiple resources or when you need an identity to persist even if the resources are rebuilt. It’s also helpful for consistent access control or auditing across multiple resources.

---

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Set Variables](#set-variables)
3. [Create a Resource Group](#create-a-resource-group)
4. [Create a Linux VM](#create-a-linux-vm)
5. [Assign Managed Identity to the VM](#assign-managed-identity-to-the-vm)
6. [Connect to the VM](#connect-to-the-vm)
7. [Use Managed Identity to Create a VNet via REST API](#use-managed-identity-to-create-a-vnet-via-rest-api)
   - [Get the Access Token](#7.1-get-the-access-token)
   - [Query the Instance Metadata Service for instance details](#7.2-query-the-instance-metadata-service-for-instance-details)
   - [Create a VNet in the Resource Group using REST](#7.3-create-a-vnet-in-the-resource-group-using-rest)
8. [Clean Up Resources](#clean-up-resources)
9. [Summary](#summary)
10. [Notes](#notes)
    - [Show the Managed Identity of Your VM](#1-show-the-managed-identity-of-your-vm)
    - [Capture the VM's principal ID in a variable for convenience](#2-capture-the-vms-principal-id-in-a-variable-for-convenience)
    - [List role assignments for that principal ID](#3-list-role-assignments-for-that-principal-id)

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
### 7.2 Query the Instance Metadata Service for instance details
```
INSTANCE_JSON=$(curl -s -H "Metadata:true" "http://169.254.169.254/metadata/instance?api-version=2021-02-01")
export VM_RESOURCE_GROUP=$(echo "$INSTANCE_JSON" | jq -r '.compute.resourceGroupName')
export VM_SUBSCRIPTION_ID=$(echo "$INSTANCE_JSON" | jq -r '.compute.subscriptionId')
export VM_LOCATION=$(echo "$INSTANCE_JSON" | jq -r '.compute.location')

echo "Resource Group: $VM_RESOURCE_GROUP"
echo "Subscription ID: $VM_SUBSCRIPTION_ID"
echo "Location: $VM_LOCATION"
```
### 7.3 Create a VNet in the Resource Group using REST as an example

We’ll do a PUT call to create (or update) a Virtual Network in our resource group. Adjust the api-version to the latest if desired:

```bash
export VNET_NAME="vnet_demo"

curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "https://management.azure.com/subscriptions/$VM_SUBSCRIPTION_ID/resourceGroups/$VM_RESOURCE_GROUP/providers/Microsoft.Network/virtualNetworks/$VNET_NAME?api-version=2023-02-01" \
  -d '{
        "location": "'"$VM_LOCATION"'",
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

## Notes
### 1. Show the Managed Identity of Your VM
```
az vm identity show \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME
```
### 2. Capture the VM's principal ID in a variable for convenience
```
export PRINCIPAL_ID=$(az vm identity show \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --query principalId -o tsv)
```
### 3. List role assignments for that principal ID
```
az role assignment list \
  --assignee $PRINCIPAL_ID \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP" \
  --output table
```
