# 🔐 Lab 01: Enterprise RBAC & Custom Role Implementation

## 🎯 Real-World Scenario
In a hybrid infrastructure environment, the Tier-1 Support team needs the ability to monitor health, troubleshoot, and restart Azure Virtual Machines. However, enforcing the **Principle of Least Privilege** dictates they must *not* be able to delete resources, alter Network Security Groups (NSGs), or modify Virtual Networks. 

The built-in `Virtual Machine Contributor` role provides too much access (it permits resource deletion). Therefore, a Custom RBAC Role is required to strictly govern access.

## 🛠️ Tech Stack & Methods
*   **Identity Provider:** Microsoft Entra ID 
*   **Tools:** Azure CLI, JSON
*   **Concept:** Role-Based Access Control (RBAC), Infrastructure Governance

## 🚀 Hands-On Implementation Steps

### Step 1: Create the Entra ID Security Group
In an enterprise, roles are assigned to groups for scalable governance, never to individual users.
```bash
# Create the Tier-1 Support Group
az ad group create --display-name "sec-tier1-support" --mail-nickname "tier1support" --description "Tier 1 VM Support Team"
```

### Step 2: Define the Custom Role (JSON)
This custom JSON configuration enforces strict boundaries. It permits restarts in the Actions array but explicitly blocks network changes and deletions in the NotActions array.
```
{
  "Name": "Tier-1 VM Operator",
  "IsCustom": true,
  "Description": "Can read VM metrics and restart VMs, but cannot delete resources or alter networking.",
  "Actions": [
    "Microsoft.Compute/virtualMachines/read",
    "Microsoft.Compute/virtualMachines/restart/action",
    "Microsoft.Compute/virtualMachines/start/action",
    "Microsoft.ResourceHealth/availabilityStatuses/read"
  ],
  "NotActions": [
    "Microsoft.Compute/virtualMachines/delete",
    "Microsoft.Network/*"
  ],
  "AssignableScopes": [
    "/subscriptions/<Your Subscription ID>"
  ]
}
```
  To know your subscription ID just Open the Bash Azure CLI

 ``` az account show --query id -o tsv ```

### Step 3: Deploy the Custom Role Definition

  Register the custom role in Azure
``` az role definition create --role-definition custom-role-def.json ```

🕵️‍♂️ Verification & Troubleshooting (Real-World Roadblocks)
1. Syntax Gotcha: Placeholder Brackets
Issue: Initial deployment threw a malformed identifier error due to keeping documentation placeholders (<subscription_id>) in the string.

Resolution: Stripped the angle brackets (< >) out of the JSON string to ensure a literal, valid subscription path.

2. Tenant Privilege Constraint: AuthorizationFailed
Issue: Executing az role definition create returned the following explicit deny error:

Code: AuthorizationFailed

Message: The client does not have authorization to perform action 'Microsoft.Authorization/roleDefinitions/write' over scope...

Root Cause Analysis: The lab was executed using a training tenant account (@deccansoftstudents.onmicrosoft.com). In enterprise-governed environments, standard Contributor roles lack identity management rights. Altering IAM blades or writing custom definitions strictly requires the execution principle to hold the User Access Administrator or Owner role at the targeted scope.

Key Takeaway: This restriction perfectly illustrates the Principle of Least Privilege in a production ecosystem. Standard day-to-day engineers cannot alter security policies or access vectors without elevated administrative governance privileges.

### Step 4 : Screenshot of Output 
After that Authentication Issue I used my Personal Azure account and Followed the above steps and I allocated that Security Group to only one particular Resource Group and I added the Role Assignments in that Resource Group by Indentity Access Management(IAM). 

<img width="1907" height="810" alt="image" src="https://github.com/user-attachments/assets/b21c0668-5c57-4566-b12d-78141e0406f4" />
