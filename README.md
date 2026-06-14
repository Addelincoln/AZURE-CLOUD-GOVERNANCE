# Azure Resource Tags & Policy Governance

Step-by-Step Azure Portal Implementation Guide for Accountability Groups

This handbook provides a fully manual, portal-based guide to design, deploy, and verify a metadata tagging and governance framework in Microsoft Azure. Designed as a curriculum guide for accountability groups and peer learning, it covers tagging schema design, custom policy definitions, scope assignment, policy enforcement testing, and Resource Graph verification.

## 1. Governance Objectives & Tagging Schema

Organizations face 'cloud sprawl' as they scale. To maintain cost control and security auditing, we implement a metadata framework using key-value pairs (tags).

Our framework mandates five core tags for all provisioned resources:

| Tag Key | Allowed Values / Format | Stakeholder & Justification |
| --- | --- | --- |
| **Environment** | Prod, Stage, Dev, Test | IT / Ops: Separates production assets from test configurations for automation (e.g. non-prod shutdown). |
| **Owner** | Email address (e.g. user@company.com) | Operations: Maps a direct point-of-contact to the resource for alerting and security incident triage. |
| **CostCenter** | CC- followed by 4 digits (e.g. CC-1001) | Finance: Used for chargeback/showback reports in Microsoft Cost Management. |
| **Application** | Alphanumeric string (e.g. CustomerPortal) | Development: Groups resources logically by workload to map architectural dependencies. |
| **DataClassification** | Public, Internal, Confidential, Restricted | Security: Classifies data sensitivity. Drives conditional NSG/firewall audit rules. |

## 2. Step 1: Creating the Custom Policy Definition

Azure Policy allows us to programmatically enforce compliance. We will build a custom policy definition that blocks (Denies) resource creation if any mandatory tags are missing or contain unapproved values.

1. Sign in to the Azure Portal (https://portal.azure.com).
2. In the top search bar, type 'Policy' and select it from the services list.
3. Under the 'Authoring' section in the left-hand navigation pane, click on 'Definitions'.
4. Click the '+ Policy definition' button in the toolbar.
5. On the Policy Definition setup panel, configure the following metadata:
   - **Definition location**: Click the selector and choose your target Subscription.
   - **Name**: Enter `Enforce-Mandatory-Tags-And-Values`.
   - **Description**: Enter `Enforce Environment, Owner, CostCenter, Application, and DataClassification tags, with specific allowed values for Environment and DataClassification.`
   - **Category**: Select 'Use existing' and choose 'Tags' (or select 'Create new' and type 'Tags').
6. In the Policy Rule editor, delete the default JSON contents completely, and copy-paste the entire structured JSON block provided below:
```json
{
  "displayName": "Enforce Mandatory Tags and Allowed Values",
  "policyType": "Custom",
  "mode": "Indexed",
  "description": "Enforces presence of Environment, Owner, CostCenter, Application, and DataClassification tags on resources. Restricts Environment to [Prod, Stage, Dev, Test] and DataClassification to [Public, Internal, Confidential, Restricted].",
  "metadata": {
    "category": "Tags"
  },
  "parameters": {
    "effect": {
      "type": "String",
      "metadata": {
        "displayName": "Policy Effect",
        "description": "Enable or disable the execution of the policy. Use 'Audit' for testing or 'Deny' to enforce compliance strictly."
      },
      "allowedValues": [
        "Audit",
        "Deny",
        "Disabled"
      ],
      "defaultValue": "Deny"
    }
  },
  "policyRule": {
    "if": {
      "anyOf": [
        { "field": "tags['Environment']", "exists": "false" },
        { "field": "tags['Environment']", "notIn": ["Prod", "Stage", "Dev", "Test"] },
        { "field": "tags['Owner']", "exists": "false" },
        { "field": "tags['CostCenter']", "exists": "false" },
        { "field": "tags['Application']", "exists": "false" },
        { "field": "tags['DataClassification']", "exists": "false" },
        { "field": "tags['DataClassification']", "notIn": ["Public", "Internal", "Confidential", "Restricted"] }
      ]
    },
    "then": {
      "effect": "[parameters('effect')]"
    }
  }
}Click 'Save' at the top of the portal. Your custom policy definition is now registered.3. Step 2: Assigning the Policy and Scope
An active policy assignment maps the definition to a scope (a Subscription or Resource Group).In Policy > Definitions, find and select your newly created policy: 'Enforce Mandatory Tags and Allowed Values'.Click the 'Assign' button in the toolbar at the top.On the Basics tab, configure the scope:Scope: Click the selector and select your active Subscription. Optionally, choose a specific Resource Group (e.g. create a test Resource Group named rg-governance-demo and scope the policy specifically to it to prevent blocking your other resource workloads).Assignment name: Enter Assign-Enforce-Tags.Enforcement: Ensure it is set to 'Enabled'.Click 'Next' or navigate to the Parameters tab:Policy Effect: Select 'Deny' (this forces Azure to reject any non-compliant deployments).Click 'Next' through Non-compliance messages, then click 'Review + create'.Verify your assignment details and click 'Create'.Note: Azure Policy assignment takes 10 to 30 minutes to propagate and take full effect.4. Step 3: Testing Policy Enforcement (Deny and Allow)
To verify that the Policy is actively blocking non-compliant resources and allowing compliant ones, we will run two tests.TEST A: Non-Compliant Attempt (Should Fail)
Navigate to the Storage Accounts services in the Azure Portal.Click '+ Create' in the toolbar.Configure basic settings: Resource Group: rg-governance-demo, unique name (e.g., noncompliantst + random number).Do NOT click next to the Tags tab. Leave the tags completely blank.Click 'Review + Create' directly.Wait for the validation bar. You should see a red validation banner stating: 'Validation failed. Click here to view details.'Click on the error message. You will see a detailed JSON error stating that the resource was disallowed by policy: 'Enforce Mandatory Tags and Allowed Values Assignment'.Screenshot Opportunity 1: Take a screenshot of the Azure Portal validation failure error message. The error details must clearly show RequestDisallowedByPolicy and point to your custom policy assignment.TEST B: Compliant Attempt (Should Succeed)
Go back to the basics configuration pane of the Storage Account creation wizard.Click 'Next: Tags >' or select the Tags tab at the top.Add the five mandatory tags exactly as defined in the schema:Environment = DevOwner = admin@company.comCostCenter = CC-1001Application = GovernanceTestDataClassification = Internal
Note: Casing is case-sensitive! Ensure they match the schema keys exactly.Click 'Review + Create' again.Validation will now pass successfully. Click 'Create' to provision the storage account. Click 'Go to resource' once it completes.5. Step 4: Remediating Legacy Resources (Manual Tagging)
Existing resources deployed before the policy assignment will not be affected or deleted, but they are now flagged as non-compliant. We can fix (remediate) them manually in the portal or automate it using Cloud Shell.Open your Azure Portal and navigate to any existing, untagged resource (e.g. a VM or database).On the resource Overview page, click on 'Tags' on the right-side summary details.Add the 5 mandatory tag keys and their corresponding values.Click 'Apply'.Alternatively, to automate checking and merging tags, open the Azure Cloud Shell (>_ icon in the top toolbar) and run this structured CLI script (replace the resource group name rg-governance-demo with yours if needed):bash# 1. Define the resource group containing your resources
export RG="rg-governance-demo"

# 2. Loop through all resources in that group and merge default tags if they are missing
ids=$(az resource list --resource-group $RG --query "[].id" -o tsv)
for id in $ids; do
  name=$(az resource show --id "$id" --query "name" -o tsv)
  echo "Remediating resource: $name"
  az resource tag --ids "$id" --tags Environment=Dev Owner=admin@company.com CostCenter=CC-1001 Application=LegacyApp DataClassification=Internal --operation Merge
done6. Step 5: Compliance Audit & Resource Graph (KQL)
To audit your cloud compliance posture, you can review the Policy Compliance dashboard or query active resources using the Azure Resource Graph.In the top search bar of the portal, search for 'Resource Graph Explorer' and select it.In the query editor pane, paste the Kusto Query Language (KQL) code block shown below:
This query filters for all resources that have tags, projects the mandatory keys, and highlights compliant ones.kustoresources
| project name, type, tags, resourceGroup
| where isnotnull(tags)
| extend Environment = tostring(tags['Environment']),
         CostCenter = tostring(tags['CostCenter']),
         Owner = tostring(tags['Owner']),
         Application = tostring(tags['Application']),
         DataClassification = tostring(tags['DataClassification'])
| project name, type, resourceGroup, Environment, CostCenter, Owner, Application, DataClassification
| where isnotempty(CostCenter) or isnotempty(Environment)
| order by Environment desc, CostCenter asc, name ascClick the 'Run query' button in the toolbar.Review the results list showing all resource tags. Click 'Export to CSV' to download the data table if you need to compile a compliance report.Now, search for 'Policy' in the portal search bar and click on 'Compliance' in the left-side menu.Locate your assignment 'Assign-Enforce-Tags' (or 'Enforce Mandatory Tags and Allowed Values Assignment') from the list and click on it.Screenshot Opportunity 2: Take a screenshot of the Azure Policy Compliance Dashboard. The screen must show your policy assignment name, the compliance state (showing 100% compliant once all resources in the scope are tagged), and the compliant resources count.7. Step 6: Post-Training Cleanup
To clean up your test environment:Navigate to 'Policy' > 'Definitions'. Find your custom policy 'Enforce-Mandatory-Tags-And-Values' and delete the definition (which automatically deletes any associated assignments).Navigate to 'Resource groups', select rg-governance-demo, click 'Delete resource group', type the name to confirm, and click 'Delete'.
