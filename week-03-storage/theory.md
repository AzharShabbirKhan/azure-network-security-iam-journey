# AZ-104 Week 3 Theory: Azure Storage

Week 3 covers how Azure stores data and who can reach it: storage accounts and redundancy (Monday), access control with firewalls, SAS tokens and keys (Wednesday), and Blob Storage with tiers, soft delete, versioning and lifecycle rules (Friday). Storage is 15 to 20% of the AZ-104 exam, shared with Week 4 (Azure Files, AzCopy, ARM templates and Bicep).

## The big picture

A storage account is the top-level container for your data. Almost every exam question in this area asks one of four things:

| Question | Feature to think of | Where it lives |
| --- | --- | --- |
| How many copies of my data exist, and where? | Redundancy: LRS, ZRS, GRS, GZRS and the read-access variants | Storage account setting |
| Who is allowed to reach the account from the network? | Storage firewall, virtual network rules, private endpoints | Storage account > Networking |
| Who is allowed to read or write the data, and with what credential? | Entra ID roles, account keys, SAS tokens | Authorization |
| How do I recover from deletes and control cost over time? | Soft delete, versioning, access tiers, lifecycle rules, object replication | Data protection and Data management |

Keep two ideas apart, because the exam mixes them on purpose:

- **Network access** decides whether a request can reach the account at all (firewall, private endpoint).
- **Authorization** decides whether a request that arrives is allowed to do what it asks (role, key, SAS).

A request must pass both. A perfect SAS token does not help a client whose IP address the firewall blocks.

**What you built in Week 2 still applies.** Azure Policy, locks and tags work on storage accounts like any other resource. A lock on a storage account protects the account itself from deletion. Soft delete and versioning protect the data inside it, and neither one protects the account from being deleted.

### Lab consequences for your tenant

- Your policies on `rg-lab-week02` require the `Owner` tag and only allow West Europe and North Europe. A storage account created in that group needs both, or the deployment is denied.
- West Europe rejected new virtual networks on your subscription with `RequestDisallowedByAzure`. Storage accounts may be rejected there too. Plan to use **North Europe** for the Week 3 lab, and create a new group `rg-lab-week03` so that your Week 2 policies do not get in the way unless you want them to.
- Storage account names are global. Pick names with a random suffix, for example `stlabazhar4821`.

## Monday: storage accounts, redundancy and encryption

### What a storage account is

A **storage account** is an Azure Resource Manager resource that gives you a unique namespace in Azure for your data. It must sit in a resource group and a region. Inside it you can have Blob containers, Azure Files shares, queues and tables. Every object has an address that includes your account name, for example `https://mystorageaccount.blob.core.windows.net`.

**Naming rules (exam favorite):**

- Storage account names are **3 to 24 characters**, **lowercase letters and numbers only**, and **globally unique** across Azure.
- Container names are 3 to 63 characters (lowercase letters, numbers and hyphens, starting with a letter or number, no consecutive hyphens).
- Blob names are 1 to 1,024 characters and are case-sensitive.

### Storage account types

| Type | Services | Redundancy options | Use it for |
| --- | --- | --- | --- |
| **Standard general-purpose v2** | Blob (including Data Lake Storage), Files, Queue, Table | LRS, GRS, RA-GRS, ZRS, GZRS, RA-GZRS | The default for most scenarios |
| **Premium block blobs** | Blob (block and append blobs) | LRS, ZRS | High transaction rates, small objects, consistently low latency |
| **Premium file shares** | Azure Files only | LRS, ZRS | High-performance file shares, and the type that supports NFS file shares |
| **Premium page blobs** | Page blobs only | LRS | VM disk-style workloads |

Key points:

- You **cannot change a storage account to a different type** after you create it. To move to another type, create a new account and copy the data.
- In PowerShell, CLI or Bicep, the type comes from `kind` (for example `StorageV2`) and the performance tier plus redundancy come from the SKU (for example `Standard_GRS`).
- Legacy types (general-purpose v1, Blob storage accounts) still exist. General-purpose v1 supports only LRS, GRS and RA-GRS, and you upgrade it to v2 before you can convert its redundancy.
- A **hierarchical namespace** turns an account into Azure Data Lake Storage. It changes which features are available. For example, blob versioning and object replication are not supported on accounts with it.

### Redundancy: how many copies, and where

Data in a storage account is always copied. The setting decides how far apart the copies are.

**In the primary region:**

- **LRS (locally redundant)** keeps three copies inside a single datacenter. It protects against drive, server and rack failures, but a fire or flood in that datacenter can lose everything. It is the cheapest option.
- **ZRS (zone-redundant)** copies data synchronously across **three or more availability zones** in the primary region. Reads and writes keep working if one zone fails. Microsoft recommends ZRS where you need high availability.

**To a secondary region (hundreds of miles away):**

- **GRS** = LRS in the primary region, then an **asynchronous** copy to the secondary region, which is again LRS.
- **GZRS** = ZRS in the primary region, then an asynchronous copy to the secondary region (LRS there).
- The secondary region is the **paired region**. It is fixed by the primary region and you cannot change it.
- With plain GRS and GZRS you **cannot read** the secondary copy unless a failover happens. **RA-GRS** and **RA-GZRS** give **read access** to the secondary at all times.

| Option | Durability over a year | Read availability |
| --- | --- | --- |
| LRS | at least 99.999999999% (11 nines) | at least 99.9% |
| ZRS | at least 99.9999999999% (12 nines) | at least 99.9% |
| GRS / RA-GRS | at least 99.99999999999999% (16 nines) | 99.9% for GRS, 99.99% for RA-GRS |
| GZRS / RA-GZRS | at least 99.99999999999999% (16 nines) | 99.9% for GZRS, 99.99% for RA-GZRS |

Where the numbers differ by access tier (for example cool or cold), Microsoft lists lower availability figures, so check the SLA page when a question asks for exact numbers.

**Three things the exam loves:**

1. **Replication to the secondary is asynchronous**, so a regional disaster can lose the most recent writes. The gap between the last write in the primary and the last one in the secondary is the **recovery point objective (RPO)**. Geo priority replication can bring that to 15 minutes or less for block blobs.
2. **Redundancy protects against hardware and regional failures, not against mistakes.** A deletion or an overwrite is applied to every copy at once. For mistakes you need soft delete, versioning and locks.
3. **Only geo-redundant options support failover.** LRS and ZRS cannot fail over to another region.

### Failover

If the primary region becomes unavailable, you can fail over to the secondary. Azure updates the DNS entries of your storage endpoints so the secondary region's endpoints become the new primary ones, and then clients can write there again. After a customer-managed failover of a GRS or RA-GRS account, the account becomes **LRS in the new primary region**. You cannot convert that account straight to ZRS or GZRS. Add zone redundancy by a manual migration.

### Changing redundancy later

You can change redundancy on an existing account:

- **Customer-initiated conversion** (portal: Storage account > Data management > Redundancy; or `Start-AzStorageAccountMigration -TargetSku`, or `az storage account migration start --sku`). A conversion usually starts within **72 hours** and has **no completion SLA**. You get no downtime for most accounts and no data loss.
- After a zone-redundancy conversion, wait **at least 24 hours** before changing the setting again. Going from LRS to GZRS is therefore a two-step process.
- **Adding geo-redundancy** incurs a one-time egress charge.
- **Archive-tier blobs** make changes expensive. The archive tier is supported only for LRS, GRS and RA-GRS, so moving to ZRS or GZRS means you must rehydrate archived blobs first. Microsoft recommends a manual migration in that case.
- If you need control over exactly when the change finishes, use a **manual migration**, which costs downtime.

### Encryption

All data in a storage account is **encrypted at rest** (AES-256) and you cannot turn this off. The question is who manages the key:

| Key option | Who holds and rotates the key | Where it lives | Services |
| --- | --- | --- | --- |
| **Microsoft-managed keys** (default) | Microsoft | Microsoft key store | All services |
| **Customer-managed keys (CMK)** | You | **Azure Key Vault** or **Key Vault Managed HSM** | Blob Storage and Azure Files |
| **Customer-provided keys** | You | Your own store; you send the key with each request | Blob Storage only |

Details for customer-managed keys:

- The key vault must have **soft delete and purge protection enabled**. The key must be an **RSA or RSA-HSM** key of size 2048, 3072 or 4096.
- The storage account uses a **managed identity** (system-assigned or user-assigned) to reach the key vault. When you set a CMK while creating the account, you must use a user-assigned identity.
- Storage wraps the account's data encryption key with your key. Protection takes effect immediately and data stays encrypted throughout. You can switch between customer-managed and Microsoft-managed keys at any time.
- The key vault can sit in another tenant, region or subscription.
- **Encryption scopes** let you apply a different key to a container or a single blob, which creates boundaries between data of different customers in one account.

### Object replication

**Object replication** copies **block blobs** asynchronously from a container in one storage account to a container in another account. Typical uses are lower read latency near users, compute workloads that read a local copy, and data distribution.

How it works:

- You create a **replication policy** with one or more **rules**. Each rule maps a source container to a destination container, optionally filtered by blob name prefix.
- You create the policy on the **destination** account first. Azure gives it a policy ID, and you attach the policy to the source account using that ID.
- A source account can replicate to **no more than two** destination accounts. A policy can hold up to **1,000 rules**.
- The accounts can be in the same or different regions, subscriptions, or even tenants.
- By default only **new** blobs written after the rule exists are copied. You can choose to copy existing blobs too, or set a custom start time.

Prerequisites and limits:

- **Blob versioning must be enabled on both accounts and change feed must be enabled on the source account.** This is the classic exam question.
- Both accounts must be general-purpose v2 or premium block blob accounts.
- You need the **Contributor** role (or higher) on the storage account scope to configure it.
- Object replication does **not** replicate snapshots, and it does not work with blobs moved to the **archive** tier. It does not work on accounts with a hierarchical namespace.
- While a policy exists, you **cannot disable versioning** on the account. Delete the policy first.
- It is asynchronous, so the two accounts are not instantly in sync. It adds transaction, storage and egress costs.

**Lab link:** Saturday you create a GRS account and a ZRS account and compare them, and Sunday you set up object replication between two accounts.

## Wednesday: access to storage

### Two layers of defense

1. **Network layer:** can the request reach the account? (firewall, virtual network rules, private endpoints)
2. **Authorization layer:** is the caller allowed to do this? (Entra ID, account keys, SAS)

### Storage firewall and network rules

A new storage account accepts connections from any network by default. The firewall controls access to the account's **public endpoint**:

| Rule type | What it allows |
| --- | --- |
| **Virtual network rules** | Traffic from a specific subnet in an Azure virtual network. The subnet needs the Azure Storage **service endpoint** enabled |
| **IP network rules** | Specific public IPv4 addresses or CIDR ranges, for internet clients and on-premises networks |
| **Resource instance rules** | A named Azure resource (and its managed identity) that cannot be isolated by network rules |
| **Trusted service exceptions** | Specific Azure services on the trusted list that run outside your network boundary |

Rules worth knowing:

- **Network rules have no effect unless the default action is Deny.** In the portal this is "Enabled from selected virtual networks and IP addresses". Grant your own access before you switch, or you lock yourself out.
- Do not enter private RFC 1918 addresses in IP rules. They are for public addresses. A storage account supports up to **400 IP network rules**.
- Once a subnet uses a service endpoint, its traffic uses the subnet's private identity, so an older IP rule for the workload's public IP no longer applies.
- Firewall changes can take **up to a minute** to take effect.
- Access from **trusted Azure services** takes the **highest precedence** over other network restrictions. If you later set public network access to Disabled, previously configured resource instance rules and the trusted-services exception **stay in effect**.
- Requests from an allowed network still need valid authorization.
- In practice, when you turn on the firewall, add your own public IP address first, or the portal cannot show you the data.

**Public network access** has three states:

- **Enabled from all networks** (default).
- **Enabled from selected virtual networks and IP addresses** (firewall on, default action Deny).
- **Disabled**: traffic is accepted only through **private endpoints**.

### Service endpoint versus private endpoint

| | Service endpoint | Private endpoint |
| --- | --- | --- |
| What you get | Traffic from your subnet to the storage public endpoint goes over the Azure backbone, and the storage firewall can allow that subnet | A **private IP address from your virtual network** is assigned to the storage account |
| Public endpoint | Still exists and is still reachable by anyone the firewall allows | Can be fully turned off (public network access Disabled) |
| Reachable from on-premises | Only through public IP rules | Yes, through VPN or ExpressRoute, using the private IP |
| Needs DNS work | No | Yes, a private DNS zone such as `privatelink.blob.core.windows.net` |

`Microsoft.Storage` is the regional service endpoint. `Microsoft.Storage.Global` reaches storage accounts in any region. A subnet cannot use both at once. Week 7 covers private endpoints in depth.

### Authorization options

| Method | Credential | Notes |
| --- | --- | --- |
| **Microsoft Entra ID** with Azure RBAC | A user, group, service principal or managed identity | **Recommended.** Data roles such as Storage Blob Data Reader and Storage Blob Data Contributor |
| **Shared Key** | One of the two account access keys | Full access to everything in the account. Avoid it |
| **Shared access signature (SAS)** | A signed token appended to a URL | Delegated, limited, time-bound access |
| **Anonymous access** | None | Only works if the account and container are configured for anonymous read access |

Remember the Week 1 point: a control-plane role such as Contributor on a storage account does not grant access to the data. You need a data role (a `DataActions` permission), for example Storage Blob Data Reader.

### Shared access signatures (SAS)

A SAS is a token appended to the URI of a storage resource. It states which **resource**, which **permissions**, **how long** it is valid, and optionally an allowed **IP range** and **protocol** (HTTPS). Every SAS is signed with a key.

There are three types:

| Type | Signed with | Scope | Notes |
| --- | --- | --- | --- |
| **User delegation SAS** | **Entra ID credentials** (a user delegation key) | Microsoft recommends it. Blob Storage for sure; the current overview also lists Queue, Table and Files | **Most secure.** Not affected by account key rotation. Cannot use a stored access policy |
| **Service SAS** | Storage account key | One service: Blob, Queue, Table or Files | Can reference a **stored access policy** |
| **Account SAS** | Storage account key | One or more services, plus service-level operations | Cannot use a stored access policy |

To create a user delegation SAS, the caller needs a role that includes the `Microsoft.Storage/storageAccounts/blobServices/generateUserDelegationKey` action.

**Ad hoc SAS or stored access policy:**

- **Ad hoc SAS:** start time, expiry and permissions are all written in the token. To end it early you must rotate the key that signed it.
- **Service SAS with a stored access policy:** the policy lives on the **container** (or file share, queue or table) and holds the start time, expiry and permissions. The SAS inherits them. You can **change or revoke** every SAS tied to the policy by editing or **deleting the policy**. This is exactly what your Saturday lab practices.

**Three facts that get tested:**

1. You cannot audit the creation of SAS tokens. Anyone allowed to use an account key, or holding a role that lets them generate a SAS, can do so without the account owner knowing.
2. A SAS is as sensitive as a key. Use HTTPS only, and keep lifetimes short.
3. **Rotating an account key revokes every account SAS and service SAS signed with it**, which is a blunt way of revoking. A user delegation SAS is not affected by key rotation.

### Account access keys

- Every storage account has **two access keys** (key1 and key2) that give full access to all the data and let anyone generate a SAS.
- Two keys exist so that you can **rotate without downtime**: switch the application to key2, regenerate key1, then repeat.
- Regenerating a key breaks every application and service that uses it. Storage Explorer and connection strings are examples. Update them first.
- The built-in roles that may rotate keys are **Owner**, **Contributor** and **Storage Account Key Operator Service Role**.
- **Key expiration policy:** in the portal, Access keys > **Set rotation reminder**. It does not rotate anything by itself. It shows a reminder when the interval has passed. The button is grayed out until each key has been rotated at least once on older accounts.
- The built-in Azure Policy **Storage account keys should not be expired** reports accounts whose keys are overdue.
- **Disable Shared Key access** (portal: Configuration > **Allow storage account key access: Disabled**, or `Set-AzStorageAccount -AllowSharedKeyAccess $false`). Requests that use Shared Key then fail with **403**. This also stops account and service SAS tokens from working, so use user delegation SAS and Entra ID instead. To use Entra Conditional Access on a storage account, you must disallow Shared Key.
- Best practice: keep keys in **Azure Key Vault** and rotate them there.

### Putting Wednesday together

| Requirement | Answer |
| --- | --- |
| Only a specific subnet may use the account | Virtual network rule plus a service endpoint, and default action Deny |
| No traffic from the internet at all | Private endpoint and public network access Disabled |
| Give a contractor read access to one container for 48 hours | User delegation SAS, or a service SAS with a stored access policy |
| Cut off a SAS you already handed out | Delete the stored access policy; if there was no policy, rotate the key |
| Applications must stop using the account keys | Disable Shared Key access, use Entra ID and managed identities |
| Rotate keys without downtime | Move the app to key2, regenerate key1, move back |

**Lab link:** Saturday you restrict an account to a virtual network and your IP, create a SAS and a stored access policy, revoke by deleting the policy, and rotate a key.

## Friday: Blob Storage

### The model

Blob Storage has three levels: the **storage account**, a **container** inside it, and a **blob** inside a container. A container can hold an unlimited number of blobs, and an account can have unlimited containers. The structure is flat. A "folder" is just a delimiter such as `/` inside the blob name, unless the account has a hierarchical namespace.

Three blob types exist: **block blobs** (documents, media, most data), **append blobs** (logs) and **page blobs** (VM disks). **Access tiers can only be set on block blobs.**

### Access tiers

| | Hot | Cool | Cold | Archive |
| --- | --- | --- | --- | --- |
| Storage cost | Highest | Lower | Lower still | Lowest |
| Access cost | Lowest | Higher | Higher | Highest |
| Recommended minimum retention | None | **30 days** | **90 days** | **180 days** |
| Latency | Milliseconds | Milliseconds | Milliseconds | **Hours** (offline) |
| Redundancy supported | All | All | All | **LRS, GRS and RA-GRS only** |

- Data moved or deleted **before the minimum period** is charged an **early deletion fee** for the remaining days. Example: a blob moved to archive and deleted after 45 days is charged for 135 more days.
- The **default account tier** (hot or cool) applies to blobs that do not have their own tier.
- **Smart tier** is a newer option that moves data between hot, cool and cold automatically based on usage.
- **Archive is offline.** You cannot read or modify an archived blob. You must **rehydrate** it to hot, cool or cold first, either by changing the tier with **Set Blob Tier** or by copying it to a new blob in an online tier.
- **Rehydration priority:** *Standard* can take up to **15 hours**. *High* is prioritized and may finish in under an hour for objects under 10 GB.
- Metadata and blob index tags of an archived blob stay readable, so you can still list it.
- The archive tier is not supported for ZRS, GZRS or RA-GZRS accounts.

### Soft delete

Soft delete keeps deleted data recoverable for a **retention period** instead of removing it immediately.

| | Blob soft delete | Container soft delete |
| --- | --- | --- |
| Protects | An individual blob, its versions, snapshots and metadata | A whole container and everything in it |
| Retention | **1 to 365 days** (Microsoft recommends at least 7) | **1 to 365 days**, default **7** |
| Recover with | Undelete Blob | **Restore Container** |
| Where in the portal | Storage account > Data management > **Data protection** | Same page |

Important details:

- Blob soft delete is **on by default** for new accounts created in the portal. It is not on by default for accounts created with PowerShell or the CLI.
- Container soft delete can restore only **whole containers**. To restore one blob when the container still exists, you need blob soft delete or versioning. You must restore the container **under its original name**. If someone has created a new container with that name, restore is impossible.
- A new retention period applies only to data deleted **after** the change.
- The least privileged built-in role to restore a soft-deleted container is **Storage Blob Data Contributor**.
- **Soft delete does not protect the storage account itself.** Deleting the account loses everything, so use a **resource lock** on the account.

### Blob versioning

With versioning enabled, Azure Storage creates a **new version with a unique version ID** when you first create a blob and **every time you modify it**. Version IDs are timestamps.

- Modify a blob: the current version becomes a **previous version** and a new current version is created.
- Delete a blob without a version ID: the current version becomes a previous version and there is no current version. All previous versions remain.
- Previous versions are **immutable**. You restore by promoting a previous version (copy it over the current one).
- Versions are kept **until you delete them**, so cost grows. Microsoft recommends staying under **1,000 versions per blob**, because listing slows down. A lifecycle rule can delete old versions.
- Disabling versioning does not delete existing versions. Future changes just overwrite without saving the previous state.
- It is not supported on accounts with a hierarchical namespace, and it cannot be disabled while an object replication policy exists.
- Versioning cannot recover from deleting the storage account or a container.

### Soft delete or versioning?

| Need | Use |
| --- | --- |
| Recover a blob deleted by mistake for a limited time | Blob soft delete |
| A complete history of every change, including overwrites | Versioning |
| Recover a whole deleted container | Container soft delete |
| Protect the account from deletion | A resource lock |
| Both overwrite and delete protection (recommended) | Enable **soft delete and versioning together** |

When both are on, overwriting a blob creates a previous version that is **not** soft-deleted. Deleting a previous version soft-deletes that version, and Undelete Blob restores **all** soft-deleted versions of the blob at once.

### Lifecycle management

A **lifecycle management policy** is a JSON document with rules that move blobs to cheaper tiers or delete them automatically.

A rule has:

- **Filters:** blob type (for example `blockBlob`), name **prefix** match, and optionally blob index tags.
- **Actions** on three targets: `baseBlob` (the **current version**), `snapshot` and `version` (previous versions).
- **Conditions** based on age: `daysAfterModificationGreaterThan`, `daysAfterCreationGreaterThan`, `daysAfterLastAccessTimeGreaterThan` (needs **last access time tracking** turned on) and `daysAfterLastTierChangeGreaterThan`.

Actions: `tierToCool`, `tierToCold`, `tierToArchive`, `enableAutoTierToHotFromCool` and `delete`.

A sample rule that moves block blobs with the prefix `logs` to cool after 30 days and deletes them after 365:

```json
{
  "rules": [
    {
      "name": "logs-rule",
      "enabled": true,
      "type": "Lifecycle",
      "definition": {
        "filters": { "blobTypes": [ "blockBlob" ], "prefixMatch": [ "logs" ] },
        "actions": {
          "baseBlob": {
            "tierToCool": { "daysAfterModificationGreaterThan": 30 },
            "delete": { "daysAfterModificationGreaterThan": 365 }
          }
        }
      }
    }
  ]
}
```

Facts to remember:

- The policy **runs once a day**. A new or changed rule can take up to **24 hours** to apply, and on very large accounts a run can take several days.
- If several actions match the same blob, the **least expensive action wins**: delete beats archive, which beats cool.
- `tierToCool` and `tierToCold` are not supported on append blobs, page blobs or premium block blob accounts. `tierToArchive` is also not supported on ZRS or GZRS accounts, or on blobs that use an encryption scope.
- Rehydrating a blob does not update its last-modified time, so a rule might move it straight back to archive. Add `daysAfterLastTierChangeGreaterThan` to the archive action to prevent that.
- `enableAutoTierToHotFromCool` moves a blob back to hot when accessed, once per 30 days at most, and works only with the last-access-time condition.
- **Interaction with soft delete:** a lifecycle rule that deletes a blob in an account with soft delete puts the blob in a soft-deleted state, and it stays for the retention period.
- You can create policies in the portal (Data management > **Lifecycle management**, List view or Code view), in PowerShell with `Set-AzStorageAccountManagementPolicy` and in the CLI.

**Lab link:** Sunday you upload blobs, change tiers, write the 30-day cool rule, enable soft delete and versioning, delete and recover a blob, and configure object replication.

## Lab scripts you will use

Explanations first, so you understand each command before you run it. Replace names with your own. Create the resource group in North Europe with the tag your policies require.

```powershell
New-AzResourceGroup -Name rg-lab-week03 -Location northeurope -Tag @{Owner="azhar"}
```
Creates a new resource group for this week.

```powershell
New-AzStorageAccount -ResourceGroupName rg-lab-week03 -Name stlabgrs4821 -Location northeurope -SkuName Standard_GRS -Kind StorageV2 -Tag @{Owner="azhar"}
```
Creates a general-purpose v2 account with GRS. For the ZRS comparison, change the SKU to `Standard_ZRS` and the name to something else. `-Kind StorageV2` is the account type, and `-SkuName` combines performance tier and redundancy.

```powershell
Update-AzStorageAccountNetworkRuleSet -ResourceGroupName rg-lab-week03 -Name stlabgrs4821 -DefaultAction Deny
Add-AzStorageAccountNetworkRule -ResourceGroupName rg-lab-week03 -AccountName stlabgrs4821 -IPAddressOrRange "<your-public-ip>"
```
The first command turns the firewall on by setting the default action to Deny. The second allows your own public IP. Add the IP first if you can, so you do not lose access.

```powershell
Enable-AzStorageBlobDeleteRetentionPolicy -ResourceGroupName rg-lab-week03 -StorageAccountName stlabgrs4821 -RetentionDays 7
Enable-AzStorageContainerDeleteRetentionPolicy -ResourceGroupName rg-lab-week03 -StorageAccountName stlabgrs4821 -RetentionDays 7
```
Turns on blob soft delete and container soft delete with seven days of retention.

```powershell
Set-AzStorageAccount -ResourceGroupName rg-lab-week03 -AccountName stlabgrs4821 -KeyExpirationPeriodInDay 90
Set-AzStorageAccount -ResourceGroupName rg-lab-week03 -AccountName stlabgrs4821 -AllowSharedKeyAccess $false
```
The first sets a key expiration policy of 90 days (the account's keys may need rotating once before this works). The second disables Shared Key authorization. Do the second one last in the lab, because it also stops key-signed tokens.

```azurecli
az storage account blob-service-properties update --resource-group rg-lab-week03 --account-name <source-account> --enable-versioning --enable-change-feed
az storage account blob-service-properties update --resource-group rg-lab-week03 --account-name <destination-account> --enable-versioning
```
The two prerequisites for object replication: versioning on both accounts, change feed on the source.

**Commands I am giving from memory, not from a Microsoft Learn page I opened:** key regeneration (`az storage account keys renew --account-name <name> --key primary`, or `New-AzStorageAccountKey -KeyName key1`) and SAS creation (`New-AzStorageContainerSASToken`, `az storage container generate-sas`). Check each with `Get-Help` or `--help` before you run it.

## How the controls combine

| Scenario | Answer |
| --- | --- |
| Data must survive the loss of an entire datacenter, in one region | ZRS |
| Data must survive the loss of a whole region and be readable during the outage | RA-GRS or RA-GZRS |
| Data must survive a zone failure and a region failure | GZRS (RA-GZRS if you need reads from the secondary) |
| Cheapest option, data easily recreated | LRS |
| Archive tier is required | LRS, GRS or RA-GRS (not ZRS, GZRS, RA-GZRS) |
| Keep data for 7 years, rarely read | Archive tier, usually through a lifecycle rule |
| Block a stolen account key from being used | Disable Shared Key access |
| A user deleted an important blob 3 days ago | Blob soft delete (if retention is long enough) or a previous version |
| Someone deleted a whole container | Container soft delete and Restore Container, under the original name |
| Someone deleted the storage account | Not recoverable by soft delete or versioning. Prevent with a lock |
| Copy blobs to another account automatically | Object replication (versioning on both, change feed on the source) |

## Self-check questions

1. **Which redundancy option keeps three synchronous copies across availability zones in one region?** ZRS.
2. **Is data in the secondary region readable with GRS?** No, only after a failover. RA-GRS and RA-GZRS give read access at all times.
3. **How is data copied to the secondary region?** Asynchronously, so recent writes can be lost (the RPO).
4. **What happens to a GRS account after a customer-managed failover?** It becomes LRS in the new primary region.
5. **Which key options can a storage account use for encryption at rest?** Microsoft-managed (default), customer-managed in Key Vault or Managed HSM, and customer-provided keys on Blob requests.
6. **What must be enabled for object replication?** Blob versioning on both accounts and change feed on the source.
7. **Why would a firewall rule not take effect?** The default action is still Allow. Network rules work only with the default action set to Deny.
8. **Which SAS type is most secure, and why?** The user delegation SAS, because it is signed with Entra credentials instead of an account key, and key rotation does not affect it.
9. **How do you revoke a service SAS that references a stored access policy?** Delete or edit the stored access policy.
10. **How do you revoke an ad hoc account SAS?** Rotate the account key that signed it.
11. **Why does a storage account have two keys?** To rotate keys without downtime.
12. **A blob moved to archive is deleted after 45 days. What happens?** An early deletion fee for the remaining 135 days.
13. **Which tiers support millisecond latency?** Hot, cool and cold. Archive takes hours.
14. **How long can a standard rehydration from archive take?** Up to 15 hours.
15. **Soft delete is on and a lifecycle rule deletes a blob. Is it gone?** No. It becomes soft-deleted and stays for the retention period.
16. **Which feature protects against someone deleting the storage account?** A resource lock, not soft delete or versioning.
17. **Two lifecycle actions match a blob: tierToCool and delete. Which runs?** Delete, the least expensive action.
18. **A user has Contributor on a storage account but cannot read blobs with their Entra account. Why?** Contributor is a control-plane role. They need a data role such as Storage Blob Data Reader.

## Common exam traps

- Redundancy protects against failures, not against deleting or overwriting data.
- ZRS, GZRS and RA-GZRS cannot use the archive tier.
- Storage account **names** are 3 to 24 characters, lowercase and numbers, globally unique.
- Account type cannot be changed after creation.
- Soft delete retention is 1 to 365 days. Container soft delete restores only whole containers.
- Disabling Shared Key also stops account and service SAS tokens.
- A firewall rule set does nothing until the default action is Deny.
- A rotation reminder does not rotate anything.

## Sources

- [Azure Storage redundancy](https://learn.microsoft.com/azure/storage/common/storage-redundancy)
- [Change how a storage account is replicated](https://learn.microsoft.com/azure/storage/common/redundancy-migration)
- [Overview of storage accounts](https://learn.microsoft.com/azure/storage/common/storage-account-overview)
- [Azure Storage encryption for data at rest](https://learn.microsoft.com/azure/storage/common/storage-service-encryption)
- [Customer-managed keys for Azure Storage encryption](https://learn.microsoft.com/azure/storage/common/customer-managed-keys-overview)
- [Object replication for block blobs](https://learn.microsoft.com/azure/storage/blobs/object-replication-overview)
- [Azure Storage firewall rules and network access control](https://learn.microsoft.com/azure/storage/common/storage-network-security)
- [Grant limited access with shared access signatures (SAS)](https://learn.microsoft.com/azure/storage/common/storage-sas-overview)
- [Define a stored access policy](https://learn.microsoft.com/rest/api/storageservices/define-stored-access-policy)
- [Manage storage account access keys](https://learn.microsoft.com/azure/storage/common/storage-account-keys-manage)
- [Prevent Shared Key authorization](https://learn.microsoft.com/azure/storage/common/shared-key-authorization-prevent)
- [Access tiers for blob data](https://learn.microsoft.com/azure/storage/blobs/access-tiers-overview)
- [Soft delete for containers](https://learn.microsoft.com/azure/storage/blobs/soft-delete-container-overview)
- [Soft delete for blobs](https://learn.microsoft.com/azure/storage/blobs/soft-delete-blob-overview)
- [Blob versioning](https://learn.microsoft.com/azure/storage/blobs/versioning-overview)
- [Lifecycle management policy structure](https://learn.microsoft.com/azure/storage/blobs/lifecycle-management-policy-structure)
- [Naming rules and restrictions for Azure resources](https://learn.microsoft.com/azure/azure-resource-manager/management/resource-name-rules)

The facts in this document were checked against these Microsoft Learn pages in October 2026. Microsoft changes limits and features often, so recheck the pages before you sit the exam. The items marked "from memory" in the lab scripts section and the "in practice" notes were not checked against a specific page.
