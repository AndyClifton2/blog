---
Author: "Andy Clifton"
title: "Azure Landing Zone verification with PowerShell (CAF Checklist)"
description: "Automatically check whether your Azure CAF implementation is complete and correct."
date: 2026-05-01
tags: ["Azure", "PowerShell", "CAF", "Terraform", "Governance"]
thumbnail: "/img/caf-thumbnail.png"
---

## Foreword

Many organisations choose to set up their Azure environment according to Microsoft's **Cloud Adoption Framework (CAF)**. That is a sensible choice: CAF offers a proven structure for governance, security, networking and identity management. But there is a catch.

The deployment itself, whether through Terraform, Bicep or by hand, is only the first step. The real question is: *is everything actually configured the way it should be?* Are all policies active? Are the right RBAC roles in place? Is the firewall working correctly? Have the DNS zones been created?

Clicking through the Azure Portal by hand to check all of this is time-consuming, error-prone and, above all, not repeatable. Every time you roll out an environment or push a change, you start over again.

This article describes an approach in which a PowerShell script takes over that verification role. The script automatically walks through 30+ checkpoints, spread across ten domains, and gives each check a clear ✅, ❌ or ⚠️ with an explanatory detail line. The names of the checks in this article refer to the CAF reference codes you will find in the script, so you can easily navigate between the blog and the script.

The script is set up generically: you adjust one configuration block at the top for your own environment, and the rest works regardless of your organisation's name or the specific resource names you use.


## <ins>How is the script structured?</ins>

The script consists of three layers: a configuration block, a set of helper functions, and the actual checks per domain.

**Configuration block**

At the top of the script is a block of variables that you adjust once per environment. Think of your Tenant ID, the Subscription IDs of your platform subscriptions, and the names of your resource groups. This is the only place you need to touch:

```powershell
$TenantId           = "<jouw-tenant-id>"
$ConnectivitySubId  = "<subscription-id-connectivity>"
$ManagementSubId    = "<subscription-id-management>"
$RootMgName         = "<naam-van-jouw-root-management-group>"
$HubResourceGroup   = "<naam-van-jouw-hub-resource-group>"
$DnsResourceGroup   = "<naam-van-jouw-dns-resource-group>"
```

**Helper functions**

The helper functions provide reusability and consistency throughout the script:

- `Write-CheckResult`: displays the result of each check (✅/❌/⚠️) including the reference code and details, and keeps the counters for the final summary
- `Find-ManagementGroup`: looks up Management Groups by both the technical `GroupId` and the `DisplayName` you see in the portal, so that naming differences do not produce false negatives
- `Get-AllMGs`: retrieves all Management Groups once and stores them in a cache, so the script does not call the same Azure API dozens of times
- `Test-PolicyAssignmentExists`: checks whether a specific policy is actually assigned at a given scope

**Pre-flight check**

Before the script runs a single check, it verifies that you are actively logged in to Azure. This prevents cryptic error messages halfway through:

```powershell
$context = Get-AzContext -ErrorAction SilentlyContinue
if (-not $context) {
    Write-Host "⚠️  Niet ingelogd bij Azure. Voer eerst Connect-AzAccount uit."
    exit 1
}
```

This sounds obvious, but in automated pipelines an expired token is one of the most common causes of silent failures.


## <ins>1. Governance & Management Groups</ins>

Management Groups are the backbone of every CAF implementation. They form the hierarchical structure on which you apply policies, RBAC roles and budgets, and they determine how far those settings "trickle down" to the subscriptions underneath.

Without the right Management Group structure, you cannot enforce consistent governance. Policies that you apply at root level automatically apply to all underlying groups and subscriptions. That is exactly the strength of this model, but also the reason why the structure has to be correct.

The CAF standard prescribes that you create at least the following groups under your root Management Group:

```
Tenant Root Group
└── [Jouw organisatienaam] (root MG)
    ├── Platform
    ├── Landing Zones
    ├── Sandbox
    └── Decommissioned
```

**Why this split?**

- **Platform** contains the subscriptions for shared infrastructure: networking, identity, security and monitoring. These are the resources that all workloads depend on.
- **Landing Zones** contains the subscriptions for the organisation's actual workloads, split per application or team.
- **Sandbox** is an isolated playground for experiments, explicitly decoupled from production. Peering with other environments is blocked here via policy.
- **Decommissioned** contains subscriptions that are being phased out. By moving them instead of deleting them straight away, you keep your audit trails intact.

<ins>G01: Verifying the MG structure</ins>

The script looks up all expected Management Groups by both `GroupId` and `DisplayName`. This distinction matters: the Azure portal sometimes shows a different name than the technical identifier used in the API.

```powershell
$mgChecks = [ordered]@{
    "Platform"      = Find-ManagementGroup -Name "Platform"
    "Landing Zones" = $(
        $lz = Find-ManagementGroup -Name "Landing Zones"
        if (-not $lz) { $lz = Find-ManagementGroup -Name "LandingZones" }
        $lz
    )
    "Sandbox"       = Find-ManagementGroup -Name "Sandbox"
    "Decommissioned"= Find-ManagementGroup -Name "Decommissioned"
}
```

<ins>G05: A maximum of 2 Owners on the Tenant Root Group</ins>

The Tenant Root Group is the highest level in your Azure tenant. Whoever is Owner here essentially has access to everything. Those should only be so-called *break-glass accounts*: emergency accounts that you only use when all other access has failed. The script checks that there are no more than two Owner assignments at this level.

<ins>G06: Audit-only policies at root level</ins>

The root Management Group should only contain *audit-only* policies, no Deny policies. The reason: Deny policies at the highest level can unexpectedly break workloads in subscriptions you do not yet have full visibility of. Deny policies belong at the level of the Landing Zones MG or lower, where you have full control over the impact.

<ins>G15 & G16: Key Vaults and Resource Locks on platform resources</ins>

Every platform subscription needs a Key Vault for managing secrets, certificates and keys. In addition, the script checks whether there are `CanNotDelete` locks on the critical platform resource groups. These locks prevent someone from accidentally, or with a wrong Terraform command, deleting the network or logging infrastructure that all workloads depend on.


## <ins>2. RBAC & Access Management</ins>

Role-Based Access Control (RBAC) determines who is allowed to do what in your Azure environment. A common mistake is to treat RBAC as a one-off configuration: something you set up during the initial deployment and never touch again. In practice, without active management, *privilege creep* quickly sets in: accounts that have more rights than they need, or rights that are permanent when they should be temporary.

The script checks three aspects of RBAC:

<ins>G08: Administrators on the root Management Group</ins>

On the root Management Group, Contributor roles must be assigned to the administrators of the environment. If they are missing, nobody has the right permissions to manage the platform infrastructure. The script checks whether at least one Contributor assignment exists at this level.

<ins>G31: Privileged Identity Management (PIM)</ins>

Admin roles should never be permanent. With Entra PIM (Privileged Identity Management) you make roles *eligible* instead of active: an administrator requests temporary access, provides a justification, and the role is activated for a limited time. This drastically reduces the attack surface: a compromised account has no admin rights without an active PIM activation.

The script uses `Get-AzRoleEligibleChildResource` to check whether PIM eligible assignments are configured. If the query fails because of insufficient permissions or an outdated module, the script reports a ⚠️ skip, not an incorrect ❌.

> **Note:** PIM requires an Entra ID P2 licence. If you do not have that licence, you run a real risk: permanent admin rights without a time limit or audit trail.

<ins>G34/35: Three standard RBAC groups per subscription</ins>

Three roles are expected on every subscription: Owner, Contributor and Reader. This sounds minimal, but it is the basic structure that makes layered access management possible. If one of the three is missing, the RBAC structure is incomplete and you have to assign access by hand, which is error-prone and hard to audit.

```powershell
$missing = @("Owner","Contributor","Reader") | Where-Object { $_ -notin $foundRoleNames }
```


## <ins>3. Azure Policy & Compliance</ins>

Azure Policy is the mechanism you use to enforce that resources comply with your organisational standards: automatically, at scale, without manual checks. The difference with RBAC is subtle but important: RBAC controls *who* may do something, Policy controls *what* is allowed to exist in the first place.

Think of examples such as: all resources must have a certain tag, public endpoints are not allowed, or VM disks must be encrypted. Without policies you rely on people making the right choices. With policies you enforce it.

<ins>G22: Microsoft Defender for Cloud</ins>

Defender for Cloud is the central security platform of Azure. It combines vulnerability management, threat protection and compliance monitoring in one dashboard. It is split into multiple plans per resource type, and any plan that is on the *Free* tier offers no active protection.

The script checks which plans are active and which are still running on Free:

```powershell
$defenderPlans = Get-AzSecurityPricing -ErrorAction SilentlyContinue
$activePlans   = $defenderPlans | Where-Object { $_.PricingTier -ne "Free" }
$g22ok         = ($activePlans).Count -gt 0
```

Enable at least the plans for `VirtualMachines`, `SqlServers`, `StorageAccounts` and `KeyVaults`. These are the resource types that attackers target most often.

<ins>G23: Custom Policy Initiatives</ins>

A *policy initiative* is a collection of multiple policies that you assign as a single package. The CAF standard prescribes that you create at least four initiatives:

- **Tagging**: enforce that all resources contain the required tags (for example `Environment`, `Owner`, `CostCenter`)
- **Landing Zone**: basic rules for workload subscriptions, such as prohibiting certain VM sizes or regions
- **Platform Connectivity**: rules for network configuration, such as prohibiting VPN gateways outside the connectivity subscription
- **Security**: baseline security rules, such as requiring HTTPS and encryption

The script checks whether these initiatives are present at the correct Management Group scope.

<ins>G18: Deny public endpoints on Landing Zones</ins>

On the Landing Zones Management Group, a *Deny* policy must be active that blocks the creation of public endpoints. Without this policy, a developer can create a Storage Account or Key Vault with public access, even though that is not the intention. The policy makes it impossible, not just undesirable.

<ins>G13: Log Analytics retention</ins>

Log data is worthless if you do not keep it long enough. For incident response and forensic investigation you need at least 30 days, preferably 90 days or more. The script checks the retention setting on all Log Analytics Workspaces and reports which ones are below the threshold.


## <ins>4. Connectivity & Networking</ins>

The network foundation of a CAF environment determines how traffic flows: between workloads, to on-premises, and to the internet. A wrong network design has direct consequences for security, not as theory, but as attack surface.

**Hub & Spoke or Virtual WAN?**

For most organisations, Hub & Spoke is the recommended network topology within CAF. In this model you manage a central hub VNet yourself, connect spoke VNets through VNet peering, and place shared services such as Azure Firewall, DNS and VPN Gateway in the hub. You have full control over the configuration, the costs are predictable, and the setup is easy to understand and manage, also for smaller teams.

Virtual WAN (vWAN) is an alternative that Microsoft offers for organisations with more complex requirements: multiple regions, large numbers of branch locations, or the wish to hand route management entirely over to the Azure platform. vWAN manages routing automatically and scales effortlessly, but comes with higher costs and less granular control over the network design.

The choice between Hub & Spoke and Virtual WAN depends on the scale, the management model and the budget of the organisation. There is no universally right answer, but there is a right answer for your situation.

In the case of this example, the customer deliberately chose Virtual WAN. The environment spans multiple locations and the customer did not want to maintain routing by hand. The script therefore specifically verifies the vWAN components. If you work with Hub & Spoke yourself, you will need to adjust the connectivity checks in the script to the relevant VNet and peering resources.

<ins>C02: Virtual WAN & Secured Hub</ins>

The script verifies that both the Virtual WAN resource and the Secured Hub are present in the Hub resource group. If the hub is missing, the rest of the network architecture has no foundation.

<ins>C05: Platform VNets</ins>

At least three platform VNets are expected:

- **Connectivity**: for the WAN connection and external connections
- **PrivateEndpoints**: for centrally hosting private endpoint DNS links
- **Management**: for management access and monitoring resources

The reason for separate VNets is *segmentation*: a problem in one VNet does not automatically affect the others.

<ins>C10: Azure Firewall Policy</ins>

An Azure Firewall without a Policy is a firewall that either allows everything or blocks everything: there are no rules. A Firewall Policy is the document in which you define the rules: which traffic is allowed, which is blocked, and what is logged. The script checks whether a policy is present in the Hub resource group.

<ins>C11: Blocking spoke-to-spoke peering</ins>

In a vWAN topology, all traffic flows through the hub. Direct peering between spokes bypasses the firewall and the central traffic policy. The script checks whether a policy is active that blocks spoke-to-spoke peering.

<ins>C12: Internet breakout via Azure Firewall</ins>

All outbound internet traffic must flow through the Azure Firewall in the hub. This is called a *forced internet breakout*. Without it, workloads can communicate directly with the internet, without inspection or logging. The script checks whether the Virtual WAN Hub, the Azure Firewall and Routing Intent are all present:

```powershell
$vhubs         = Get-AzVirtualHub -ResourceGroupName $HubResourceGroup
$afw           = Get-AzResource -ResourceType "Microsoft.Network/azureFirewalls" `
                     -ResourceGroupName $HubResourceGroup
$routingIntent = Get-AzResource `
                     -ResourceType "Microsoft.Network/virtualHubs/routingIntent"

$c12ok = ($vhubs).Count -gt 0 -and ($afw).Count -gt 0
```

Routing Intent is the link that instructs vWAN to send all traffic through the firewall. Without Routing Intent the firewall has rules, but never sees the traffic.


## <ins>5. DNS & Private Endpoints</ins>

DNS is the silent infrastructure that determines whether private connectivity is actually private. A common mistake: creating private endpoints but forgetting the corresponding DNS configuration. The result: resources that are reached via public DNS instead of via the private IP, so traffic unintentionally leaves the private network.

<ins>C07: DNS Private Resolver</ins>

In hybrid environments, where on-premises systems connect to Azure via ExpressRoute or VPN, a **DNS Private Resolver** is required. It takes care of *conditional forwarding*: DNS queries for `privatelink.*` domains are forwarded to Azure's private DNS, so that on-premises clients can resolve the private IP addresses of Azure resources.

If the Private Resolver is missing, on-premises systems simply cannot reach private endpoints.

<ins>C09a: Mandatory Private DNS Zones</ins>

For every Azure service that is offered through private endpoints, a corresponding Private DNS Zone must be present. The script checks at least four zones that appear in almost every environment:

```powershell
$requiredZones = @(
    "privatelink.blob.core.windows.net",
    "privatelink.file.core.windows.net",
    "privatelink.vaultcore.azure.net",
    "privatelink.database.windows.net"
)
$missingZones = $requiredZones | Where-Object { $dnsZonesCheck.Name -notcontains $_ }
```

If a zone is missing, the private endpoint for that service falls outside DNS resolution. Depending on the network configuration, that leads to one of two problems: the resource is unreachable, or the client falls back to public DNS and still makes a public connection.

<ins>C08: Private Endpoints on platform resources</ins>

The script also verifies whether platform resources themselves are reachable through private endpoints, not through public internet endpoints. This applies in particular to Key Vaults and Storage Accounts in the platform subscriptions.


## <ins>6. Security & Monitoring</ins>

An Azure environment without proper monitoring is an environment where you only find out afterwards that something went wrong. Security and observability are not separate disciplines: they are two sides of the same coin.

<ins>S01: Security policies via EPAC</ins>

**EPAC** stands for *Enterprise Azure Policy as Code*, a Microsoft open-source framework for managing Azure Policy as version-controlled code. The script checks whether EPAC-based security policies are active on the root Management Group. This includes at least audit policies for compliance and security baselines.

Why EPAC? Because manually managed policies in the portal are not auditable, not repeatable, and quickly drift out of sync with reality. Policy as code solves that.

<ins>IAM-3: User-assigned Managed Identities</ins>

Service principals with passwords are a risk: passwords expire, get shared, or leak. **Managed Identities** are the alternative: an Azure resource gets an identity that is automatically managed by the platform, without a password or secret. The script checks whether user-assigned Managed Identities are present as a signal of a mature identity strategy.

<ins>M03: Two Log Analytics Workspaces</ins>

Separating infrastructure logging from security logging is a deliberate architectural choice. Infrastructure logs contain performance and diagnostic data that can be widely accessible. Security logs contain sensitive information, such as login attempts, policy changes and privileged access, that should only be available to the security team. By keeping them in separate workspaces, you can manage access rights per workspace.

The script expects at least two workspaces in the Management subscription: one for infra, one for security (connected to Sentinel).

<ins>M09/10: AMBA Alert Rules</ins>

**AMBA** stands for *Azure Monitor Baseline Alerts*, a set of recommended thresholds for standard Azure resources published by Microsoft. Think of alerts on high CPU usage, queues piling up, Key Vault errors, or unexpected outages of VPN Gateways. The script checks both *Metric Alerts* (based on measured values) and *Activity Log Alerts* (based on actions, such as deleting a resource).

<ins>G14: Action Groups</ins>

An alert without an Action Group is a silent alarm. An Action Group defines *what happens* when an alert fires: send an email, call a webhook, start a runbook. The script checks whether at least one Action Group is configured. Without an Action Group, your alerts do not trigger anything.


## <ins>7. Landing Zones & Business Continuity</ins>

The previous sections were about the platform infrastructure: the shared foundation on which workloads run. This final section looks at the landing zones themselves: the environments where applications and services are actually hosted.

<ins>LZ02: Prod and Non-Prod per landing zone</ins>

Every application or team gets its own landing zone, with two Management Groups: Prod and Non-Prod. This makes it possible to apply different policies per environment. In Prod you want strict restrictions; in Non-Prod more room for experimentation. The script searches recursively through the MG hierarchy to verify that this split is present.

<ins>LZ-std: Standard resources per landing zone</ins>

Every landing zone must contain at least two resources:

- A **Log Analytics Workspace**, so that application logs of workloads in this landing zone are collected centrally
- A **Key Vault** for securely storing secrets, API keys and certificates that workloads in this landing zone need

By enforcing this as a standard, you prevent teams from putting secrets in code or sending logs nowhere.

<ins>B22: Zone-Redundant Storage (ZRS)</ins>

Storage Accounts running on LRS (Locally Redundant Storage) store data in a single datacenter. If that datacenter goes down, due to hardware or a zone issue, the data is temporarily unavailable. ZRS (Zone-Redundant Storage) replicates data across three availability zones within the same region, which eliminates this risk. The script checks all Storage Accounts and reports which ones are still running on LRS or GRS:

```powershell
$nonZrsSAs = $allSAs | Where-Object { $_.Sku.Name -notlike "*ZRS*" }
$b22ok     = ($allSAs).Count -gt 0 -and ($nonZrsSAs).Count -eq 0
```

<ins>B15-18: Resource Locks on critical resources</ins>

`CanNotDelete` locks on the DNS and Hub resource groups are a simple but effective measure. They prevent an administrator or an erroneous Terraform `destroy` command from deleting the network or DNS infrastructure that all workloads depend on. A lock costs nothing and potentially saves hours of recovery work.


## <ins>Running the script</ins>

The script has minimal requirements and runs on any machine where Azure PowerShell is available.

**Requirements:**

- Az PowerShell module: `Install-Module Az -Force`
- Azure CLI (optional, for budget checks): `winget install Microsoft.AzureCLI`
- Permissions: at least *Reader* at Management Group scope + *Security Reader* on the subscriptions

**Running it:**

```powershell
# Step 1: log in
Connect-AzAccount

# Step 2: adjust the configuration block at the top of the script for your environment

# Step 3: run
.\CAF.ps1
```

**What you will see:**

For each check, a line appears with the result, the reference code and a detail message:

```
  ✅  G01    Management group structuur aanwezig
       → Alle MG's gevonden: Platform | Landing Zones | Sandbox | Decommissioned

  ❌  G06    Audit-only policies op root MG
       → Geen audit-only policies gevonden op scope [jouw-root-mg]

  ⚠️  G31    Admin-rollen via PIM
       → Onvoldoende rechten voor PIM-query of Az.Resources module te oud (vereist 6.x+)
```

At the end, a summary appears:

```
╔══════════════════════════════════════════════════════════════╗
║                      SAMENVATTING                           ║
╠══════════════════════════════════════════════════════════════╣
║  ✅  Geslaagd    : 24  (80%)                                ║
║  ❌  Mislukt     : 4   (13%)                                ║
║  ⚠️   Overgeslagen: 2                                        ║
║  📋  Totaal      : 30                                        ║
╚══════════════════════════════════════════════════════════════╝
```

Red ❌ items always include a detail line (→) that states exactly what is missing or incorrect. Start there when resolving findings.


## Finally

CAF is not a goal in itself: it is a means to set up Azure in a manageable, secure and scalable way. Its value lies not in following the framework, but in consistently enforcing it.

That is exactly where this script helps. It does not verify whether you once did the right things, but whether they are *still* correct now. Configurations drift. People make changes. Terraform runs sometimes unintentionally overwrite existing settings. A verification script that you run after every deployment, or periodically as a scheduled task, gives you the certainty that reality matches the intention.

Anyone who takes Azure seriously should not only ask "Have we deployed everything?", but above all "Is it also correct, complete and demonstrably so?"

This script helps with that last question.

The full script is available on GitHub: [CAF.ps1 →](https://github.com/AndyClifton2/Scripts/tree/master/CAF)

## Azure governance is only effective when you actively verify it. Not once, but structurally.
