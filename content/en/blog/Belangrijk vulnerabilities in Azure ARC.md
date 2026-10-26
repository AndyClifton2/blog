---
Author: "Andy Clifton"
title: "Important vulnerabilities in Azure Arc"
description: "Test environment Azure Arc."
date: 2026-03-13
tags: ["ARC"]
thumbnail: "/img/Designer.png"
---





### ==> Disclaimer: work in progress. <==

## Foreword

Many organisations see Azure Arc as a logical and relatively "safe" extension of Azure management to on-premises and multi-cloud environments. It feels like management, not like identity infrastructure. It is precisely that perception that makes vulnerabilities in Azure Arc extra interesting, and dangerous.

Over the past few years we have seen a clear trend: more and more cloud functionality is being moved into agents that run locally but act on behalf of the cloud. Azure Arc is a textbook example of this.

The Arc agent is not a passive monitor, but an active part of the Azure control plane, with access to identities, configurations and management tasks.
In this article we look at CVE-2026-26117, a vulnerability in the Azure Arc agent for Windows that at first glance looks like "just" a local privilege escalation. The research by Cymulate, however, shows that the impact reaches much further: from local escalation to cloud identity takeover.
This blog is deliberately technical, to give insight into where the trust boundaries lie, how this vulnerability arises, and what it means for the way we should approach Azure Arc from an architectural and security perspective.


## <ins>Why Azure Arc is vulnerable here:</ins>

Azure Arc-enabled servers rely on an agent-based trust model

Important (Windows) components such as:

- azcmagent: the agent for Azure Arc
- HIMDS (Hybrid Instance Metadata Service): the local metadata endpoint
- Guest Configuration service
- Arc Proxy service

These services run locally with elevated privileges and are responsible for:

- Authentication towards Azure
- Token acquisition for the machine identity
- Executing cloud-initiated actions (extensions, policies)

The core of the problem:

Local service-to-service communication is considered trusted, while that communication is accessible to a local user with low privileges.



## <ins>Technical root cause (CWE-288)</ins>


Microsoft classifies CVE-2026-26117 as:

Authentication Bypass Using an Alternate Path or Channel (CWE-288) [nvd.nist.gov]

In concrete terms this means that:

- Internal Arc services do not sufficiently enforce who is allowed to call them
- Authentication between components can be bypassed through alternative communication paths
- Responses from internal services can be manipulated

As a result, an unprivileged local user can effectively impersonate a trusted Arc component.

## <ins> Attack chain (technical) </ins>

<ins> Step 1: Initial access (Local, Low Privilege) </ins>
The attacker has:

- A local Windows account
- No admin rights required

This could be a service account, a support user or a compromised application.

![alt](/Images/vulnability/AzureArc7.png)


<ins>Step 2: Interception of Arc service communication </ins>
The attacker:

- Intercepts or imitates communication between Arc services
- Abuses weak authentication between HIMDS / Guest Configuration / Arc Proxy

Result:
The attacker can inject fake responses that other Arc components treat as legitimate.

![alt](/Images/vulnability/AzureArc8.png)
![alt](/Images/vulnability/AzureArc9.png)

<ins>Step 3: Local Privilege Escalation </ins>

By abusing the service interaction:

- Actions are executed in the context of SYSTEM
- The attacker gains full control over the OS

This makes it very easy for an attacker to move through the network and maintain access.

![alt](/Images/vulnability/AzureArc11.png)

<ins>Step 4: Cloud Identity Impersonation </ins>
After gaining SYSTEM access:

- The attacker can obtain tokens for the Azure Arc machine identity
- This identity may hold Azure RBAC permissions

Examples of abuse:

- Modifying Azure resource properties
- Running Arc extensions
- Reading or changing configurations
- Interacting with other Azure resources within the RBAC scope

![alt](/Images/vulnability/AzureArc13.png)
![alt](/Images/vulnability/AzureArc14.png)

<ins>Step 5: Tenant Hijack (worst case) </ins>
Cymulate demonstrates that it is possible to:

- Re-onboard the machine
- Into a tenant controlled by the attacker

With this, the original tenant completely loses visibility and control over the machine.

## Impact analysis

This is not a classic "local only" vulnerability, but a hybrid privilege escalation

## Affected scope
The following are vulnerable:

- All Azure Arc-enabled Windows machines
- With Azure Arc Agent < 1.61
- Regardless of hosting location (on-prem, AWS, GCP, edge)

## Solutions

<ins> 1. Enforce the patch level </ins>

Microsoft has fixed the issue in

Azure Arc Agent Services version 1.61

Updating and restarting the agent services is required. (This can be done through Update Manager or locally through Windows Update)


<ins> 2. Minimise Azure RBAC </ins>

Check:

- Which Arc machine identities hold permissions.
- Remove Contributor-like roles where possible
- Use custom roles with minimal permissions

90% of the permissions that get handed out are unnecessary. With an EPM-like solution you can properly check which permissions accounts or identities actually need.

<ins> 3. Treat Arc as a Tier-0 component </ins>

Treat Arc agents in the same way as:

- domain controllers
- identity brokers
- management jump hosts

Local compromise = cloud impact.
So put as many permissions and capabilities as possible behind PIM roles.

<ins> 4. Logging & detection </ins>

Watch for:

- Re-registration events of Arc machines
- Token usage of machine identities
- Unusual ARM API calls from Arc identities

These are all things you want to capture in an Application Log and have surface in Azure Monitor or Sentinel.

## Finally

What this vulnerability exposes above all is a shift in architectural thinking. Where identity and authorisation used to live primarily in the datacenter or in Azure itself, that responsibility is increasingly moving to agents at the edge of the network.
Azure Arc is not an exception to this, but a harbinger.
Anyone using Azure Arc should not only ask "What can I manage with it?", but above all "What position of trust am I giving this agent?"

CVE-2026-26117 shows that this is no longer a theoretical question.

## Azure Arc is functionally powerful, but it also shifts identity and trust to the edge. That requires a different security model than traditional on-prem tooling.
