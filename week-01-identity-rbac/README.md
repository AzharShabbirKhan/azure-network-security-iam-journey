# Week 1 Lab: Identities, Groups and Azure RBAC

Lab date: 2026-10-13. Theory notes: [theory.md](theory.md).

## Goal

Build a small identity setup in Microsoft Entra ID and control access to a resource group with Azure RBAC, using the portal first and then PowerShell and the Azure CLI.

## Environment

- Personal Entra tenant ("Default Directory"), Entra ID Free plan
- Azure subscription: pay-as-you-go
- Tools: Azure portal, Azure PowerShell (Az 16.4.0), Azure CLI 2.91.0
- Budget alert set before starting (monthly, 400 DKK, alerts at 50%, 80%, 100%)

## What I did

### 1. Resource group (portal, PowerShell, CLI)

- Created `rg-lab-week01` in West Europe with a tag.
- Created and deleted test groups `rg-test-ps` (PowerShell) and `rg-test-cli` (Azure CLI) to practice both tools.
- Confirmed that PowerShell and the CLI list the same groups, so both are connected to the same subscription.

```powershell
New-AzResourceGroup -Name rg-test-ps -Location westeurope
Get-AzResourceGroup | Select-Object ResourceGroupName, Location
Remove-AzResourceGroup -Name rg-test-ps -Force
```

```
az group create --name rg-test-cli --location westeurope
az group list --output table
az group delete --name rg-test-cli --yes
```

### 2. Users, groups and a guest (Entra admin center)

- Created 5 users with department and usage location: Ali Sales and Sara Sales (Sales), Omar IT and Lina IT (IT), Noah HR (HR).
- Created two security groups with assigned membership: `grp-it-team` (Omar, Lina) and `grp-sales-team` (Ali, Sara).
- Invited an external user (`Test Guest`) from a personal address, accepted the invitation and added the guest to `grp-sales-team`.

### 3. Azure RBAC

| Principal | Role | Scope |
|---|---|---|
| grp-sales-team | Reader | rg-lab-week01 |
| grp-it-team | Contributor | rg-lab-week01 |
| Noah HR | Reader | Subscription (inherited by the resource group) |

- **Check access** showed the expected effective permissions for every user, including the guest, who got Reader through the group.
- Contributor and Owner sit under the "Privileged administrator roles" tab in the role picker, not under "Job function roles".

```powershell
Get-AzRoleAssignment -ResourceGroupName rg-lab-week01 | Select-Object DisplayName, RoleDefinitionName, Scope
(Get-AzRoleDefinition -Name "Contributor").Permissions | Select-Object Actions, NotActions
```

Contributor has `Actions: *` but its `NotActions` exclude `Microsoft.Authorization/*/Write`, so a Contributor cannot assign roles.

## What I couldn't do and why

- **Dynamic group:** the dynamic membership rule needs Entra ID P1. Opening "Add dynamic query" returned a 401 "No access" error. The rule I would have used is `user.department -eq "Sales"`. I used assigned groups instead.
- **Group-based licensing:** no licenses exist in my tenant and no trial is offered on my pay-as-you-go account, so I could not test it.
- **SSPR pilot:** after selecting "Selected", the group picker did not appear and Save stayed disabled, so the pilot could not be configured. I reviewed the Authentication methods, Registration and Notifications settings instead.

## What I learned

- Entra roles and Azure roles are separate systems: Global Administrator manages the directory, not subscriptions.
- Role assignments are additive and inherited downward through the scope hierarchy.
- Contributor can manage resources but cannot assign roles. Only Owner, User Access Administrator or Role Based Access Control Administrator can.
- A guest user gets access through group membership like any other user.
- `Get-AzRoleDefinition` will change in Az 16: use `.Permissions[n].Actions` instead of the flattened `Actions` property.
- Azure now requires MFA for the portal, the CLI and PowerShell.

## Cleanup

Nothing billable was created. The resource group is empty and kept for Week 2.

## Screenshots

Subscription IDs, tenant IDs and email addresses are blurred.

- `screenshots/01-rg-powershell-cli.png`
- `screenshots/03-dynamic-group-needs-p1.png`
- `screenshots/06-sspr-needs-p1.png`
- `screenshots/07-rbac-assignments.png`
- `screenshots/08-check-access.png`
