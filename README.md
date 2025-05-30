# cst8922-task2
Below is a full Terraform-based solution that implements the requirement:

**Policy-based restrictions on premium services (e.g., Premium Storage), with automated approval workflows to unlock usage as needed.**

## ✅ Deliverables You Can Use
1- Terraform Script (main.tf) – creates the policy and deploys the Logic App via ARM template.

2- Logic App Template (logicapp-approval.json) – defines the approval workflow.

3- Workflow Description – explains how policy enforcement and automated approval works together.

## ✅ High-Level Architecture (Terraform + Logic App)
**1-Terraform Template**
- Creates an Azure Policy to deny Premium Storage usage unless a specific tag (e.g., AllowPremium = true) is present.
- Assigns the policy to a scope (Management Group, Subscription, or Resource Group).
- Deploys a Logic App for the approval workflow (defined as a separate ARM template).
- Integrates the Logic App via outputs or Azure portal notifications.

**2-Logic App (ARM Template)**
- Provides an approval-based workflow:
- Receives a request (e.g., via HTTP or email).
- Sends approval to designated users.
- Upon approval, updates resource with required tag.

### 🧾 1. Terraform Template (main.tf)

```bash
provider "azurerm" {
  features {}
}

# Variables
variable "resource_group_name" {
  default = "rg-policy-lockdown"
}

variable "location" {
  default = "East US"
}

# Resource Group
resource "azurerm_resource_group" "main" {
  name     = var.resource_group_name
  location = var.location
}

# Policy Definition: Deny Premium Storage SKUs unless tag is set
resource "azurerm_policy_definition" "deny_premium_storage" {
  name         = "deny-premium-storage-unless-approved"
  policy_type  = "Custom"
  mode         = "All"
  display_name = "Deny Premium Storage unless tagged as approved"

  policy_rule = jsonencode({
    "if": {
      "allOf": [
        {
          "field": "sku.name",
          "in": [
            "Premium_LRS",
            "Premium_ZRS",
            "Premium_GRS",
            "Premium_RA_GRS"
          ]
        },
        {
          "not": {
            "field": "tags['AllowPremium']",
            "equals": "true"
          }
        }
      ]
    },
    "then": {
      "effect": "deny"
    }
  })

  metadata = jsonencode({
    category = "Storage"
  })
}

# Policy Assignment
resource "azurerm_policy_assignment" "assign_premium_lockdown" {
  name                 = "assign-premium-lockdown"
  scope                = azurerm_resource_group.main.id
  policy_definition_id = azurerm_policy_definition.deny_premium_storage.id
  display_name         = "Restrict Premium Storage unless tagged"
  description          = "Prevents use of Premium SKUs unless explicitly approved"
  enforcement_mode     = "Default"
}

# Deploy Logic App via ARM template (separate file)
resource "azurerm_template_deployment" "logic_app" {
  name                = "deploy-logicapp-approval"
  resource_group_name = azurerm_resource_group.main.name
  deployment_mode     = "Incremental"

  template_body = file("${path.module}/logicapp-approval.json")

  parameters = {
    logicAppName = {
      value = "premiumApprovalWorkflow"
    }
    location = {
      value = var.location
    }
  }
}
```

### 🧰 2. Logic App ARM Template (logicapp-approval.json)
This example Logic App will start an approval email when triggered manually or via HTTP request.

```bash
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "logicAppName": {
      "type": "string"
    },
    "location": {
      "type": "string"
    }
  },
  "resources": [
    {
      "type": "Microsoft.Logic/workflows",
      "apiVersion": "2019-05-01",
      "name": "[parameters('logicAppName')]",
      "location": "[parameters('location')]",
      "properties": {
        "definition": {
          "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
          "contentVersion": "1.0.0.0",
          "actions": {
            "Approval_email": {
              "type": "Approval",
              "inputs": {
                "host": {
                  "connection": {
                    "name": "@parameters('$connections')['office365']['connectionId']"
                  }
                },
                "method": "Email",
                "parameters": {
                  "subject": "Approval Required: Premium Storage Request",
                  "body": "A request was made to allow Premium Storage. Approve?",
                  "to": "admin@example.com"
                }
              },
              "runAfter": {}
            },
            "Condition": {
              "type": "If",
              "expression": {
                "equals": [
                  "@body('Approval_email')?['response']",
                  "Approve"
                ]
              },
              "actions": {
                "Update_Tag": {
                  "type": "Http",
                  "inputs": {
                    "method": "PATCH",
                    "uri": "<RESOURCE_ID_ENDPOINT>",
                    "headers": {
                      "Content-Type": "application/json"
                    },
                    "body": {
                      "tags": {
                        "AllowPremium": "true"
                      }
                    }
                  }
                }
              }
            }
          },
          "triggers": {
            "manual": {
              "type": "Request",
              "kind": "Http",
              "inputs": {
                "schema": {}
              }
            }
          }
        },
        "parameters": {}
      }
    }
  ]
}
```
🔐 Replace <RESOURCE_ID_ENDPOINT> with your Azure REST API endpoint for tagging the resource.

## ✅ Summary of Workflow
1- Azure Policy denies Premium Storage SKUs unless the AllowPremium tag is true.

2- User attempts to provision Premium Storage → Denied.

3- User triggers Logic App (manually or via portal request).

4- Logic App sends approval email.

5- If approved, Logic App uses Azure REST API to tag the resource with AllowPremium=true.

6- User re-deploys with tag → Deployment succeeds.

### 🧠 Optional Enhancements
Use Azure Event Grid to trigger the Logic App when a deployment fails due to policy.

Integrate with Power Automate instead of Logic Apps if preferred.

Store approvals in an Azure Table or Cosmos DB for auditing.

Notify FinOps team via Teams/Slack on approval events.

### 📚 References
Azure Policy Overview: https://learn.microsoft.com/en-us/azure/governance/policy/overview

Logic App Approval Workflows: https://learn.microsoft.com/en-us/azure/logic-apps/logic-apps-create-approval

Terraform azurerm_policy_definition: https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/policy_definition

Azure REST API for tagging: https://learn.microsoft.com/en-us/rest/api/resources/resources/update-by-id

## ✅ What You Can Do Next
If you'd like to test or customize this setup:

**1- 🚀 Deploy locally using:**

```bash
terraform init
terraform apply
```
**2- 🧪 Simulate a premium storage request and trigger the Logic App.**

**3- 📤 Email approval will be sent (ensure you configure Office 365 or Outlook connector).**

**4- 🏷️ If approved, Logic App sets the required tag and lets you redeploy the resource.**

### 🧰 Optional Add-ons:

**1- ✅ Add audit logging for approvals via Azure Storage or Cosmos DB.**

**2- 🔄 Integrate this with Azure DevOps pipelines for pre-deployment validation.**

**3- 📦 Package it all into a GitHub repo with CI/CD.**






