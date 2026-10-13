# AZ-104 Week 1 Theory: Identities, Groups and Azure RBAC

Week 1 covers who can sign in (Microsoft Entra ID) and what they can do in Azure (Azure RBAC), which together make up about a quarter of the AZ-104 exam.

## The big picture

Microsoft Entra ID answers "who are you?" and Azure RBAC answers "what may you do to Azure resources?". They are managed separately, and each Azure subscription is associated with one Entra tenant.

| Question | System | Where you manage it | Example roles |
| --- | --- | --- | --- |
| Who signed in? | Microsoft Entra ID (users, groups, guests) | Entra admin center: Users and Groups | none, these are identities |
| What may someone do inside Entra, such as create users? | Entra roles | Entra admin center: Roles and administrators | Global Administrator, User Administrator |
| What may someone do to Azure resources, such as create a VM? | Azure RBAC | Access control (IAM) on a management group, subscription, resource group or resource | Owner, Contributor, Reader |

The exam mixes these two role systems on purpose, so always ask which system a question is about.

### What your personal tenant can do

Your sign-in `joinazhar@gmail.com` most likely created a tenant called "Default Directory" on the Microsoft Entra ID Free plan. Check the plan under Entra admin center > Overview. Some Week 1 features need a paid plan.

| Week 1 feature | Entra ID Free | Needs a paid license |
| --- | --- | --- |
| Create users, guests and groups with assigned membership | Yes | |
| Azure RBAC with built-in roles | Yes | |
| SSPR: a cloud-only user changes a password they know | Yes | |
| SSPR: a cloud-only user resets a forgotten password | | Entra ID P1 or P2, or Microsoft 365 Business Standard or Premium |
| Dynamic membership groups | | Entra ID P1 or P2 |
| Custom Entra roles | | Entra ID P1 or P2 |

**Lab consequence:** the dynamic group and the SSPR reset test in your Saturday lab need Entra ID P1. Look under Entra admin center > Licenses for a trial and start it only on lab day, because trials run out. If none is offered, do those parts as theory and do the rest of the lab as planned.

## Monday: Entra ID, users, groups and licenses

**Microsoft Entra ID** is the cloud identity service behind Azure and Microsoft 365. It stores identities (users, groups, apps) and answers one question: *who are you?* Every Azure subscription trusts exactly one Entra tenant.

### Users

- A user has a **User principal name (UPN)** like `ali@yourtenant.onmicrosoft.com`, which is the sign-in name, plus properties such as department, job title, manager and **usage location**.
- **Usage location** (a country) is mandatory before you can assign a license, because service availability depends on the country.
- Create users one by one in the portal, in bulk with a CSV template (Bulk create), or with PowerShell/Graph.
- Deleted users go to **Deleted users** and can be restored for **30 days**, after which they are permanently removed.

### Groups

- **Security group**: used to grant access to resources and Azure roles. **Microsoft 365 group**: adds a shared mailbox, Teams and SharePoint; also usable for access.
- **Membership type**:
  - *Assigned*: you add members manually.
  - *Dynamic user*: a rule adds or removes members automatically, for example `user.department -eq "Sales"`. Needs Entra ID P1 (see section 1).
  - *Dynamic device*: same idea for devices.
- Group **owners** manage membership; members inherit whatever the group has been granted.
- Giving access to groups instead of individuals is the core best practice: when a person changes role, you change the group, not twenty assignments.

### Licenses

- A license (for example Entra ID P1, Microsoft 365) is assigned to a user **directly** or through a group (**group-based licensing**).
- With group-based licensing every member of the group receives the license, and it is removed when they leave the group.
- Rules to remember: the user needs a **usage location**; **nested groups are not supported** for licensing (only direct members count); licensing errors (not enough licenses, conflicting service plans) show on the group, and after fixing the cause you use **Reprocess**.
- The plan required for group-based licensing is **not confirmed here**: check it in your own tenant when you open Licenses > All products.

**Lab link:** Saturday you create 5 users, 2 groups (one dynamic), assign a license through a group and watch it appear on the user.

## Wednesday: external users and self-service password reset

### External users (B2B collaboration)

- A **guest user** is someone from another organization (or with a personal email) invited into your tenant to use your resources, while they keep signing in with their own identity.
- Flow: you **invite** them by email, they **redeem** the invitation (accept and consent), and the guest object becomes active. Until redeemed, the invitation is pending.
- Guest accounts have user type **Guest** and a UPN containing `#EXT#`, for example `bob_gmail.com#EXT#@yourtenant.onmicrosoft.com`.
- Guests can be added to groups and given Azure RBAC roles like any other user.
- Two settings control this:
  - **External collaboration settings**: who may invite guests (admins, members, guests, anyone) and which domains are allowed or blocked.
  - **Guest access restrictions**: how much of the directory a guest can see (same as members, limited, or most restricted; limited is the default).
- **Cross-tenant access settings** are the more advanced control for trusting specific partner organizations.

### Self-service password reset (SSPR)

- SSPR lets users reset or unlock their own password without calling the help desk.
- **Enablement scope**: *None*, *Selected* (a group, ideal for a pilot) or *All*.
- **Authentication methods**: you choose how many methods are required to reset (**1 or 2**) and which are allowed (mobile app notification or code, email, phone call, SMS, security questions).
- Users must **register** their methods first; you can require registration at sign-in and set **reconfirm after N days**.
- **Notifications** tell users and admins when a password is reset.
- **Password writeback** syncs a cloud reset back to on-premises Active Directory (needs Entra Connect).
- **Administrators are always required to use two methods** and the strong policy, regardless of your settings.
- **Personal Microsoft accounts** invited as guests cannot use SSPR in your tenant.
- Licensing: SSPR for cloud users beyond the basic change-password flow needs **Entra ID P1** or higher, so in the free tenant you can read the settings but may not be able to test the full flow (see section 1).

**Lab link:** Saturday you invite a guest and enable SSPR for a pilot group.

## Friday: Azure role-based access control (RBAC)

Azure RBAC answers *what can you do with Azure resources?* An **assignment** has three parts: a **security principal** (who), a **role definition** (what actions), and a **scope** (where).

### Security principals

A user, a group, a service principal (an app identity) or a managed identity. Prefer assigning to **groups**; membership is transitive for access, so a user gets what their groups have.

### Built-in roles you must know

| Role | What it allows |
| --- | --- |
| Owner | Everything, including assigning roles to others |
| Contributor | Create and manage all resources, but **cannot assign roles** |
| Reader | View only |
| User Access Administrator | Manage role assignments only, no resource changes |
| Virtual Machine Contributor, Storage Blob Data Reader, etc. | Narrow, service-specific roles |

A role definition lists **Actions** and **NotActions** (control plane) and **DataActions** (data plane, for example reading blob contents). Contributor on a storage account does not let you read blobs through Entra authorization; you need a data role.

### Scope and inheritance

Scopes form a hierarchy: **management group > subscription > resource group > resource**. A role assigned at a scope applies to everything below it.

- Assignments are **additive**: a user's permissions are the union of all their assignments.
- **Deny assignments** (created by Azure Blueprints/managed apps, not by you in the portal) override allows.
- Follow **least privilege**: smallest role, smallest scope.

### Entra roles vs Azure roles

Global Administrator manages the **directory**, not subscriptions. A Global Admin can use **Elevate access** (in Entra properties) to get User Access Administrator at root scope when locked out, which should be switched off afterwards. The old **classic administrator roles** (Co-Administrator, Account Administrator) were retired; use Azure RBAC only.

### Assigning a role

- Portal: resource group > **Access control (IAM)** > Add > Add role assignment.
- PowerShell: `New-AzRoleAssignment -SignInName ali@yourtenant.onmicrosoft.com -RoleDefinitionName "Reader" -ResourceGroupName rg-lab` (creates an assignment of Reader to that user at that resource group).
- CLI: `az role assignment create --assignee ali@yourtenant.onmicrosoft.com --role Reader --resource-group rg-lab` (the same thing).

### Interpreting Access control (IAM)

- **Check access**: pick a user and see their effective permissions at this scope.
- **Role assignments**: who has which role here, and whether it is *inherited* from a higher scope or assigned *this resource*.
- **Roles**: all role definitions, with the number of assignments.
- **Deny assignments**: any deny rules applying here.

### Custom roles

When no built-in role fits, you create a custom role (JSON with Actions, NotActions, AssignableScopes). Keep the built-ins first; know that custom roles exist and where they are defined.

**Lab link:** Sunday you assign roles at different scopes, verify with Check access, and repeat with PowerShell.

## Self-check questions

1. **Why does a license assignment fail for a new user?** Usage location is not set (or no licenses are left).
2. **Can nested groups receive a license through group-based licensing?** No, only direct members.
3. **What happens 31 days after deleting a user?** They were permanently removed after 30 days, so they can no longer be restored.
4. **Dynamic or assigned: members change automatically when the department attribute changes?** Dynamic.
5. **What is the `#EXT#` in a UPN?** Marks an external (guest) user.
6. **How many methods must admins use for SSPR?** Two, always.
7. **Contributor vs Owner?** Contributor cannot assign roles.
8. **A user is Reader on the subscription and Contributor on rg-lab. What can they do in rg-lab?** Contributor actions (additive; the union applies).
9. **Which role do you give someone who only manages access?** User Access Administrator.
10. **Global Administrator cannot see a subscription. Fix?** Use Elevate access, or be assigned an Azure role.

## Sources

- [Group-based licensing](https://learn.microsoft.com/microsoft-365/admin/manage/assign-licenses-to-users)
- [How SSPR works](https://learn.microsoft.com/entra/identity/authentication/concept-sspr-howitworks)
- [B2B collaboration overview](https://learn.microsoft.com/entra/external-id/what-is-b2b)
- [What is Azure RBAC?](https://learn.microsoft.com/azure/role-based-access-control/overview)
- [AZ-104 study guide](https://learn.microsoft.com/credentials/certifications/resources/study-guides/az-104)
