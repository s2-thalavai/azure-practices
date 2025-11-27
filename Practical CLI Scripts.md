# 1. Azure CLI Script — Bulk Tag VMs by Department

assign a tag like:

```
Department = Finance
Department = HR
Department = IT
```
can apply these via Portal, CLI, PowerShell, ARM/Bicep, Terraform, or Azure Policy.

## Option 1 — Apply the same department tag to all VMs

```bash
# Variables
RESOURCE_GROUP="RG1"
DEPARTMENT="Finance"

# Bulk tag all VMs in RG1
for vm in $(az vm list -g $RESOURCE_GROUP --query "[].name" -o tsv); do
  echo "Tagging VM: $vm with Department=$DEPARTMENT"
  az resource tag \
    --ids $(az vm show -g $RESOURCE_GROUP -n $vm --query id -o tsv) \
    --tags Department=$DEPARTMENT
done
```

## Option 2 — Assign different department tags to each VM

Use a mapping file (CSV or bash associative array).

Example using a Bash associative array

```bash
# Resource group
RESOURCE_GROUP="RG1"

# VM → Department mapping
declare -A deptMap
deptMap=(
  ["vm-fin-01"]="Finance"
  ["vm-fin-02"]="Finance"
  ["vm-hr-01"]="HR"
  ["vm-it-01"]="IT"
  ["vm-it-02"]="IT"
)

for vm in "${!deptMap[@]}"; do
  echo "Tagging $vm with Department=${deptMap[$vm]}"
  az resource tag \
    --ids $(az vm show -g $RESOURCE_GROUP -n $vm --query id -o tsv) \
    --tags Department="${deptMap[$vm]}"
done
```

## Option 3 — Bulk-tag using CSV input

### vm-tags.csv

```csv
vm-name,department
vm-fin-01,Finance
vm-hr-01,HR
vm-it-01,IT
```

### CLI script

``` bash
RESOURCE_GROUP="RG1"

while IFS=',' read -r VM DEPT; do
    # Skip header
    if [[ $VM == "vm-name" ]]; then continue; fi

    echo "Tagging $VM with Department=$DEPT"
    az resource tag \
      --ids $(az vm show -g $RESOURCE_GROUP -n $VM --query id -o tsv) \
      --tags Department=$DEPT
done < vm-tags.csv
```

## **Azure Policy** that enforces a **Department** tag on all Virtual Machines.  

1️. **Audit-only** (just check)  
2️. **Deny** if missing  
3️. **Auto-add default tag value** (recommended for governance)

## **1. Enforce: Deny VM creation/edit if Department tag is missing**

```json
{
  "properties": {
    "displayName": "Require Department tag on all Virtual Machines",
    "mode": "Indexed",
    "parameters": {
      "tagName": {
        "type": "String",
        "defaultValue": "Department",
        "metadata": {
          "displayName": "Tag Name",
          "description": "The tag name to enforce"
        }
      }
    },
    "policyRule": {
      "if": {
        "allOf": [
          {
            "field": "type",
            "equals": "Microsoft.Compute/virtualMachines"
          },
          {
            "field": "[concat('tags[', parameters('tagName'), ']')]",
            "exists": "false"
          }
        ]
      },
      "then": {
        "effect": "deny"
      }
    }
  }
}

```

# **2. Audit: Show VMs missing Department tag**

```json
{
  "properties": {
    "displayName": "Audit VMs missing Department tag",
    "mode": "Indexed",
    "parameters": {
      "tagName": {
        "type": "String",
        "defaultValue": "Department"
      }
    },
    "policyRule": {
      "if": {
        "allOf": [
          {
            "field": "type",
            "equals": "Microsoft.Compute/virtualMachines"
          },
          {
            "field": "[concat('tags[', parameters('tagName'), ']')]",
            "exists": "false"
          }
        ]
      },
      "then": {
        "effect": "audit"
      }
    }
  }
}

```

# **3. Auto-add Missing Tag with a Default Value (Modify Policy)**

This will **automatically add** a Department tag if the user forgets it.

```json
{
  "properties": {
    "displayName": "Add default Department tag to VMs",
    "mode": "Indexed",
    "parameters": {
      "tagName": {
        "type": "String",
        "defaultValue": "Department"
      },
      "tagValue": {
        "type": "String",
        "defaultValue": "Unknown",
        "metadata": {
          "description": "Default department value to assign"
        }
      }
    },
    "policyRule": {
      "if": {
        "allOf": [
          {
            "field": "type",
            "equals": "Microsoft.Compute/virtualMachines"
          },
          {
            "field": "[concat('tags[', parameters('tagName'), ']')]",
            "exists": "false"
          }
        ]
      },
      "then": {
        "effect": "modify",
        "details": {
          "roleDefinitionIds": [
            "/providers/microsoft.authorization/roleDefinitions/fd72f4d3-6a4c-4b90-a206-75c78e3b39d4"
          ],
          "operations": [
            {
              "operation": "addOrReplace",
              "field": "[concat('tags[', parameters('tagName'), ']')]",
              "value": "[parameters('tagValue')]"
            }
          ]
        }
      }
    }
  }
}
```

> Note: The `roleDefinitionIds` includes **Tag Contributor**, required for Modify policies.

## Example (assign policy to RG1):

``` bash
az policy assignment create \
  --name require-department-tag \
  --policy require-department-tag.json \
  --scope /subscriptions/<SUB-ID>/resourceGroups/RG1
```

## Azure Policy: Require Multiple Tags on All VMs (Deny if missing)

``` json
{
  "properties": {
    "displayName": "Require multiple tags on all Virtual Machines",
    "description": "Ensures that all VMs have the required tags: Department, CostCenter, Owner, Environment.",
    "mode": "Indexed",
    "parameters": {
      "requiredTags": {
        "type": "Array",
        "metadata": {
          "displayName": "Required Tags",
          "description": "List of tag names that must exist on a resource."
        },
        "defaultValue": [
          "Department",
          "CostCenter",
          "Owner",
          "Environment"
        ]
      }
    },
    "policyRule": {
      "if": {
        "allOf": [
          {
            "field": "type",
            "equals": "Microsoft.Compute/virtualMachines"
          },
          {
            "anyOf": [
              {
                "value": "[length(parameters('requiredTags'))]",
                "less": 1
              },
              {
                "anyOf": [
                  {
                    "not": {
                      "field": "[concat('tags[', parameters('requiredTags')[0], ']')]",
                      "exists": "true"
                    }
                  },
                  {
                    "not": {
                      "field": "[concat('tags[', parameters('requiredTags')[1], ']')]",
                      "exists": "true"
                    }
                  },
                  {
                    "not": {
                      "field": "[concat('tags[', parameters('requiredTags')[2], ']')]",
                      "exists": "true"
                    }
                  },
                  {
                    "not": {
                      "field": "[concat('tags[', parameters('requiredTags')[3], ']')]",
                      "exists": "true"
                    }
                  }
                ]
              }
            ]
          }
        ]
      },
      "then": {
        "effect": "deny"
      }
    }
  }
}

```

### 1. Why the policy lists tags individually?

Azure Policy **does not allow looping** (no foreach).  
So required tags must be expressed as explicit conditions.

-------------

# 2. Create an Azure AD Conditional Access Policy

Goal: Require Global Administrators to use MFA and an Azure AD–joined / compliant device when accessing Azure AD from untrusted locations.

### **1. Go to Azure AD (Entra ID)**

-   Open **Microsoft Entra admin center**
    
-   Navigate to:  
    **Protection → Conditional Access → Create New Policy**
    

# **2. Name the policy**

Example:  
**"GA – Require MFA + Compliant Device from Untrusted Locations"**

# **3. Assignments**

### ** Users**

-   **Include:**
    
    -   _Directory roles → Global Administrator_
        
-   **Exclude (optional but recommended):**
    
    -   Break-glass emergency account (if your org uses one)
        

### ** Cloud Apps**

-   Select:  
    **Microsoft Azure Management**  
    (covers Azure Portal, ARM, CLI, PowerShell)
    

You can also choose **All cloud apps** if required.

### ** Conditions → Locations**

-   **Include:**
    
    -   **All locations**
        
-   **Exclude:**
    
    -   **Trusted locations** (Named locations → "Trusted Network" or IP ranges your org trusts)
        

This ensures the policy applies only from _untrusted_ locations.

# **4. Access Controls**

### ** Grant access → Require:**

-   ** Multi-factor authentication**
    
-   ** Require device to be marked as compliant** _(recommended)_  
    or
    
-   ** Require Hybrid Azure AD joined device / Require Azure AD joined device**
    

This satisfies:

-   MFA required
    
-   Device trust requirement
    

**Important:** Selecting _Require all selected controls_ enforces both.

# **5. Enable Policy**

-   Set **Enable policy → On**
    
-   Save
    

# **Policy Summary**

Your policy now enforces:

-   Scope: **Global Administrators**
    
-   Condition: **If accessing from untrusted locations**
    
-   Controls enforced:
    
    -   MFA
        
    -   Azure AD–joined or compliant device
        
-   App context: Azure AD / Azure management

--------

# Use an ARM template to deploy VMs while ensuring the admin password is NOT stored in plain text.

To achieve this, Azure requires TWO things:

1. A secure place to store the password → Azure Key Vault
2. A way for the ARM template/VM to read the secret → An access policy

| **Answer Area**                                                 | **Correct Option**     |
| --------------------------------------------------------------- | ---------------------- |
| **Component to store the password securely**                    | **An Azure Key Vault** |
| **Component to allow the ARM template/VM to read the password** | **An access policy**   |

### **Azure Key Vault**

-   Stores secrets (like passwords) securely
    
-   Prevents passwords from appearing in ARM templates
    
-   ARM templates reference Key Vault secrets using `"reference"` objects
    

### **Access Policy**

-   Grants the ARM deployment identity permission to **Get** secrets from Key Vault
    
-   Without this, the ARM template cannot retrieve the password securely

| Option                           | Why it’s wrong                                      |
| -------------------------------- | --------------------------------------------------- |
| **Azure Storage account**        | Can't store secrets securely for ARM templates      |
| **Azure AD Identity Protection** | Used for risky sign-in analysis, not secret storage |
| **Azure policy**                 | Enforces compliance; doesn’t store secrets          |
| **Backup policy**                | Manages backup schedules; irrelevant                |


**ARM template snippet** you need to securely reference an admin password stored in **Azure Key Vault**, using a **secureString parameter** and **Key Vault secret reference**.

This is the _correct and secure_ way to avoid storing passwords in plain text.


**production-ready ARM template** that deploys:

### Azure Key Vault

### A Secret stored securely in Key Vault

### Access Policy allowing the VM's Managed Identity to read the secret

### A VM (Windows Server) that uses that secret as its **admin password**

This is a **single, combined ARM template**—no external resources required.

----------

# **COMPLETE ARM TEMPLATE: VM + Key Vault + Secret + Access Policy**

> **Note:**  
> The VM uses a **System-Assigned Managed Identity** to retrieve the password from Key Vault during deployment.

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "vmName": {
      "type": "string",
      "defaultValue": "myVM"
    },
    "adminUsername": {
      "type": "string",
      "defaultValue": "azureuser"
    },
    "adminPasswordValue": {
      "type": "secureString",
      "metadata": {
        "description": "Password stored in Key Vault secret"
      }
    },
    "keyVaultName": {
      "type": "string",
      "defaultValue": "myKeyVault12345"
    },
    "secretName": {
      "type": "string",
      "defaultValue": "adminPassword"
    },
    "location": {
      "type": "string",
      "defaultValue": "[resourceGroup().location]"
    }
  },
  "variables": {
    "nicName": "[concat(parameters('vmName'), '-nic')]",
    "vnetName": "[concat(parameters('vmName'), '-vnet')]",
    "subnetName": "default",
    "publicIPName": "[concat(parameters('vmName'), '-pip')]"
  },
  "resources": [
    {
      "type": "Microsoft.KeyVault/vaults",
      "apiVersion": "2022-07-01",
      "name": "[parameters('keyVaultName')]",
      "location": "[parameters('location')]",
      "properties": {
        "tenantId": "[subscription().tenantId]",
        "sku": {
          "name": "standard",
          "family": "A"
        },
        "accessPolicies": []
      }
    },

    {
      "type": "Microsoft.KeyVault/vaults/secrets",
      "apiVersion": "2022-07-01",
      "name": "[format('{0}/{1}', parameters('keyVaultName'), parameters('secretName'))]",
      "dependsOn": [
        "[resourceId('Microsoft.KeyVault/vaults', parameters('keyVaultName'))]"
      ],
      "properties": {
        "value": "[parameters('adminPasswordValue')]"
      }
    },

    {
      "type": "Microsoft.Network/publicIPAddresses",
      "apiVersion": "2021-05-01",
      "name": "[variables('publicIPName')]",
      "location": "[parameters('location')]",
      "properties": {
        "publicIPAllocationMethod": "Dynamic"
      }
    },

    {
      "type": "Microsoft.Network/virtualNetworks",
      "apiVersion": "2021-05-01",
      "name": "[variables('vnetName')]",
      "location": "[parameters('location')]",
      "properties": {
        "addressSpace": {
          "addressPrefixes": [
            "10.0.0.0/16"
          ]
        },
        "subnets": [
          {
            "name": "[variables('subnetName')]",
            "properties": {
              "addressPrefix": "10.0.1.0/24"
            }
          }
        ]
      }
    },

    {
      "type": "Microsoft.Network/networkInterfaces",
      "apiVersion": "2021-05-01",
      "name": "[variables('nicName')]",
      "location": "[parameters('location')]",
      "dependsOn": [
        "[resourceId('Microsoft.Network/publicIPAddresses', variables('publicIPName'))]",
        "[resourceId('Microsoft.Network/virtualNetworks', variables('vnetName'))]"
      ],
      "properties": {
        "ipConfigurations": [
          {
            "name": "ipconfig1",
            "properties": {
              "subnet": {
                "id": "[resourceId('Microsoft.Network/virtualNetworks/subnets', variables('vnetName'), variables('subnetName'))]"
              },
              "publicIPAddress": {
                "id": "[resourceId('Microsoft.Network/publicIPAddresses', variables('publicIPName'))]"
              }
            }
          }
        ]
      }
    },

    {
      "type": "Microsoft.Compute/virtualMachines",
      "apiVersion": "2021-07-01",
      "name": "[parameters('vmName')]",
      "location": "[parameters('location')]",
      "identity": {
        "type": "SystemAssigned"
      },
      "dependsOn": [
        "[resourceId('Microsoft.Network/networkInterfaces', variables('nicName'))]"
      ],
      "properties": {
        "hardwareProfile": {
          "vmSize": "Standard_DS1_v2"
        },
        "osProfile": {
          "computerName": "[parameters('vmName')]",
          "adminUsername": "[parameters('adminUsername')]",
          "adminPassword": "[reference(resourceId('Microsoft.KeyVault/vaults', parameters('keyVaultName')), '2022-07-01', 'Full').secrets[parameters('secretName')].value]"
        },
        "storageProfile": {
          "imageReference": {
            "publisher": "MicrosoftWindowsServer",
            "offer": "WindowsServer",
            "sku": "2019-Datacenter",
            "version": "latest"
          },
          "osDisk": {
            "createOption": "FromImage"
          }
        },
        "networkProfile": {
          "networkInterfaces": [
            {
              "id": "[resourceId('Microsoft.Network/networkInterfaces', variables('nicName'))]"
            }
          ]
        }
      }
    },

    {
      "type": "Microsoft.KeyVault/vaults/accessPolicies",
      "apiVersion": "2022-07-01",
      "name": "[format('{0}/add', parameters('keyVaultName'))]",
      "dependsOn": [
        "[resourceId('Microsoft.Compute/virtualMachines', parameters('vmName'))]"
      ],
      "properties": {
        "accessPolicies": [
          {
            "tenantId": "[subscription().tenantId]",
            "objectId": "[reference(resourceId('Microsoft.Compute/virtualMachines', parameters('vmName')), '2021-07-01', 'Full').identity.principalId]",
            "permissions": {
              "secrets": [
                "get",
                "list"
              ]
            }
          }
        ]
      }
    }
  ],
  "outputs": {
    "vmId": {
      "type": "string",
      "value": "[resourceId('Microsoft.Compute/virtualMachines', parameters('vmName'))]"
    }
  }
}

```

| Component                    | Purpose                                     |
| ---------------------------- | ------------------------------------------- |
| **Key Vault**                | Safely stores the admin password            |
| **Secret**                   | Holds the encrypted password                |
| **Access Policy**            | Grants the VM permission to read the secret |
| **VM with Managed Identity** | Retrieves password securely                 |
| **No plain text password**   | Fully secure deployment                     |

------



# **clean, modern, production-ready Bicep template** that deploys:

 **Azure Key Vault**  
 **A secret inside Key Vault**  
 **Access policy granting the VM’s Managed Identity permission to read the secret**  
 **A Windows VM that securely retrieves its admin password from Key Vault**  
 **No plaintext passwords**

This is the **combined end-to-end deployment**.

# **FULL BICEP TEMPLATE — VM + Key Vault + Secret + Access Policy**

```bicep
param vmName string = 'myVM'
param adminUsername string = 'azureuser'
param adminPasswordValue securestring
param keyVaultName string = 'myKeyVault12345'
param secretName string = 'adminPassword'
param location string = resourceGroup().location

// Networking variables
var vnetName = '${vmName}-vnet'
var subnetName = 'default'
var nicName = '${vmName}-nic'
var pipName = '${vmName}-pip'

// ---------------------------
// Key Vault
// ---------------------------
resource keyVault 'Microsoft.KeyVault/vaults@2022-07-01' = {
  name: keyVaultName
  location: location
  properties: {
    tenantId: subscription().tenantId
    sku: {
      name: 'standard'
      family: 'A'
    }
    accessPolicies: []
  }
}

// ---------------------------
// Key Vault Secret
// ---------------------------
resource kvSecret 'Microsoft.KeyVault/vaults/secrets@2022-07-01' = {
  name: '${keyVaultName}/${secretName}'
  dependsOn: [
    keyVault
  ]
  properties: {
    value: adminPasswordValue
  }
}

// ---------------------------
// Public IP
// ---------------------------
resource publicIP 'Microsoft.Network/publicIPAddresses@2021-05-01' = {
  name: pipName
  location: location
  properties: {
    publicIPAllocationMethod: 'Dynamic'
  }
}

// ---------------------------
// VNet + Subnet
// ---------------------------
resource vnet 'Microsoft.Network/virtualNetworks@2021-05-01' = {
  name: vnetName
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [
        '10.0.0.0/16'
      ]
    }
    subnets: [
      {
        name: subnetName
        properties: {
          addressPrefix: '10.0.1.0/24'
        }
      }
    ]
  }
}

// ---------------------------
// NIC
// ---------------------------
resource nic 'Microsoft.Network/networkInterfaces@2021-05-01' = {
  name: nicName
  location: location
  dependsOn: [
    publicIP
    vnet
  ]
  properties: {
    ipConfigurations: [
      {
        name: 'ipconfig1'
        properties: {
          subnet: {
            id: vnet.properties.subnets[0].id
          }
          publicIPAddress: {
            id: publicIP.id
          }
        }
      }
    ]
  }
}

// ---------------------------
// Virtual Machine with Managed Identity
// ---------------------------
resource vm 'Microsoft.Compute/virtualMachines@2021-07-01' = {
  name: vmName
  location: location
  identity: {
    type: 'SystemAssigned'
  }
  dependsOn: [
    nic
  ]
  properties: {
    hardwareProfile: {
      vmSize: 'Standard_DS1_v2'
    }
    osProfile: {
      computerName: vmName
      adminUsername: adminUsername
      adminPassword: reference(keyVault.id, '2022-07-01', 'Full').secrets[secretName].value
    }
    storageProfile: {
      imageReference: {
        publisher: 'MicrosoftWindowsServer'
        offer: 'WindowsServer'
        sku: '2019-Datacenter'
        version: 'latest'
      }
      osDisk: {
        createOption: 'FromImage'
      }
    }
    networkProfile: {
      networkInterfaces: [
        {
          id: nic.id
        }
      ]
    }
  }
}

// ---------------------------
// Grant Managed Identity access to Key Vault Secret
// ---------------------------
resource kvAccessPolicy 'Microsoft.KeyVault/vaults/accessPolicies@2022-07-01' = {
  name: '${keyVaultName}/add'
  dependsOn: [
    vm
    kvSecret
  ]
  properties: {
    accessPolicies: [
      {
        tenantId: subscription().tenantId
        objectId: vm.identity.principalId
        permissions: {
          secrets: [
            'get'
            'list'
          ]
        }
      }
    ]
  }
}

// ---------------------------
// Outputs
// ---------------------------
output vmId string = vm.id
output keyVaultUri string = keyVault.properties.vaultUri
output adminSecretName string = secretName

```

| Component               | Purpose                                |
| ----------------------- | -------------------------------------- |
| **Key Vault**           | Secure store for admin password        |
| **Secret**              | Encrypted password stored in KV        |
| **VM Managed Identity** | Authenticates to Key Vault             |
| **Access policy**       | Grants MI secret-get permissions       |
| **VM**                  | Pulls password securely from Key Vault |
| **Networking**          | VNet, Subnet, NIC, Public IP           |

-------

# workflow to create a Managed Image from the uploaded VHD &
# How to deploy multiple VMs from the custom image

You have:

-   A **generalized on-premises VM image** (Windows or Linux)
    
-   Likely exported as a **.vhd**
    
-   You need to **upload the VHD to Azure** so it can be used to create Azure VMs
    

The correct PowerShell cmdlet for uploading a **generalized VHD** from on-premises to Azure storage is:

### **`Add-AzVhd`**

This cmdlet:

-   Uploads a local VHD file to an Azure storage account
    
-   Converts it if necessary
    
-   Prepares the VHD to be used as an Azure **OS image**
    

Example usage:

```
Add-AzVhd -LocalFilePath "C:\Images\myvm.vhd" `
          -ResourceGroupName "RG1" `
          -Destination "https://storageaccount.blob.core.windows.net/vhds/myvm.vhd"
``` 

**Notes:**

-   The destination **must be a page blob**
    
-   The disk must be **VHD** format (not VHDX)
  
Once uploaded, you use the VHD to create an **Azure Managed Image** or a VM directly.

Add-AzVM                Deploys a VM


## Create a Managed Image from the uploaded VHD

Use the New-AzImageConfig and New-AzImage cmdlets.

``` powershell
# Set variables
$rg = "MyResourceGroup"
$imageName = "MyCustomImage"
$location = "eastus"
$vhdUri = "https://<storageaccount>.blob.core.windows.net/vhds/MyCustomVM.vhd"

# Create the image config
$imageConfig = New-AzImageConfig `
  -Location $location

# Add OS disk from VHD
$imageConfig = Set-AzImageOsDisk `
  -Image $imageConfig `
  -OsState Generalized `
  -OsType Windows `
  -BlobUri $vhdUri

# Create the managed image
New-AzImage `
  -ImageName $imageName `
  -ResourceGroupName $rg `
  -Image $imageConfig
```

This creates a Managed Image in Azure.


# **Create a VM from the Managed Image**

``` powershell
New-AzVM `
  -ResourceGroupName $rg `
  -Location $location `
  -Name "MyVM01" `
  -ImageName $imageName `
  -VirtualNetworkName "MyVnet" `
  -SubnetName "default" `
  -SecurityGroupName "MyNSG" `
  -PublicIpAddressName "MyPublicIP"

``` 

The VM will now:

-   Use your **custom image**
    
-   Include all baked-in software configurations
    
-   Start in a generalized state (OOBE)

--------

# pipeline (Azure DevOps/GitHub Actions) to automate image creation

Both pipelines:

1. Build an image from a base OS  
2. Run customization scripts  
3. Output a **Managed Image** or **Shared Image Gallery (SIG)** image  
4. Can upload your own VHD or use Marketplace images  
5. Trigger on demand or on schedule


# **Azure DevOps Pipeline (YAML)**

## **azure-pipelines.yml**

```yaml
trigger: none

pool:
  vmImage: 'ubuntu-latest'

variables:
  location: 'eastus'
  resourceGroup: 'AIB-RG'
  imageTemplateName: 'MyAIBTemplate'
  sigName: 'MyImageGallery'
  sigImageName: 'MyCustomImage'
  sigImageVersion: '1.0.0'

steps:

- task: AzureCLI@2
  displayName: "Create RG if missing"
  inputs:
    azureSubscription: "My-Service-Connection"
    scriptType: bash
    scriptLocation: inlineScript: |
      az group create -n $resourceGroup -l $location

- task: AzureCLI@2
  displayName: "Deploy Azure Image Builder Template"
  inputs:
    azureSubscription: "My-Service-Connection"
    scriptType: bash
    scriptLocation: inlineScript: |
      az deployment group create \
        --resource-group $resourceGroup \
        --template-file aibTemplate.json \
        --parameters \
            imageTemplateName=$imageTemplateName \
            location=$location \
            sigName=$sigName \
            sigImageName=$sigImageName \
            sigImageVersion=$sigImageVersion

- task: AzureCLI@2
  displayName: "Start Image Build"
  inputs:
    azureSubscription: "My-Service-Connection"
    scriptType: bash
    scriptLocation: inlineScript: |
      az image builder run \
        --name $imageTemplateName \
        --resource-group $resourceGroup
```

-------------------

# **GitHub Actions Workflow**

## **.github/workflows/build-image.yml**

``` yaml
name: Build Azure Image

on:
  workflow_dispatch:

jobs:
  build-image:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3

    - name: Azure Login
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}

    - name: Create/Update AIB Template
      run: |
        az group create -n AIB-RG -l eastus

        az deployment group create \
          --resource-group AIB-RG \
          --template-file aibTemplate.json \
          --parameters \
            imageTemplateName=MyAIBTemplate \
            location=eastus \
            sigName=MyImageGallery \
            sigImageName=MyCustomImage \
            sigImageVersion=1.0.0

    - name: Run AIB Build
      run: |
        az image builder run \
          --resource-group AIB-RG \
          --name MyAIBTemplate
```


# **A fully automated image creation pipeline**

Your pipeline now:

-   Builds from **Windows/Linux marketplace image**
    
-   Runs **PowerShell or Bash customization scripts**
    
-   Outputs a **Managed Image** or **Shared Image Gallery version**
    
-   Allows consistent VM deployment at scale
    
-   Is fully reusable and automatic

------------------


# **replicating an on-premises Hyper-V VM (VM1) to Azure using Azure Site Recovery (ASR).**

When enabling ASR for **Hyper-V (without SCVMM)**, you **must create** the following objects:

## **1. Azure Recovery Services Vault**

This is required to store replication metadata and orchestrate failover.

## **2. Hyper-V Site**

Represents your on-premises Hyper-V host(s) and is required so ASR can recognize and communicate with your Hyper-V environment.

## **3. Replication Policy**

Defines replication frequency, retention, and recovery point settings.


----------------

# Your company has two Azure virtual networks:

VNetA — has a VPN gateway

VNetB — peered with VNetA

Your on-prem network connects to VNetA using a site-to-site VPN.
A Windows 10 computer connects to VNetA using a point-to-site VPN.

On-premises devices can access VNetB, but the Windows 10 P2S client cannot access VNetB.

You enable Allow gateway transit on VNetA.

Does this fix the problem?

A. Yes
B. No

## Enable allow gateway transit on VNetA (if not already):

az network vnet peering update \
  --resource-group RG \
  --name PeeringFromAtoB \
  --vnet-name VirtualNetworkA \
  --allow-gateway-transit true


## Enable use remote gateways on VNetB:

az network vnet peering update \
  --resource-group RG \
  --name PeeringFromBtoA \
  --vnet-name VirtualNetworkB \
  --use-remote-gateways true


After both settings are configured (and prerequisites met), the P2S client connected to VirtualNetworkA should be able to reach VirtualNetworkB.

So the proposed single-step solution does not meet the goal.

-----------------------

#  two VPN types

# **Site-to-Site VPN (S2S)**

**Connects an entire on-premises network to an Azure virtual network.**

### Think of it like:

> **Office network ↔ Azure network**

### Used for:

-   Corporate offices connecting permanently to Azure
    
-   Branch offices connecting to central resources
    
-   Always-on connectivity
    

### Diagram:

```On-Prem Network ────────(VPN Tunnel)──────── Azure VNet``` 

----------

# **Point-to-Site VPN (P2S)**

**Connects a single device (like a laptop) directly to an Azure virtual network.**

### Think of it like:

> **Single computer ↔ Azure network**

### Used for:

-   Remote workers
    
-   Developers connecting from home
    
-   Admins testing connectivity
    

### Diagram:

```Laptop/PC ────────(VPN Tunnel)──────── Azure VNet``` 

----------

# Side-by-Side Comparison

| Feature         | Site-to-Site (S2S)     | Point-to-Site (P2S)          |
| --------------- | ---------------------- | ---------------------------- |
| Who connects?   | Whole on-prem network  | One device (PC/laptop)       |
| Gateway needed? | Yes                    | Yes                          |
| Always on?      | Yes                    | Optional (connect on demand) |
| Use case        | Corporate connectivity | Remote user connectivity     |

----------

# A diagram combining S2S + P2S

<img width="1024" height="454" alt="image" src="https://github.com/user-attachments/assets/c79ec8a3-fefc-424e-9e58-3828e0dc2162" />

    <img width="2575" height="906" alt="image" src="https://github.com/user-attachments/assets/a2cf8558-7a5a-48a5-8fb9-84c65f056526" />

-----

#  Real-world architecture examples

# **Hybrid Cloud Corporate Network (Most Common in Enterprises)**

✔ Corporate network connected via **S2S VPN**  
✔ Remote workers connected via **P2S VPN**  
✔ Shared services in a **Hub VNet**  
✔ Business apps in **Spoke VNets**

![https://miro.medium.com/0%2ADxyjWIP-IAvGxsZJ.png](https://miro.medium.com/0%2ADxyjWIP-IAvGxsZJ.png)

![https://learn.microsoft.com/en-us/azure/architecture/networking/architecture/_images/hub-spoke.png](https://learn.microsoft.com/en-us/azure/architecture/networking/architecture/_images/hub-spoke.png)

![https://learn.microsoft.com/en-us/azure/virtual-wan/media/virtual-wan-about/virtualwanp2s.png](https://learn.microsoft.com/en-us/azure/virtual-wan/media/virtual-wan-about/virtualwanp2s.png)

### How it works:

-   **Hub VNet** contains:
    
    -   VPN Gateway (S2S + P2S)
        
    -   Firewall / NVA
        
    -   Bastion
        
-   **On-premises** connects to hub using S2S IPSec.
    
-   **Remote users** connect using P2S (cert or AAD auth).
    
-   **Workloads live in spoke VNets**, reachable via peering.
    

This architecture ensures:

-   Centralized routing & security
    
-   Scalable networking
    
-   Segmentation between apps

------

# **Hybrid Cloud With Azure Firewall + Forced Tunneling**

A common security-driven setup:

![https://learn.microsoft.com/en-us/azure/architecture/networking/architecture/_images/hub-spoke.png](https://learn.microsoft.com/en-us/azure/architecture/networking/architecture/_images/hub-spoke.png)

![https://journeyofthegeek.com/wp-content/uploads/2020/04/lab.gif?crop=1&h=476&w=594](https://journeyofthegeek.com/wp-content/uploads/2020/04/lab.gif?crop=1&h=476&w=594)

![https://kodekloud.com/kk-media/image/upload/v1752882173/notes-assets/images/Microsoft-Azure-Security-Technologies-AZ-500-Explore-hub-and-spoke-topology/hub-and-spoke-azure-topology-diagram.jpg](https://kodekloud.com/kk-media/image/upload/v1752882173/notes-assets/images/Microsoft-Azure-Security-Technologies-AZ-500-Explore-hub-and-spoke-topology/hub-and-spoke-azure-topology-diagram.jpg)


### Flow:

-   All on-prem + P2S traffic enters **Hub Gateway**
    
-   Routes force traffic through **Azure Firewall**
    
-   Spokes cannot talk to each other unless firewall rules allow it
    

Ideal for:

-   Zero trust segmentation
    
-   Secure enterprise workloads
    
-   PCI, HIPAA, NIST environments

-----


# **3. Branch Offices Connected With Multiple S2S VPNs**

Large enterprises often have many branches connecting to Azure.

![https://learn.microsoft.com/en-us/azure/vpn-gateway/media/vpn-gateway-about-point-to-site-routing/multiple-s2s.jpg](https://learn.microsoft.com/en-us/azure/vpn-gateway/media/vpn-gateway-about-point-to-site-routing/multiple-s2s.jpg)

![https://learn.microsoft.com/en-us/azure/vpn-gateway/media/design/multi-site.png](https://learn.microsoft.com/en-us/azure/vpn-gateway/media/design/multi-site.png)

![https://learn.microsoft.com/en-us/azure/vpn-gateway/media/vpn-gateway-connect-multiple-policybased-rm-ps/policybasedmultisite.png](https://learn.microsoft.com/en-us/azure/vpn-gateway/media/vpn-gateway-connect-multiple-policybased-rm-ps/policybasedmultisite.png)


### Key points:

-   Multiple branch offices → multiple S2S tunnels
    
-   All centralized through a **Hub VNet VPN Gateway**
    
-   P2S users can also connect using OpenVPN/Azure AD
    

Used in:

-   Retail chains
    
-   Manufacturing plants
    
-   Global HQ + regional offices

-------

# **4. Multi-region DR with S2S + ASR + Peering**

![https://learn.microsoft.com/en-us/azure/virtual-wan/media/disaster-recovery-design/multi-branch.png](https://learn.microsoft.com/en-us/azure/virtual-wan/media/disaster-recovery-design/multi-branch.png)

![https://learn.microsoft.com/en-us/azure/route-server/media/multiregion/multiregion.png](https://learn.microsoft.com/en-us/azure/route-server/media/multiregion/multiregion.png)

![https://miro.medium.com/v2/resize%3Afit%3A1400/0%2AWnc1Q-zzEbSVFyWI.png](https://miro.medium.com/v2/resize%3Afit%3A1400/0%2AWnc1Q-zzEbSVFyWI.png)

4

### Components:

-   Region 1: Primary Hub (S2S + P2S)
    
-   Region 2: DR Hub
    
-   Azure Site Recovery (ASR) replicates VMs
    
-   Global peering between hubs
    

Used by:

-   Healthcare
    
-   Financial institutions
    
-   Government workloads

----


# **5. DevOps / Developer Remote Access Platform**

✔ Developers connect to P2S → Hub  
✔ Dev/test workloads in spoke VNets  
✔ Azure Bastion for secure RDP/SSH

![https://learn-attachment.microsoft.com/api/attachments/6da6a936-d517-4161-9431-13c9601cecb2?platform=QnA](https://learn-attachment.microsoft.com/api/attachments/6da6a936-d517-4161-9431-13c9601cecb2?platform=QnA)

![https://learn.microsoft.com/en-us/azure/architecture/networking/architecture/_images/hub-spoke.png](https://learn.microsoft.com/en-us/azure/architecture/networking/architecture/_images/hub-spoke.png)

![https://learn.microsoft.com/en-us/azure/virtual-wan/media/virtual-wan-about/virtualwanp2s.png](https://learn.microsoft.com/en-us/azure/virtual-wan/media/virtual-wan-about/virtualwanp2s.png)

------------

# **6. High-security Architecture With No Inbound Ports**

✔ P2S only  
✔ No inbound S2S  
✔ Azure AD authentication required  
✔ All workloads reachable only from P2S clients or Bastion

![https://media2.dev.to/dynamic/image/width%3D1000%2Cheight%3D500%2Cfit%3Dcover%2Cgravity%3Dauto%2Cformat%3Dauto/https%3A%2F%2Fdev-to-uploads.s3.amazonaws.com%2Fuploads%2Farticles%2F4x5abz0us2dgngwqaqa4.png](https://media2.dev.to/dynamic/image/width%3D1000%2Cheight%3D500%2Cfit%3Dcover%2Cgravity%3Dauto%2Cformat%3Dauto/https%3A%2F%2Fdev-to-uploads.s3.amazonaws.com%2Fuploads%2Farticles%2F4x5abz0us2dgngwqaqa4.png)

![https://learn.microsoft.com/en-us/azure/vpn-gateway/media/point-to-site-about/p2s.png](https://learn.microsoft.com/en-us/azure/vpn-gateway/media/point-to-site-about/p2s.png)

![https://learn.microsoft.com/en-us/azure/security/fundamentals/media/management/typical-management-network-topology.png](https://learn.microsoft.com/en-us/azure/security/fundamentals/media/management/typical-management-network-topology.png)

4

Used for:

-   Dev/test environments
    
-   Sensitive workloads
    
-   Security-first SaaS platforms

-----

# VPN Gateway vs ExpressRoute

| Feature      | **VPN Gateway**                              | **ExpressRoute**                               |
| ------------ | -------------------------------------------- | ---------------------------------------------- |
| Connectivity | Public Internet (encrypted IPSec tunnel)     | Private dedicated circuit (no public internet) |
| Performance  | Medium (100 Mbps – 10 Gbps depending on SKU) | High (up to 100 Gbps)                          |
| Reliability  | Depends on internet quality                  | SLA-backed telco-grade connection              |
| Cost         | Low                                          | High                                           |
| Security     | Encrypted over Internet                      | Private MPLS-like link                         |
| Best for     | Small/medium workloads, remote users         | Enterprise workloads, predictable throughput   |


# **Architecture Diagram: VPN Gateway (S2S + P2S)**

![https://learn.microsoft.com/en-us/azure/vpn-gateway/media/point-to-site-about/p2s.png](https://learn.microsoft.com/en-us/azure/vpn-gateway/media/point-to-site-about/p2s.png)

![https://learn.microsoft.com/en-us/azure/vpn-gateway/media/vpn-gateway-howto-point-to-site-rm-ps/point-to-site-diagram.png](https://learn.microsoft.com/en-us/azure/vpn-gateway/media/vpn-gateway-howto-point-to-site-rm-ps/point-to-site-diagram.png)

![https://learn.microsoft.com/en-us/azure/vpn-gateway/media/vpn-gateway-about-point-to-site-routing/multiple.jpg](https://learn.microsoft.com/en-us/azure/vpn-gateway/media/vpn-gateway-about-point-to-site-routing/multiple.jpg)

4

### How it works:

-   Uses **IPSec/IKE** tunnels over the **public Internet**
    
-   Can support:
    
    -   **Site-to-Site VPN** (office → Azure)
        
    -   **Point-to-Site VPN** (laptop → Azure)
        
-   Cheaper but dependent on internet quality
    
-   Great for backup/failover to ExpressRoute

----


# **Architecture Diagram: ExpressRoute**

![https://learn.microsoft.com/en-us/azure/expressroute/media/expressroute-introduction/expressroute-connection-overview.png](https://learn.microsoft.com/en-us/azure/expressroute/media/expressroute-introduction/expressroute-connection-overview.png)

![https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/hybrid-networking/images/expressroute-vpn-failover.svg](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/hybrid-networking/images/expressroute-vpn-failover.svg)

![https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/_images/guidance-hybrid-network-expressroute/figure3.png](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/_images/guidance-hybrid-network-expressroute/figure3.png)


### How it works:

-   Private, dedicated connection between your datacenter and Microsoft cloud
    
-   Not over public internet
    
-   Carrier or partner manages the circuit
    
-   Consistent latency and high throughput
    
-   SLA-backed and highly reliable

-------------


# **Detailed Comparison (Real-World View)**

## 1. **Connectivity Type**

-   **VPN Gateway**: encrypted traffic over the **public Internet**
    
-   **ExpressRoute**: private fiber/MPLS-like link
    

→ ExpressRoute is not exposed to the Internet.

----------

## 2. **Speed & Throughput**

-   VPN Gateway: 100 Mbps – ~10 Gbps (VpnGw SKUs)
    
-   ExpressRoute: 50 Mbps – **up to 100 Gbps** depending on SKU
    

→ ExpressRoute is significantly faster.

----------

## 3. **Latency**

-   VPN Gateway: varies with Internet weather
    
-   ExpressRoute: predictable, low latency
    

→ ExpressRoute preferred for databases, SAP, replication.

----------

## 4. **Security Model**

-   VPN Gateway:
    
    -   IPSec encryption required
        
    -   Public IP facing
        
-   ExpressRoute:
    
    -   No public IP
        
    -   Private peering to Microsoft backbone
        

→ ExpressRoute is inherently more secure.

----------

## 5. **Reliability / SLA**

-   VPN Gateway: relies on ISP quality; best-effort internet
    
-   ExpressRoute: carrier SLA + Azure SLA
    

→ ExpressRoute is enterprise-grade.

----------

## 6. **Use Cases**

### VPN Gateway is best for:

-   Small/medium businesses
    
-   Development/testing environments
    
-   Backup for ExpressRoute
    
-   Quick connectivity without telco involvement
    
-   Remote user access (P2S)
    

### ExpressRoute is best for:

-   Large production workloads
    
-   Low latency requirements (SQL, SAP, ERP)
    
-   Large data migrations (Data Box + ER)
    
-   High bandwidth scenarios (multi-GB transfers)
    
-   Compliance-driven environments
    

----------

# **Using Both: ExpressRoute + VPN Failover**

The most **enterprise** solution is to use:

**ExpressRoute = primary**  
**VPN Gateway = automatic failover**

![https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/hybrid-networking/images/expressroute-vpn-failover.svg](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/hybrid-networking/images/expressroute-vpn-failover.svg)

![https://learn.microsoft.com/en-us/azure/architecture/networking/architecture/_images/hub-spoke.png](https://learn.microsoft.com/en-us/azure/architecture/networking/architecture/_images/hub-spoke.png)

![https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/_images/guidance-hybrid-network-expressroute/figure3.png](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/_images/guidance-hybrid-network-expressroute/figure3.png)

4

This gives:

-   High performance (ER)
    
-   High availability (VPN backup)
    
-   BGP handles automatic routing failover
    

----------

# 🎯 **Simple Decision Guide**

### Choose **VPN Gateway** if:

✔ You need fast, cheap deployment  
✔ You need P2S (remote workers)  
✔ You have small/medium workloads  
✔ You want DR/failover for ExpressRoute

### Choose **ExpressRoute** if:

✔ You need predictable high-speed performance  
✔ You run enterprise apps (SAP/SQL clusters)  
✔ You need private connections (not Internet-based)  
✔ You require SLA-backed reliability

### Choose **Both** if:

✔ You want mission-critical reliability  
✔ You need automatic failover (BGP)  
✔ You run large workloads but need Internet-based resilience

--------

For a SQL Server Always On availability group **listener** behind an **Azure Internal Load Balancer (ILB)**, you must configure a **TCP health probe**, **NOT an HTTP probe**.

### Requirements for AG Listener in Azure:

-   Use an **Internal Load Balancer (ILB)**
    
-   Configure a **TCP health probe**
    
-   Typical probe ports: **59999**, **59990**, or any unused custom TCP port
    
-   **NOT port 1433** (that’s the SQL endpoint)
    
-   **NOT HTTP** (AG listener does not respond to HTTP)
    

### Why the proposed solution fails:

-   An **HTTP probe** looks for HTTP responses; SQL Always On availability groups do **not** return HTTP responses.
    
-   Therefore, the ILB would mark the SQL node as _unhealthy_.
    
-   And using port **1433** for the probe also doesn’t work because probes must use a **separate, dedicated TCP port**.
    

----------

# Correct solution would be:

Create a **TCP** health probe on a **custom port** (e.g., 59999) and configure the probe endpoint on each SQL node using:

```powershell
New-NetFirewallRule -DisplayName "ProbePort" `
  -Direction Inbound `
  -LocalPort 59999 `
  -Protocol TCP `
  -Action Allow
```
----------



For configuring a **SQL Server Always On Availability Group Listener** behind an **Azure Internal Load Balancer (ILB)**, the required load balancer setting is:

### ** Session persistence = None**

This ensures that the ILB distributes traffic based on the ILB rule + probe result, allowing SQL connections to always route to the **current primary replica**.

----------

#  Why "Client IP" is incorrect

Setting **Session persistence = Client IP** causes the load balancer to _pin_ a client’s traffic to a specific backend node.  
But in a SQL Always On configuration:

-   Only the **primary** replica should receive traffic
    
-   When failover occurs, the ILB must route traffic to the **new primary**
    
-   Sticky sessions (Client IP persistence) **break failover routing**
    

This leads to:

-   Clients being stuck to the old server
    
-   Connection failures after failover
    
-   Incorrect listener behavior
    

Therefore, setting persistence to **Client IP does NOT meet the goal**.

# Correct Load Balancer Settings for SQL AG Listener

| Setting                                | Required                               |
| -------------------------------------- | -------------------------------------- |
| **LB Type**                            | Internal Load Balancer                 |
| **Protocol**                           | TCP                                    |
| **Health Probe**                       | TCP probe on custom port (e.g., 59999) |
| **Session Persistence**                | **None**                               |
| **Floating IP (Direct Server Return)** | **Enabled**                            |
| **Backend Pool**                       | SQL AG nodes                           |


# **Settings that will NOT work**

-   HTTP health probe
    
-   Health probe on port 1433
    
-   Session persistence = **Client IP**
    
-   TCP probe missing or misconfigured
    
-   Floating IP = **Disabled**

------


Each Azure virtual machine needs at least **one network interface (NIC)** to connect to a virtual network subnet.

For this question:

### Requirements:

-   Five VMs
    
-   Each VM needs:
    
    -   **1 private IP**
        
    -   **1 public IP**
        
-   Inbound/outbound **security rules must be identical** on all VMs
    

### Key Azure facts:

-   A **NIC can have multiple IP configurations**:
    
    -   1 primary private IP
        
    -   Additional private IPs (optional)
        
    -   1 associated public IP (optional)
        
-   **Network Security Groups (NSGs)** can be applied to:
    
    -   the NIC, or
        
    -   the subnet
        

So for each VM:

-   **1 NIC**
    
-   Add **one public IP** to the NIC
    
-   Attach the **same NSG** to all NICs (or the subnet)
    

No need for multiple NICs unless:

-   You need multiple subnets
    
-   Or require traffic isolation per NIC (not required here)
    

Therefore:

### ➤ **Five VMs × 1 NIC each = 5 NICs total**

Public IPs are attached to the NIC, not separate NICs.

You can create a single Network Security Group (NSG) with the required inbound and outbound rules and associate it to the subnet that contains the five VMs. All VMs in that subnet inherit the NSG rules, so one NSG is sufficient to enforce identical security for all VMs.

-------

Your company's Azure subscription includes Azure virtual machines (VMs) that run Windows Server 2016.
One of the VMs is backed up every day using Azure Backup Instant Restore.
When the VM becomes infected with data encrypting ransomware, you are required to restore the VM.
Which of the following actions should you take?

A. You should restore the VM after deleting the infected VM.
B. You should restore the VM to any VM within the company's subscription.
C. You should restore the VM to a new Azure VM. Most Voted
D. You should restore the VM to an on-premise Windows device.


This scenario is **not a file-level recovery** (like the previous question).  
This time, the requirement is:

> **“You are required to restore the VM.”**

This refers to a **full VM restore**, not just recovering files.

With Azure Backup:

### 🔹 **When restoring a protected Azure VM, you have two supported options:**

1.  **Restore the VM as a _new_ VM**
    
2.  **Restore disks only**, then create a VM manually
    

### 🔸 What you _cannot_ do:

-   **You cannot overwrite the existing infected VM directly**  
    Azure Backup **does not support in-place overwrite restore** for IaaS VMs.
    

Because the existing VM is infected with ransomware, the **safe and supported approach** is:

✔ Restore as a **new Azure VM**  
✔ Then delete the infected VM  
✔ Reattach networking / reconfigure IPs as needed

----------

# Why other answers are incorrect

### **A. Restore the VM after deleting the infected VM — Incorrect**

Azure Backup does **not** restore _into_ the same old VM resource.  
Restoring after deletion still creates a **new** VM internally.  
So this answer is misleading.

### **B. Restore the VM to any VM — Incorrect**

That applies only to **file-level recovery**, not full VM recovery.

### **D. Restore the VM to an on-premise Windows device — Incorrect**

Azure VM restore cannot be restored directly on-prem; at best, disks can be downloaded manually, but that’s not a supported "VM restore."
--------

You are troubleshooting **performance issues** on Azure infrastructure (VMs, storage, networking, PaaS resources, etc.).  
The tool designed specifically to collect, analyze, and visualize **performance metrics** is:

## 🔹 **Azure Monitor**

Azure Monitor provides:

-   CPU / Memory / Disk metrics
    
-   Network throughput
    
-   Storage latency
    
-   App Insights telemetry
    
-   Log Analytics integration
    
-   Alerts based on metric conditions
    

This is exactly what you need to diagnose performance problems.

----------

# Why the other options are wrong

### **A. Azure Traffic Analytics**

-   Focuses on **NSG flow logs**, traffic patterns, and security analysis
    
-   Not about VM/storage/performance metrics
    

### **C. Azure Activity Log**

-   Shows **control-plane operations** (resource creation, updates, access events)
    
-   Does **not** show performance metrics
    

### **D. Azure Advisor**

-   Provides **recommendations** for cost, performance, HA, and security
    
-   Does _not_ show live performance metrics
    
-   Useful after analysis, not for root cause
    

----------


To create **guest users** (B2B users) in Azure AD, you must **send an invitation**.  
The correct cmdlet is:

### **`New-AzureADMSInvitation`**

This cmdlet:

-   Creates a **guest account** (UserType = _Guest_)
    
-   Sends the invitation email
    
-   Sets up the external identity properly for B2B access

-----


# You have an Azure Active Directory (Azure AD) tenant named contoso.com. 

You have a CSV file that contains the names and email addresses of 500 external users. You need to create a guest user account in contoso.com for each of the 500 external users. 

Solution: You create a PowerShell script that runs the New-AzureADUser cmdlet for each user. Does this meet the goal?



**PowerShell script** to bulk-invite 500 external users from a CSV file into Azure AD using **`New-AzureADMSInvitation`**.

This is the _right_ method for creating **guest users** (B2B).

----------

# **CSV Format (example)**

Save as **users.csv**:

```
Email,DisplayName
bob@example.com,Bob Smith
alice@partner.com,Alice P
john@vendor.org,John Vendor
``` 

----------

# **PowerShell Script (Works for Bulk Guest Invitations)**

> Requires the **AzureAD** or **AzureADPreview** module.

```
# Connect to Azure AD
Connect-AzureAD

# Import the CSV
$users = Import-Csv -Path "C:\temp\users.csv"

foreach ($user in $users) {
    New-AzureADMSInvitation `
        -InvitedUserEmailAddress $user.Email `
        -InvitedUserDisplayName $user.DisplayName `
        -SendInvitationMessage $true `
        -InviteRedirectUrl "https://myapps.microsoft.com"
}
``` 

----------

# **What this script does**

-   Reads all rows from **users.csv**
    
-   Invites each external user
    
-   Creates a **Guest** account (UserType = _Guest_)
    
-   Sends them an invitation email
    
-   Redirects them to the MyApps portal after accepting
