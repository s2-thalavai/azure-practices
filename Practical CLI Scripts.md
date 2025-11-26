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
