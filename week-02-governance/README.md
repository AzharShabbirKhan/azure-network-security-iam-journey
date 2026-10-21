# Week 2 Lab: Azure Governance

Lab date: 2026-10-21. Theory notes: [theory.md](theory.md).

## Goal

Build a small governance setup in Azure: a management group hierarchy, two Azure Policy assignments that block non-compliant deployments, a resource lock, bulk tagging with PowerShell, and a resource move between resource groups.

## Environment

- Personal Entra tenant, pay-as-you-go subscription
- Tools: Azure portal and Azure PowerShell (Az 16)
- Budget alert from Week 1 still active (400 DKK per month), reused instead of creating a new one

## Hierarchy

```
Tenant Root Group
└── Lab (mg-lab)
    └── Azure subscription 1
        ├── rg-lab-week01
        └── rg-lab-week02
```

## What I did

### 1. Management group and resource group

- Created the management group `mg-lab` (display name `Lab`) in the portal and moved the subscription under it.
- Created `rg-lab-week02` in West Europe with tags, then verified the hierarchy.

```powershell
New-AzResourceGroup -Name rg-lab-week02 -Location westeurope -Tag @{Purpose="lab"; Week="2"}
Get-AzManagementGroup -GroupName mg-lab -Expand | Select-Object DisplayName, Children
Get-AzResourceGroup | Select-Object ResourceGroupName, Location
```

### 2. Azure Policy on rg-lab-week02

| Assignment | Built-in definition | Setting | Effect |
|---|---|---|---|
| Allowed locations - lab week02 | Allowed locations | West Europe, North Europe | Deny |
| Require Owner tag - lab week02 | Require a tag on resources | Tag name `Owner` | Deny (fixed in the definition) |

Both are scoped to the resource group only, so Week 1 resources are unaffected. Policy enforcement is **Default**, which means the policies actually block deployments.

### 3. Breaking the rules

I tried to create virtual networks (free) and read the errors.

| Attempt | Result |
|---|---|
| Region `eastus` with the `Owner` tag | Blocked: `RequestDisallowedByPolicy`, Allowed locations |
| Region `northeurope` without a tag | Blocked: `RequestDisallowedByPolicy`, Require Owner tag |
| Region `northeurope` with the `Owner` tag | Created |

Blocked deployments are recorded in the Activity log as `'deny' Policy action`. Policy > Compliance showed 100% compliant, with 0 non-compliant resources. That is expected: a Deny policy stops violations before the resource exists, so there is nothing to report as non-compliant.

### 4. Resource lock

On a throwaway group `rg-lock-test` (outside the policy scope):

```powershell
New-AzResourceGroup -Name rg-lock-test -Location westeurope
New-AzResourceLock -LockName LockGroup -LockLevel CanNotDelete -ResourceGroupName rg-lock-test -Force
Remove-AzResourceGroup -Name rg-lock-test -Force      # fails with ScopeLocked
Remove-AzResourceLock -LockName LockGroup -ResourceGroupName rg-lock-test -Force
Remove-AzResourceGroup -Name rg-lock-test -Force      # now succeeds
```

The lock blocked the deletion even though I am the Owner. A lock has to be removed before the protected group can be deleted.

### 5. Tagging with PowerShell

```powershell
Get-AzResourceGroup | ForEach-Object { Update-AzTag -ResourceId $_.ResourceId -Tag @{Owner="azhar"} -Operation Merge }
```

`Merge` adds the tag and keeps existing ones. The virtual network did not inherit the `Purpose` and `Week` tags from its resource group, which confirms that tags are not inherited. A third group, `NetworkWatcherRG`, appeared and was tagged too: Azure creates it automatically when the first virtual network is deployed in a region.

### 6. Azure Advisor

All five categories (Cost, Security, Reliability, Operational excellence, Performance) showed no recommendations. That is normal for a nearly empty lab subscription. Advisor only gives advice and changes nothing on its own.

### 7. Moving a resource

Moved `vnet-lab-ok` from `rg-lab-week02` to `rg-lab-week01`.

```powershell
Move-AzResource -DestinationResourceGroupName rg-lab-week01 -ResourceId $vnet.ResourceId -Force
```

The resource ID changed from `.../resourceGroups/rg-lab-week02/...` to `.../resourceGroups/rg-lab-week01/...`, and the region stayed North Europe: a move never changes the region.

## What went wrong and why

- **West Europe rejected new virtual networks** with `RequestDisallowedByAzure` ("the selected region is currently not accepting new customers"). This is a platform restriction on my subscription, not a policy. It is a different error code from `RequestDisallowedByPolicy`, and the difference is worth knowing. I used North Europe instead, which my policy allows.
- **The move reported an error but succeeded.** PowerShell returned `ResourceMoveFailed: All move in provider succeeded. However, the batch move job failed.` I checked the result afterwards: the network was in `rg-lab-week01`, `rg-lab-week02` was empty, and the new ID pointed to the new group. I did not find out why the batch job reported a failure.

## What I learned

- A Deny policy blocks the deployment before it happens, so compliance reports show no violations for it. The blocked attempts show up in the Activity log.
- Read the error code, not only the message: `RequestDisallowedByAzure` is the platform, `RequestDisallowedByPolicy` is my policy.
- Locks apply even to Owners and must be removed before deleting.
- Tags are not inherited, so bulk tagging with `Update-AzTag -Operation Merge` is how to apply them consistently.
- After a failed command, check the real state before assuming it failed.

## Cleanup

Nothing here is billable: policies, locks, management groups, Advisor, an empty virtual network and `NetworkWatcherRG` are all free. I am keeping the setup for later weeks.

## Screenshots

Subscription IDs, tenant IDs and email addresses are blurred.

- `screenshots/01-management-group-tree.png`
- `screenshots/01-powershell-hierarchy.png`
- `screenshots/02-allowed-locations-assignment.png`
- `screenshots/02-require-owner-tag-assignment.png`
- `screenshots/03-region-capacity-error.png`
- `screenshots/03-denied-location.png`
- `screenshots/03-denied-tag.png`
- `screenshots/03-allowed-vnet.png`
- `screenshots/03-compliance-overview.png`
- `screenshots/03-activity-log-denied.png`
- `screenshots/04-lock-blocks-delete.png`
- `screenshots/04-lock-portal.png`
- `screenshots/05-tags-powershell.png`
- `screenshots/06-advisor-overview.png`
- `screenshots/07-move-resource.png`
- `screenshots/07-move-error.png`
- `screenshots/07-rg-week01-portal.png`
