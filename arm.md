**Note before beginning**: Bicep is a Domain Specific Language (DSL) that acts as a *simpler, higher-level abstraction* over Azure Resource Manager (ARM) templates, which are the underlying JSON for deploying Azure resources. Bicep makes development easier by reducing complexity and improving readability, but ARM remains the core deployment engine. Think of Bicep as a "translator" that converts to ARM JSON behind the scenes, providing a better authoring experience for Azure infrastructure-as-code. 

This demo will be a brief overview of ARM to help us understand what is happening in the background.

### JSON Template Format
In its simplest structure, a template has the following elements:
```
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "languageVersion": "",
  "contentVersion": "",
  "apiProfile": "",
  "definitions": { },
  "parameters": { },
  "variables": { },
  "functions": [ ],
  "resources": [ ], /* or "resources": { } with languageVersion 2.0 */
  "outputs": { }
}
```
**Required Elements**
- **$schema** describes the version of the template language.
- **contentVersion** Version of the template (such as 1.0.0.0). You can provide any value for this element. Use this value to document significant changes in your template. 
- **[resources](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/syntax#resources)** Resource types that are deployed or updated in a resource group or subscription.

**Optional Elements**
- **languageVersion** Language version of the template.
- **apiProfile** An API version that serves as a collection of API versions for resource types. Use this value to avoid having to specify API versions for each resource in the template. 
- **[definitions](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/syntax#definitions)** Schemas that are used to validate array and object values. Definitions are only supported in languageVersion 2.0.
- **[parameters](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/syntax#parameters)**	Values that are provided when deployment is executed to customize resource deployment.
- **[variables](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/syntax#variables)** Values that are used as JSON fragments in the template to simplify template language expressions
- **[functions](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/syntax#functions)** User-defined functions that are available within the template.
- **[outputs](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/syntax#outputs)** Values that are returned after deployment.

Note: JSON does not support comments.

# Demonstration

Create a folder named `arm-demo` and a file within named `template.json`.

This file will be an ARM template that defines repeatable IaC to deploy an *Azure Storage account and a private blob container inside.*

**The storage account resource will specify the following:**
1. **type**: Specifies the type of Azure resource, which in this case is a storage account under the Microsoft.Storage namespace.
2. **apiVersion**: Indicates the version of the Azure API being used, which is 2019-06-01 in this case.
3. **name**: Defines the name of the storage account, using a parameter called 'storageAccountName'.
4. **location**: Specifies the geographical location where the storage account will be deployed, utilizing the value provided by the parameter 'location'.
5. **sku**: Describes the SKU (Stock Keeping Unit) configuration for the storage account, indicating that it is of type Standard_LRS (Locally Redundant Storage) and belongs to the Standard tier.
6. **kind**: Specifies the kind of storage account, which is StorageV2, indicating it is a general-purpose v2 storage account.
7. **properties**: Contains additional properties specific to the storage account, such as the access tier, which is set to Hot, indicating that data is stored in a manner that allows for frequent access.

**The blob container resource specifies the following:**
1. **type**: Specifies the type of Azure resource, which is a blob container within a blob service in a storage account under the Microsoft.Storage namespace.
2. **apiVersion**: Indicates the version of the Azure API being used, which is 2019-06-01 in this case.
3. **name**: Defines the name of the blob container, concatenating values from parameters 'storageAccountName' and 'containerName' to form the full container name.
4. **dependsOn**: Specifies the dependency of this resource on another resource, specifically on the storage account defined by the 'storageAccountName' parameter.
5. **properties**: Contains additional properties specific to the blob container, such as the public access level, which is set to None, indicating that no anonymous access is allowed to the contents of the container.



Within the file, add the following:
```
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "location": {
            "type": "string",
            "defaultValue": "[resourceGroup().location]",
            "metadata": {
                "description": "Location for all resources."
            }
        },
        "storageAccountName": {
            "type": "string",
            "metadata": {
                "description": "Name of the storage account."
            }
        },
        "containerName": {
            "type": "string",
            "defaultValue": "mycontainer",
            "metadata": {
                "description": "Name of the container."
            }
        }
    },
    "resources": [
        {
            "type": "Microsoft.Storage/storageAccounts",
            "apiVersion": "2019-06-01",
            "name": "[parameters('storageAccountName')]",
            "location": "[parameters('location')]",
            "sku": {
                "name": "Standard_LRS",
                "tier": "Standard"
            },
            "kind": "StorageV2",
            "properties": {
                "accessTier": "Hot"
            }
        },
        {
            "type": "Microsoft.Storage/storageAccounts/blobServices/containers",
            "apiVersion": "2019-06-01",
            "name": "[concat(parameters('storageAccountName'), '/default/', parameters('containerName'))]",
            "dependsOn": [
                "[resourceId('Microsoft.Storage/storageAccounts', parameters('storageAccountName'))]"
            ],
            "properties": {
                "publicAccess": "None"
            }
        }
    ]
}
```
- parameters: Inputs you provide at deployment time (or use defaults):
- location: Where resources will be created; defaults to the resource group’s region (resourceGroup().location).
- storageAccountName: Required input—this becomes the name of the storage account (must be globally unique, lowercase, 3–24 chars).
- containerName: Blob container name; defaults to "mycontainer".

**The deployment will contain:**
- Storage account
  - Type: Microsoft.Storage/storageAccounts
  - Name: Uses your storageAccountName parameter.
  - SKU: Standard_LRS = standard performance with Locally Redundant Storage (3 copies in one datacenter region).
  - Kind: StorageV2 = General-purpose v2 storage account (recommended modern type).
  - Access tier: "Hot" = optimized for frequently accessed blob data (applies to blob storage).
- Blob container inside the storage account
  - Type: Microsoft.Storage/storageAccounts/blobServices/containers
  - Name: Built by concatenating: storageAccountName + "/default/" + containerName<br>
  The "default" part refers to the storage account’s default blob service.
  - dependsOn: Ensures Azure creates the storage account first, then creates the container.
  - publicAccess: "None" means the container is not publicly accessible (no anonymous blob/container access).

  1. **Login** to Azure `az login`
  2. Create or use an existing Resource Group<br>
  To use an existing group, set variables in PowerShell<br>
  Confirm groups `az group list`.
```
$RG="RESOURCEGROUPNAME"
$LOC="eastus"
$SA="NEWSANAME"  # must be globally unique, 3-24 lowercase letters/numbers
$TEMPLATE_FILE="template.json"
```
If no group exists, create one and set the variables above.<br>
```
az group create --location eastus --resource-group <name>
```

**Validate** the deployment schema:
```
az deployment group validate `
  --resource-group $RG `
  --template-file $TEMPLATE_FILE `
  --parameters storageAccountName=$SA
```

**What-if** shows changes before deploying:
```
az deployment group what-if `
  --resource-group $RG `
  --template-file $TEMPLATE_FILE `
  --parameters storageAccountName=$SA
```

**Deploy**
```
az deployment group create `
  --resource-group $RG `
  --name ("deploy-storage-" + (Get-Date -Format "yyyyMMddHHmmss")) `
  --template-file $TEMPLATE_FILE `
  --parameters storageAccountName=$SA
```

### Clean up Demo
Delete the storage account (deletes containers)
```
az storage account delete `
  --name $SA `
  --resource-group $RG `
  --yes
```

You can also delete the resource group `az group delete --name $RG --yes --no-wait`.

Confirm resources have deleted from the Azure Portal.