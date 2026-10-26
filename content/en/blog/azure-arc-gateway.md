---
title: "Azure Arc gateway: one endpoint for all your hybrid traffic"
date: 2026-09-30
author: Andy Clifton
tags: [Azure, Azure Arc, Hybride cloud, Netwerk, Azure Local]
thumbnail: "/img/gateway.jpg"
description: "The Azure Arc gateway reduces dozens of firewall endpoints to seven, with a single gateway URL for all Arc traffic. How it works, what changed in 2026 (GA for servers, Kubernetes and Azure Local, plus the bypass list), how to set it up and where its limits are."
---

# Azure Arc gateway: one endpoint for all your hybrid traffic

![Azure Arc gateway: dozens of servers talking to Azure through a single gateway](/Images/arcgateway/arc-gateway-hero.png)

## Why network teams sigh at Azure Arc

Anyone who has ever guided an Azure Arc onboarding at an organisation with a strict network policy knows the conversation. You hand over the list of required endpoints and the network team looks at you as if you just asked whether the firewall could be opened up completely. Dozens of FQDNs, some with wildcards, some region dependent, and every new extension adds a few more. Azure Monitor Agent, Update Manager, Defender, Machine Configuration, Key Vault sync: they all talk to their own set of endpoints.

The result is predictable. The change takes weeks, typos creep into the allowlist, and six months later you discover that an extension has quietly stopped working because Microsoft added an endpoint nobody noticed. At government organisations and water authorities, where every outbound connection has to be justified, this is often the biggest delaying factor in a hybrid cloud project. Not the technology, but the allowlist.

![Without the gateway every server goes through the firewall to dozens of endpoints; with the gateway the Arc proxy bundles everything into a single rule next to six unchanged base endpoints](/Images/arcgateway/arc-gateway-voor-na.png)

The Azure Arc gateway is Microsoft's answer to this. Instead of dozens of endpoints you open seven, one of which is your own gateway URL through which almost all Arc traffic flows. In this blog I explain how the gateway works, what has changed over the past year, how to set it up, and above all: where its limits are. Because "one endpoint" sounds great, but it is not the whole story.

## How the Arc gateway works

The Arc gateway consists of two parts that work together. On the Azure side there is the **gateway resource**, a management object of type `Microsoft.HybridCompute/gateways`. When you create it, it gets a unique URL in the form `<prefix>.gw.arc.azure.com`. This URL is a front end based on Azure Front Door and acts as the shared front door for Arc traffic. On the server side there is the **Arc proxy**, a component that is already part of the Connected Machine agent and that behaves as a forward proxy for the agent itself and for all extensions. You do not need to install or configure that proxy separately; as soon as a server is associated with a gateway, the agent starts using it.

The traffic flow then becomes: Arc agent and extensions, to the local Arc proxy on `localhost:40343`, optionally via your own enterprise proxy, to the Arc gateway, and from there to the actual Azure service. To your firewall it looks as if the server only talks to a handful of addresses, while behind the gateway the dozens of Azure endpoints still exist.


An important detail for security teams is how TLS fits into this. The Arc proxy sets up a TLS session to the gateway and sends a nested HTTP CONNECT to the target endpoint inside it. If that target also uses TLS, which is almost always the case, a second, end to end encrypted session is established inside the first one. A TLS inspecting proxy does see that nested CONNECT, but cannot break open the inner session without nested TLS termination, and no standard proxy can do that. Microsoft therefore explicitly recommends excluding `<prefix>.gw.arc.azure.com` from TLS inspection. If your policy says that all outbound traffic must be inspected, you will need to make a deliberate exception here.

![Nested TLS: the outer session runs from the Arc proxy to the gateway URL, the inner end to end session to the Azure endpoint cannot be opened by an inspecting proxy](/Images/arcgateway/arc-gateway-geneste-tls.png)

The region you choose when creating the gateway only determines where the resource lives in the control plane. The gateway itself is a global service; a server in the Netherlands talking to a gateway in West Europe does not take a longer route than necessary.

## What has changed over the past year

The Arc gateway is not new; the public preview dates back to late 2024. What is new is that it has matured into a product you can deploy in production at customers for all three Arc resource types. Within a year the gateway went from preview to GA and gained a number of features that make the difference in practice.

![Timeline from public preview in November 2024 to the bypass list in agent 1.64 in May 2026](/Images/arcgateway/arc-gateway-tijdlijn.png)

| When | What | Status |
| --- | --- | --- |
| July 2026 (agent 1.66) | Arc proxy correctly routes entries in the gateway bypass list around the gateway | GA |
| May 2026 (agent 1.64) | Gateway bypass list: FQDNs you specify yourself go through your enterprise proxy or directly, instead of through the gateway | GA |
| March 2026 | Arc gateway GA for Arc enabled Kubernetes | GA |
| October 2025 (agent 1.58) | Preview flag removed from `connection.type` | GA |
| September 2025 | Arc gateway GA for Arc enabled servers and for Azure Local | GA |

For me, the bypass list is the most important addition. In the first GA version it was all or nothing: if a server was associated with the gateway, all Arc traffic went through it, and the agent's classic `proxy.bypass` setting no longer worked. That was awkward for organisations that deliberately want certain services to go through their own proxy, for example because logging or DLP is applied there, or because an endpoint is served internally through a private endpoint. Since agent 1.64 you can name those exceptions. I will come back to that further on.

For Azure Local there is one important limitation you need to know up front: the gateway can only be enabled on a new deployment (version 2506 or higher). Putting an existing Azure Local instance on the gateway afterwards is not supported. For servers and Kubernetes clusters it is.

## Hands on: creating a gateway and associating servers

You need the **Azure Arc gateway manager** role on the subscription or resource group where the gateway will live. Creating it is a single command and takes about ten minutes. I prefer to put the gateway in the connectivity or management resource group of the landing zone, next to the other shared network components.

```powershell
New-AzConnectedMachineGateway -ResourceGroupName "rg-hybrid-prd-weu-01" `
    -Location "westeurope" `
    -Name "arcgw-prd-weu-01"
```

Or with the Azure CLI:

```bash
az arcgateway create --resource-group rg-hybrid-prd-weu-01 \
    --location westeurope \
    --name arcgw-prd-weu-01

az arcgateway list --query "[].{name:name, url:gatewayEndpoint}" -o table
```

The second command gives you the gateway URL, the `<prefix>.gw.arc.azure.com` that the network team needs to add to the allowlist. Do that before you associate the first server, otherwise the agent will sit there waiting for the gateway and you lose the connection.

### Onboarding new servers

When generating the onboarding script in the portal, choose **Public endpoint** under *Connectivity method* and select the gateway you just created under *Gateway resource*. The script then contains the `--gateway-id` parameter with the full resource ID:

```powershell
& "$env:ProgramFiles\AzureConnectedMachineAgent\azcmagent.exe" connect `
    --resource-group "rg-servers-prd-weu-01" `
    --tenant-id "<tenant-id>" `
    --location "westeurope" `
    --subscription-id "<subscription-id>" `
    --cloud "AzureCloud" `
    --gateway-id "/subscriptions/<subscription-id>/resourceGroups/rg-hybrid-prd-weu-01/providers/Microsoft.HybridCompute/gateways/arcgw-prd-weu-01" `
    --correlation-id "<correlation-id>"
```

If you have an enterprise proxy, you set it as usual with `azcmagent config set proxy.url` before the `connect` command. The Arc proxy then uses it as upstream.

### Checking that it works

Two commands tell you everything. `azcmagent show` should display `http://localhost:40343` under *Using HTTPS Proxy*, with your enterprise proxy below it as *Upstream Proxy*. And `azcmagent check` shows per endpoint whether it is reachable; with a properly working gateway, `connection.type` is set to `gateway` and every line shows *Reachable: true*.

```powershell
azcmagent show
azcmagent check
azcmagent config get connection.type
```

After running `azcmagent logs` you will find the Arc proxy logs in the zip file under `ProgramData\AzureConnectedMachineAgent\Log\arcproxy.log` (Windows) or `/var/opt/azcmagent/log/arcproxy.log` (Linux). That is the first place to look if an extension refuses to work after the switch.

### Kubernetes clusters

For Arc enabled Kubernetes the pattern is the same, with `--gateway-resource-id` on `az connectedk8s connect` for new clusters and `az connectedk8s update` for existing ones. With `az connectedk8s show --query gateway.enabled` you check whether the association is active. Microsoft recommends waiting at least an hour after enabling it before removing the old endpoints from the firewall.

## The bypass list: deliberately routing traffic around the gateway

Since agent 1.64 you can specify, per server, a list of FQDNs that the Arc proxy does not send through the gateway, but through your enterprise proxy or directly. It is the mirror image of the old `proxy.bypass` setting: that one determined what your enterprise proxy skipped, the gateway bypass list determines what the gateway skips. The two are mutually exclusive, because `proxy.bypass` is ignored as soon as a server is set to `connection.type gateway`.

When would you want this? Three situations from practice:

1. **Private endpoints for Log Analytics or Key Vault.** Some organisations have an Azure Monitor Private Link Scope or a Key Vault with a private endpoint, reachable via ExpressRoute or VPN. You want to keep that traffic internal and not route it through a public gateway. Put `<workspace-id>.ods.opinsights.azure.com` or `<vault>.vault.azure.net` on the bypass list and the agent uses the internal route.
2. **Logging or DLP on the enterprise proxy.** If the security policy requires that specific categories of traffic, for example everything towards storage accounts, remain visible on your own proxy, you keep those endpoints outside the gateway.
3. **Troubleshooting.** If an extension has stopped working since the switch to the gateway, you can temporarily put that extension's endpoint on the bypass list to determine whether the gateway is the problem.

You configure the bypass list with `azcmagent config set`. Because the exact property name can differ per agent version and the documentation is still lagging behind here, `azcmagent config info` is your friend. That command shows all available properties for the installed agent version, including the gateway related ones. Then verify with `azcmagent check` that the endpoints on the bypass list do indeed go through your proxy or directly, and no longer through the gateway.

Keep in mind: everything you put on the bypass list has to be explicitly opened up again by your network team. Every exception eats into the gains of the gateway. So use the list deliberately and document why an endpoint is on it.

In summary, the agent's routing decision looks like this:

![Decision tree: without the gateway, through the corporate proxy to Azure with the full allowlist; with the gateway, FQDNs on the bypass list go through the corporate proxy and everything else through the Arc proxy and the gateway](/Images/arcgateway/arc-gateway-beslisboom.png)

## What the gateway does not cover

"One endpoint" is the marketing version. The honest version is: seven endpoints for the basics, plus a number of extra ones depending on what you run on the servers.

![The seven base endpoints that are always required, next to the extra endpoints that extensions such as Azure Monitor Agent, Update Manager and Defender require outside the gateway](/Images/arcgateway/arc-gateway-allowlist.png)

This is the minimal allowlist for Arc enabled servers with the gateway:

| Endpoint | Purpose |
| --- | --- |
| `<prefix>.gw.arc.azure.com` | Your own gateway URL |
| `management.azure.com` | Azure Resource Manager |
| `login.microsoftonline.com` | Microsoft Entra ID |
| `<region>.login.microsoft.com` | Microsoft Entra ID, regional |
| `gbl.his.arc.azure.com` | Arc agent cloud service |
| `<region>.his.arc.azure.com` | Arc control channel, regional |
| `download.microsoft.com` or `packages.microsoft.com` | Agent installation (Windows and Linux respectively) |

For Kubernetes, add `mcr.microsoft.com`, `*.data.mcr.microsoft.com` and optionally `<region>.obo.arc.azure.com` for cluster connect. Azure Local additionally needs bootstrap endpoints, the Key Vault and the storage account for the cloud witness, and the OCSP endpoints for certificate validation.

On top of that, there are extensions that simply keep requiring their own endpoints, gateway or not:

| Scenario | Extra endpoints |
| --- | --- |
| Azure Monitor Agent | `<workspace-id>.ods.opinsights.azure.com` |
| Arc enabled data services | `*.ods.opinsights.azure.com`, `*.oms.opinsights.azure.com`, `*.monitoring.azure.com` |
| Key Vault certificate sync | `<vault>.vault.azure.net` |
| Automation Hybrid Runbook Worker | `*.azure-automation.net` |
| Azure Update Manager (Windows) | All Windows Update endpoints |
| Microsoft Defender for Servers | All Defender endpoints |

What does go fully through the gateway without extra work: SSH via Arc, Extended Security Updates and the Azure extension for SQL Server. Those three are exactly the reason many customers roll out Arc in the first place, so that is good news.

There are three more conditions to remember. The gateway only works in the Azure public cloud, not in sovereign clouds. You can create a maximum of five gateway resources per subscription. And Microsoft gives a rule of thumb for capacity per region: calculate (servers ÷ 20) + (Kubernetes clusters ÷ 10) + (Azure Local instances ÷ 10). If that stays below 100, one gateway is enough. For most Dutch organisations that is more than sufficient; only at around two thousand servers in a single region do you start thinking about a second gateway.



## Migrating existing Arc servers to the gateway

Most organisations already have Arc servers running the classic way, with the full allowlist. You do not need to onboard those again. The association with a gateway is a property of the Azure resource, and the agent picks it up by itself.

```powershell
Set-AzConnectedMachine -ResourceGroupName "rg-servers-prd-weu-01" `
    -Name "srv-app-01" `
    -GatewayResourceId "/subscriptions/<subscription-id>/resourceGroups/rg-hybrid-prd-weu-01/providers/Microsoft.HybridCompute/gateways/arcgw-prd-weu-01"
```

For larger numbers, you loop over a resource group or a tag with `Get-AzConnectedMachine` and call `Set-AzConnectedMachine` per machine. It also works in the portal: open the gateway resource, go to *Associated resources*, choose *Add* and select the servers.

If an agent of version 1.50 or older is still running, you also need to run `azcmagent config set connection.type gateway` locally. From 1.51 onwards the agent handles that itself. Given the support policy, only agents younger than a year are supported, this is also a good moment to update old agents or enable automatic upgrades.

![Migration in four steps: create the gateway, associate a pilot group, roll out in batches, and only clean up after watching the proxy logs for at least a day](/Images/arcgateway/arc-gateway-migratie.png)

An approach that works well for me:

1. Create the gateway and have the network team add the gateway URL to the existing allowlist. Do not remove anything yet.
2. First associate a handful of servers, preferably with the heaviest extension set (Azure Monitor Agent, Update Manager, Defender), and verify with `azcmagent check` and in the portal that all extensions remain healthy.
3. Then roll out the association in batches across the rest, for example per resource group or per application team.
4. Wait at least a day, check the proxy logs for traffic to the old endpoints, and only then have the network team remove the redundant rules.

You can always roll back. Run `azcmagent config set connection.type direct` on the server and remove the association with `Set-AzConnectedMachine` without a gateway ID, or via *Remove* in the portal. From that moment on the full network requirements apply again, so only remove the old rules once you are sure you will not need to go back.

## Pitfalls from practice



**TLS inspection on the gateway URL.** This is by far the most common cause of a gateway that does not work. The nested TLS session cannot pass through an inspecting proxy. Symptom: the agent shows *Connected*, but extensions fail during installation or stop reporting. Solution: exclude `<prefix>.gw.arc.azure.com` from inspection. Arrange this before the first association, not after.

**Forgetting the gateway URL in the allowlist.** Sounds silly, happens anyway. The onboarding often succeeds because `login.microsoftonline.com` and `management.azure.com` were already open, and the server appears in the portal. After that the agent sends all remaining traffic through the gateway, and that traffic goes nowhere. Always verify with `azcmagent check` after the association.

**Cleaning up too early.** The temptation is strong to remove the old firewall rules immediately after the migration. Some extensions only talk to their endpoint once an hour or once a day. Watch the proxy logs for at least a day and, for Kubernetes, wait the hour Microsoft recommends.

**Expecting Defender and Update Manager to come along "for free".** Both have their own endpoint requirements that fall outside the gateway. If you sell the gateway as "nothing else needs to be opened after this", you will still end up having that conversation with the network team.

**Azure Local afterwards.** For existing Azure Local instances the gateway is not an option; only new deployments from version 2506 onwards support it. Take this into account in your planning if you want to combine Azure Local and Arc servers in a single network design. Creating the gateway resource for Azure Local can also take up to two hours.

**Old agents.** Below 1.51 you have to set `connection.type` manually, and below 1.64 there is no bypass list. So combine the gateway rollout with an agent upgrade round. Anything older than a year is out of support anyway.

## Conclusion

The Arc gateway solves the problem that most often delays hybrid cloud projects: not the technology, but the allowlist. Going from dozens of endpoints to seven, with a single gateway URL that carries all Arc traffic, is a difference that network teams understand immediately. Now that the gateway is GA for servers, Kubernetes and Azure Local, and the bypass list has smoothed out the rough edges of the first version, there is little reason to do new Arc rollouts without the gateway.

What it is not, is a magic wand. Defender, Update Manager and Azure Monitor Agent keep their own endpoints, TLS inspection on the gateway URL is a hard no go, and Azure Local can only join on a new deployment. If you name those limits up front, you keep the conversation with the network team honest and avoid the gateway later being labelled as "doesn't quite work after all".

My advice for organisations already running Arc: create the gateway, associate in batches, watch the logs for a few days and only then clean up. For new rollouts: start with it right away and include the seven base endpoints plus the extension endpoints as a single package in the change.

## Sources

* [Simplify network configuration requirements with Azure Arc gateway](https://learn.microsoft.com/en-us/azure/azure-arc/servers/arc-gateway), Microsoft Learn
* [Azure Arc gateway for Arc enabled Kubernetes](https://learn.microsoft.com/en-us/azure/azure-arc/kubernetes/arc-gateway-simplify-networking), Microsoft Learn
* [Overview of Azure Arc gateway for Azure Local](https://learn.microsoft.com/en-us/azure/azure-local/deploy/deployment-azure-arc-gateway-existing-cluster), Microsoft Learn
* [What's new with Azure Connected Machine agent](https://learn.microsoft.com/en-us/azure/azure-arc/servers/agent-release-notes), Microsoft Learn (versions 1.58, 1.64 and 1.66)
* [Announcing the General Availability of the Azure Arc gateway for Arc enabled Servers](https://techcommunity.microsoft.com/blog/azurearcblog/announcing-the-general-availability-of-the-azure-arc-gateway-for-arc-enabled-ser/4456356), Azure Arc Blog, 25 September 2025
* [Announcing the General Availability of Arc gateway for Azure Local](https://techcommunity.microsoft.com/blog/azurearcblog/announcing-the-general-availability-of-arc-gateway-for-azure-local/4456256), Azure Arc Blog, September 2025
* [Announcing the General Availability of the Azure Arc gateway for Arc enabled Kubernetes](https://techcommunity.microsoft.com/blog/azurearcblog/announcing-the-general-availability-of-the-azure-arc-gateway-for-arc-enabled-kub/4498561), Azure Arc Blog, 2 March 2026
