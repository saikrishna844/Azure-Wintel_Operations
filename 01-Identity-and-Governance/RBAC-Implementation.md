# 🔐 Lab 01: Core Concepts of MicrsoftEntraID, Azure Policies, RBAC

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


---

---

## 🛡️ Part 2: Enterprise Governance via Azure Policy

### 💡 Core Concept: RBAC vs. Azure Policy
When discussing cloud security in enterprise environments, it is critical to distinguish between these two tools:
*   **RBAC (Who can do what):** Focuses on user actions and identity. It controls whether a specific user or group (like our `sec-tier1-support`) has the authorization to execute actions, such as restarting a virtual machine.
*   **Azure Policy (What resources can look like):** Focuses on resource properties and compliance. It acts as a guardrail ensuring that regardless of *who* is creating a resource, the resource itself must comply with enterprise standards (e.g., forbidding expensive VM sizes or mandating specific tags).

### 🎯 Real-World Scenario: Cost Control Guardrails
To prevent accidental over-spending within the `rg-enterprise-ops` resource group, an enterprise guardrail is required. We must restrict virtual machine deployments strictly to low-cost SKUs (`Standard_B1s` and `Standard_B2s`).

### 🛠️ Implementation Steps (Azure Portal)
*Note: Due to a known parsing bug in the Azure CLI regarding nested JSON parameters, this policy was deployed via the Azure Portal UI to ensure exact parameter matching.*

1. **Scope Definition:** Navigated to **Azure Policy** > **Assignments** and scoped the assignment strictly to the `rg-enterprise-ops` resource group to prevent subscription-wide disruptions.
2. **Policy Selection:** Assigned the built-in Azure Policy definition: **"Allowed virtual machine size SKUs"**.
3. **Parameter Configuration:** Configured the `Allowed Size SKUs` parameter to strictly permit only `Standard_B1s` and `Standard_B2s`.
4. **Enforcement:** Deployed the policy. Azure Resource Manager (ARM) now actively evaluates all compute deployments in this resource group against this rule.

### 🚨 Verification & Expected Behavior
*   **Compliant Attempt:** Deploying a `Standard_B1s` virtual machine succeeds seamlessly.
*   **Non-Compliant Attempt:** If an engineer attempts to provision an expensive, unapproved instance (e.g., `Standard_D4s_v5`), ARM blocks the deployment instantly during the pre-flight validation phase with a `RequestDisallowedByPolicy` error code.

---

<img width="1857" height="817" alt="image" src="https://github.com/user-attachments/assets/d23f2a33-9b20-4241-8f85-17a0770e699c" /> 

<img width="1906" height="767" alt="image" src="https://github.com/user-attachments/assets/20baa8ca-991c-47a0-b88c-fe9fced0f5a1" />


---

## Part 3- 🧠 Core Concepts of Azure RBAC

To understand cloud governance, imagine Azure as a highly secure corporate office skyscraper:

* **The Security Principal (The "Who"):** This is an employee with an ID badge. It can be an individual person, a department team (**Security Group** like `sec-tier1-support`), or an automated application account (**Service Principal**).
* **The Role Definition (The "What"):** This is a standardized job description sheet created by HR. Instead of writing custom rules for every single person, standard profiles are created (e.g., "Facilities Manager" or "Guest Reader") that outline exactly what tools and keys they are allowed to hold.
* **The Scope (The "Where"):** This is the exact physical area or floor of the building they are allowed into:
    * *Subscription Scope:* The employee is granted a master key to the entire skyscraper.
    * *Resource Group Scope:* The employee only has a key to one specific room or floor (e.g., `rg-enterprise-ops`). They are locked out of all other floors.

**Role-Based Access Control (RBAC)** is simply the act of taking an **employee's ID badge**, stamping a **job description sheet** onto it, and defining the **exact floor** where that badge is allowed to scan.

### 💡 The Golden Rule: Principle of Least Privilege
In an enterprise infrastructure, you do not give a junior technician a master key to the entire skyscraper just to change a lightbulb in the basement. You grant them access *only* to the basement, and *only* the tools needed to change that specific lightbulb. This is the **Principle of Least Privilege**—giving identities the absolute minimum access required to complete their operational tasks, and nothing more.

---

## 🛠️ Step-by-Step Guide: Assigning RBAC Roles via Azure Portal

This workflow demonstrates how to assign permissions using the Azure Portal graphical user interface (GUI) to maintain explicit boundary enforcement.

### Step 1: Navigate to Your Scope
1. Log into the Azure Portal.
2. In the top global search bar, type **Resource groups** and select it.
3. Click on the targeted deployment boundary: **`rg-enterprise-ops`**.

### Step 2: Open the Access Control (IAM) Blade
1. On the left-hand navigation menu of the resource group, click on **Access control (IAM)**.
2. Click the **+ Add** button at the top left of the blade, and select **Add role assignment** from the dropdown menu.

### Step 3: Select Your Role Definition
1. Under the **Job function roles** tab, use the search filter to find the required role.
2. *Tip:* If assigning a custom role, click the **Type** filter next to the search bar and check **CustomRole** to instantly isolate custom definitions.
3. Click on the role name to highlight it, then click **Next**.

### Step 4: Map the Security Principal (Members)
1. On the **Members** tab, ensure the radio button is set to **User, group, or service principal**.
2. Click the **+ Select members** link.
3. In the right-hand flyout panel, type the name of the destination Entra ID group: **`sec-tier1-support`**.
4. Click on the group name when it populates to add it to the active selection list, then click the blue **Select** button.
5. Click **Next**.

### Step 5: Review and Finalize Assignment
1. On the **Review + assign** tab, conduct a final validation flight ensuring the **Scope**, **Role**, and **Members** match design specs perfectly.
2. Click the **Review + assign** button at the bottom left to commit the changes to the Azure Resource Manager (ARM) engine.
3. Confirm deployment success when the green "Added Role assignment" notification appears in the portal header.

### 📸 Portal Verification Screenshot

<img width="1911" height="721" alt="image" src="https://github.com/user-attachments/assets/f9f22218-8b0a-47f0-9d84-e619ac54afb9" />  
<img width="1810" height="737" alt="image" src="https://github.com/user-attachments/assets/1ae2f6ce-df14-4b87-997a-73071d91392f" />

<img width="1907" height="747" alt="image" src="https://github.com/user-attachments/assets/1b4a683b-1355-471f-80a4-b2b60f86d279" />

<img width="1911" height="772" alt="image" src="https://github.com/user-attachments/assets/01117210-7cde-484f-9bc8-cb1171d0bfa4" />






---

