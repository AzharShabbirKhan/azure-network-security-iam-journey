# AZ-104 Week 2 Theory: Azure Governance

## The big picture

Week 1 answered *who may do what*. Week 2 answers *where things live, what is allowed, what is protected and what it costs*. Each tool below has one job, and the exam tests whether you pick the right one.

| Tool | Question it answers | Works at | Remember |
| --- | --- | --- | --- |
| Management group | How do I organize many subscriptions? | Above subscriptions | Settings inherit downward; up to 6 levels deep, not counting the root and subscription levels |
| Subscription | Where is the billing and access boundary? | Under a management group | Trusts exactly one Entra tenant |
| Resource group | Which resources share a lifecycle? | Inside a subscription | A resource lives in only one group; deleting the group deletes its resources |
| Tags | How do I label resources by owner, cost center or environment? | Subscriptions, resource groups, resources (not management groups) | Not inherited by default |
| Azure Policy | What may a resource look like? | Management group, subscription, resource group, resource | Judges resource properties, whoever made the change |
| Resource locks | How do I stop accidental delete or change? | Subscription, resource group, resource (not management groups) | Overrides every role, even Owner |
| Azure RBAC | Who may act? | All scopes | Week 1 |
| Cost Management | What am I spending, and who warns me? | Subscription, resource group | Budgets alert; they do not stop spending |

RBAC and Policy are easy to mix up: **RBAC controls who can act, Policy controls what the result may look like.** Even an Owner cannot create a resource that a Deny policy forbids.

**Your lab:** you have one subscription, so the Saturday hierarchy uses one management group, your subscription and extra resource groups.

## Monday: management groups, subscriptions and resource groups

Azure has four management scopes: **management group > subscription > resource group > resource**. A setting applied at one level is inherited by everything below it, which is why the exam keeps asking "where do you assign this?".

### Management groups

- A management group is a governance scope above subscriptions. Policy and RBAC assigned to it cascade to every subscription underneath.
- Every directory has one **root management group**, called *Tenant root group* by default. Its ID equals the Entra tenant ID. It cannot be moved or deleted, and new subscriptions land there by default.
- Limits to remember: up to **10,000** management groups per directory, up to **6 levels** of depth (not counting the root and subscription levels), and each management group or subscription has **only one parent**.
- All subscriptions in a hierarchy must trust the **same Entra tenant**.
- Nobody has default access to the root. A **Global Administrator** can **elevate access** (User Access Administrator at root), then assign roles from there. Keep root assignments to the "must have" minimum, because they apply to everything in the tenant.
- Moving a subscription needs write permission on the subscription, the current parent and the target parent. Azure caches the hierarchy for up to **30 minutes**, so the portal may lag.
- You **cannot** apply tags or locks to a management group.

### Subscriptions

- A subscription is the **billing and access boundary**. It holds resource groups, and each trusts one Entra tenant.
- You can move a subscription between management groups, and organize several subscriptions (for example production and development) under one management group to give them the same policies and role assignments.
- Cancelling a subscription does not delete resources immediately: Azure deactivates them and deletes them permanently after a waiting period.

### Resource groups

- A resource group is a **container for resources that share a lifecycle**: you deploy, update and delete them together.
- **Each resource lives in exactly one resource group.** You can add, remove or **move** resources between groups at any time.
- Deleting a resource group deletes **all** its resources.
- The group has a **location**, which only says where its **metadata** is stored. Its resources can be in other regions, though the same region is recommended.
- Resources in a group can connect to resources in other groups.
- Limit: up to 800 instances of a resource type per resource group (some types are exempt).
- Policy, roles, locks and tags can all be applied to a resource group, but resources do **not inherit its tags**.

### Tags

- Tags are **name-value pairs** such as `Environment = Production`, applied to resources, resource groups and subscriptions.
- Limits: **50** tag pairs per resource, resource group or subscription; tag names up to **512** characters and values up to **256**. Tag names are **case-insensitive**, tag values are **case-sensitive**.
- Tags are plain text: never store secrets in them.
- Not every resource type supports tags.
- **No inheritance by default.** To copy tags down from a resource group or subscription, use an Azure Policy (the *Inherit a tag* policies); see Wednesday.
- To apply tags you need write access to the resource (Contributor) or the **Tag Contributor** role, which only needs access to the tags.
- Tags appear in cost reports, so they are how you group spending by cost center or environment.

### Commands you will use in the lab

```powershell
New-AzManagementGroup -GroupName mg-lab -DisplayName "Lab"
New-AzResourceGroup -Name rg-lab-week02 -Location westeurope -Tag @{Environment="Lab"}
Move-AzResource -DestinationResourceGroupName rg-lab-week02 -ResourceId <id>
```

The first creates a management group, the second a resource group with a tag, and the third moves a resource to another group.

**Lab link:** Saturday you build the hierarchy and watch an assignment cascade down it.

## Wednesday: Azure Policy, locks and tag enforcement

### Azure Policy: the objects

- A **policy definition** is a JSON business rule: a condition plus an **effect** that happens when the condition matches. Built-ins include *Allowed locations*, *Allowed resource types*, *Allowed virtual machine SKUs* and *Add a tag to resources*.
- An **initiative definition** (a *policySet* in PowerShell and the CLI) groups several definitions toward one goal. Best practice: assign initiatives even for a single policy, so you can add policies later without adding assignments.
- An **assignment** applies a definition or initiative to a **scope**: management group, subscription, resource group or resource. Child scopes inherit it, and you can **exclude** sub-scopes (*notScopes*) or grant a time-limited **exemption**.
- **Parameters** make a definition reusable, for example one *allowed locations* rule assigned with different region lists.
- Policy objects (definitions, initiatives, assignments) are readable by every role at that scope, for transparency.
- Although you can assign at a management group, only resources at the **subscription or resource group level** are evaluated.

### Effects

Each definition has **one** effect. Evaluation order: `disabled` first, then `append` and `modify`, then `deny`, then `audit`, then `auditIfNotExists`, and `denyAction` last.

| Effect | What it does | Typical use |
| --- | --- | --- |
| Deny | Blocks the create or update that breaks the rule | Allowed locations, required tag |
| Audit | Lets it happen, records it as non-compliant | Safe first step before enforcing |
| AuditIfNotExists | Flags a resource when a related resource is missing | VM without an extension |
| Append | Adds fields to the request as it is created or updated | Add a tag on creation |
| Modify | Changes properties or tags, and can fix existing resources through a remediation task | Add or inherit tags |
| DeployIfNotExists | Deploys a related resource when it is missing | Deploy diagnostic settings |
| DenyAction | Blocks a specific action such as delete | Protect critical resources |
| Disabled | Switches the rule off without removing the assignment | Testing |

- **Recommended start:** `audit` or `auditIfNotExists`, to see the impact before you enforce with `deny`, `modify` or `deployIfNotExists`.
- **Layered assignments are cumulative most restrictive.** If two assignments cover a resource, a resource must satisfy both; one Deny is enough to block it.
- Azure Policy is an **explicit deny system**. You cannot override a parent Deny by assigning a more permissive policy lower down. Exclude the child scope from the parent assignment instead.

### Compliance and remediation

- A resource is evaluated when it is created or updated in a scope with an assignment, when a new assignment lands, when an assigned definition changes, and in the **standard cycle every 24 hours**. A new assignment can take a while to show results.
- The **Compliance** page shows an overall percentage and lets you drill down to each resource.
- Existing non-compliant resources are fixed with a **remediation task**. It works with `modify` and `deployIfNotExists` assignments.
- Those two effects use a **managed identity** on the assignment, which needs enough permissions to change the targeted resources. **User Access Administrator** is what grants it.
- Roles: **Resource Policy Contributor** has most Policy operations; **Owner** has all; **Contributor** can trigger remediation but cannot create definitions or assignments; Reader can read.

### Policy versus RBAC

| | Azure RBAC | Azure Policy |
| --- | --- | --- |
| Controls | What users may do | What resources may look like |
| Looks at | The identity and the action | Resource properties (and, optionally, the requester) |
| Result | Allow or no access | Compliant, non-compliant, deny, fix |

Both apply together: a user with the right role still gets blocked if the result breaks policy.

### Tags with Policy

Resources do not inherit tags from their group, so Policy does it:

- **Require a tag on resources** (Deny) blocks creation without the tag.
- **Add a tag to resources** (Modify) adds a tag and value when it is missing.
- **Inherit a tag from the resource group** (Modify) copies a tag down from the group, and a remediation task fixes existing resources. A "*if missing*" variant does not overwrite an existing value.
- Tags are plain text and must not hold secrets.

### Resource locks

- Two lock levels. In the portal: **Delete** and **Read-only**. In PowerShell and the CLI: **CanNotDelete** and **ReadOnly**.
  - CanNotDelete: authorized users can read and change a resource, but not delete it.
  - ReadOnly: they can only read it, like holding the Reader role.
- A lock **overrides every user permission**, even Owner's, and applies to all users and roles.
- Locks apply at **subscription, resource group or resource**. They are **inherited** by everything below, including resources added later, and the **most restrictive lock wins**. They cannot be added to management groups.
- Locks protect the **control plane** only (management.azure.com). Data operations such as writing blobs still work.
- A ReadOnly lock can surprise you, because some operations that look harmless use POST: a read-only lock stops a VM from starting or restarting, and stops listing storage account keys.
- A Delete lock on a resource group blocks deleting the group, even when the other resources are unlocked.
- To create or remove locks you need `Microsoft.Authorization/locks/*`. **Owner** and **User Access Administrator** have it.

```powershell
New-AzResourceLock -LockName LockGroup -LockLevel CanNotDelete -ResourceGroupName rg-lab-week02
Get-AzResourceLock -ResourceGroupName rg-lab-week02
```

```
az lock create --name LockGroup --lock-type CanNotDelete --resource-group rg-lab-week02
az lock list --resource-group rg-lab-week02
```

Each pair creates a CanNotDelete lock on a resource group and then lists the locks on it.

**Lab link:** Saturday you assign Policy for allowed locations and a required tag, try to break both rules, read the compliance results, then add a CanNotDelete lock and test deleting.

## Friday: costs, Advisor and moving resources

### Cost Management and budgets

- **Cost analysis** shows spending over time. Group and filter it by resource group, service, location or **tag**, which is why tagging matters. Cost and usage data is typically available within **8 to 24 hours**.
- A **budget** compares spending with an amount you set and sends alerts. **It does not stop resources or spending.**
- Budget scopes: **management group, subscription, resource group**, and billing scopes. Supported by Azure RBAC scopes, so a budget on a resource group is a way to watch one project.
- Setup: a **name**, a **reset period** (monthly, quarterly, annually; pay-as-you-go also offers billing month, quarter and year, aligned to your invoice), an **expiration date**, an **amount**, and optional **filters** (a service or resource group).
- Alerts: at least one threshold (% of budget) and one email address, up to **5 thresholds** and **5 email addresses**. Thresholds can be on **Actual** cost or **Forecasted** cost. A forecast alert warns you before you overspend.
- Budgets are evaluated **every 24 hours**, and the email normally arrives within an hour after that.
- A budget can also call an **action group** (subscription and resource group scopes), for example to push a notification or trigger automation.
- A **new subscription** can take up to **48 hours** before you can use all Cost Management features.
- Permissions: **Owner** can create, edit or delete budgets; **Contributor** and **Cost Management Contributor** can manage their own budgets; **Reader** and **Cost Management Reader** can view them.
- Budget evaluations need all subscriptions in a scope to share **one currency**.
- Add `azure-noreply@microsoft.com` to your safe senders so alerts avoid spam.

### Azure Advisor

- Advisor analyzes your configuration and usage, then gives recommendations in five categories: **Reliability, Security, Performance, Cost, Operational excellence**.
- You can read recommendations as **Owner, Contributor or Reader** on a subscription, resource group or resource.
- You can change a recommendation's status to **Postponed**, **Dismissed** or **Completed**. After you apply one, Advisor can take **up to a day** to register it.
- It includes recommendations from Microsoft Defender for Cloud.

### Moving resources

- You can move resources between resource groups, or between subscriptions in the **same Entra tenant**. Both subscriptions must be active, and the destination subscription must be registered for the resource provider.
- The move **changes the resource ID**. Scripts, dashboards and templates that use the old ID must be updated. It does not change the resource's region.
- During the move, source and target resource groups are **locked for up to 4 hours**: no creating, deleting or updating resources in them. Existing resources keep running.
- Only the **parent resource** is named in the request; child resources move with it. Not every resource type supports moving.
- A **ReadOnly lock** on the source, the target or the subscription blocks the move.
- **Role assignments do not move** and become orphaned; recreate them afterwards. Tags, roles and policies on the group are not carried over either.
- You **cannot move a resource group** to another subscription. You move the resources in it.
- Required permissions: `moveResources/action` on the source group and `resourceGroups/write` on the destination group.
- Across subscriptions, a resource and its **dependent resources** must move together, from one resource group. At most 800 resources per operation.
- A policy that denies the resource in the target can fail the move (`RequestDisallowedByPolicy`).
- **Validate first** if you want to test the move without doing it.

```powershell
Move-AzResource -DestinationResourceGroupName rg-lab-week02 -ResourceId $resource.ResourceId
```

This moves a resource to `rg-lab-week02`; pass `-DestinationSubscriptionId` to change subscription.

### Scripts for Sunday's lab

```powershell
Get-AzResourceGroup | ForEach-Object { Update-AzTag -ResourceId $_.ResourceId -Tag @{Owner="azhar"} -Operation Merge }
Get-AzPolicyState -Filter "ComplianceState eq 'NonCompliant'"
```

The first line adds an `Owner` tag to every resource group and keeps existing tags (`Merge`). The second lists non-compliant policy results (it needs the `Az.PolicyInsights` module: `Install-Module Az.PolicyInsights`).

### How the controls combine

An action in Azure must pass all of these, in any order:

1. **RBAC**: you have a role that allows it.
2. **Locks**: no lock forbids it, even for Owner.
3. **Policy**: the result complies, or the effect is not Deny.

Cost Management and tags do not block anything; they inform. That is the exam distinction between *controls that stop* and *controls that report*.

**Lab link:** Sunday you create a budget with an email alert, review Advisor, run the tagging script and move a resource between groups.

## Self-check questions

1. **A policy is assigned on a management group. Which resources are evaluated?** Resources at the subscription and resource group level beneath it.
2. **You tag a resource group `Environment = Lab`. Do its resources get the tag?** No. Use an *Inherit a tag from the resource group* policy.
3. **Which effect blocks a resource without the required tag?** Deny (*Require a tag on resources*).
4. **Which effect is the safest first step when you introduce a new policy?** Audit (or auditIfNotExists): it reports without blocking.
5. **Two assignments, one Deny and one Audit, cover the same resource. What happens?** The effects are cumulative most restrictive: the Deny blocks it.
6. **A parent management group denies a resource type. Can a child subscription allow it with a permissive policy?** No. Policy is an explicit deny system; you exclude the child scope from the parent assignment.
7. **An Owner tries to delete a resource group that has a CanNotDelete lock. Result?** The delete fails. Locks override every role until the lock is removed.
8. **Can you put a lock or a tag on a management group?** No, neither.
9. **A budget reaches 100%. What happens to your resources?** Nothing: the alert is sent, but spending and resources are not stopped.
10. **You move a VM to another resource group. What happens to its role assignments and its resource ID?** The ID changes, and role assignments on it do not move; recreate them.
11. **Which effects need a managed identity on the assignment, and what fixes existing resources?** Modify and DeployIfNotExists; a remediation task.
12. **Where do you assign a policy so that all of your subscriptions get it?** On the management group above them (the root group applies to the whole tenant, so be careful).

Friday's plan also includes 20 practice questions across Weeks 1 and 2; use the Udemy simulations for those.

## Sources

Microsoft Learn pages opened for this doc, 13 October 2026:

- [What are Azure management groups?](https://learn.microsoft.com/azure/governance/management-groups/overview)
- [What is Azure Resource Manager?](https://learn.microsoft.com/azure/azure-resource-manager/management/overview)
- [What is Azure Policy?](https://learn.microsoft.com/azure/governance/policy/overview)
- [Azure Policy definitions effect basics](https://learn.microsoft.com/azure/governance/policy/concepts/effect-basics)
- [Assign policy definitions for tag compliance](https://learn.microsoft.com/azure/azure-resource-manager/management/tag-policies)
- [Use tags to organize your Azure resources](https://learn.microsoft.com/azure/azure-resource-manager/management/tag-resources)
- [Lock your Azure resources](https://learn.microsoft.com/azure/azure-resource-manager/management/lock-resources)
- [Tutorial: Create and manage budgets](https://learn.microsoft.com/azure/cost-management-billing/costs/tutorial-acm-create-budgets)
- [Introduction to Azure Advisor](https://learn.microsoft.com/azure/advisor/advisor-overview)
- [Move Azure resources to a new resource group or subscription](https://learn.microsoft.com/azure/azure-resource-manager/management/move-resource-group-and-subscription)
- [AZ-104 study guide](https://learn.microsoft.com/credentials/certifications/resources/study-guides/az-104)

The effect descriptions for Audit, Append, DeployIfNotExists and DenyAction, and the `Update-AzTag` and `Get-AzPolicyState` commands, come from general knowledge rather than the pages above; check them against the current documentation before you rely on them.
