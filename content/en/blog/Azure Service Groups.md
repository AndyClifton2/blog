---
Author: "Andy Clifton"
title: "Azure Service Groups"
description: "Azure Service Groups (public preview) let you logically group resources across multiple subscriptions, without moving them. An overview of what it is, why it matters and how to set it up with Bicep."
date: 2026-08-24
tags: [""]
thumbnail: "/img/azure-service-groups-cover.png"
---

# Azure Service Groups: finally a grip on workloads spread across multiple subscriptions

If you have been working on Azure governance for a while, you will probably recognise this. Management groups give you a nice hierarchy to organise subscriptions in and to apply policies and RBAC to. But as soon as a workload itself is spread across multiple subscriptions, things get harder. Think of a network team that manages gateways, load balancers and other network components that live in five different subscriptions. You want to be able to view and manage those resources as a single whole, without moving them all into one subscription. Microsoft now has an answer to that, in public preview: Azure Service Groups.

## What exactly is a Service Group

A Service Group is a new type of logical object, separate from resource groups, subscriptions and management groups, but similar in how you work with it. It is essentially a container that keeps metadata about a collection of resources, regardless of which subscription or resource group those resources actually live in. The resources themselves simply stay where they are. You only link them to the Service Group.

That means you can finally create a logical grouping that matches what a workload looks like in practice, instead of how you happened to organise your subscriptions when you started.

## Why this matters

For me, the most interesting part is not even the grouping itself, but what you get back from it. As soon as resources are attached to a Service Group, you automatically get:

* An inventory of all resources within that group
* An overview of the issues affecting those resources
* Application maps and availability tests, if Application Insights is connected
* Monitoring recommendations and coverage for VMs and AKS clusters

On top of that, with the Service Group Reader role you can give someone access to exactly that collection of resources, without that person having to browse around your entire tenant. For a team that is responsible for one specific workload but does not need full access to subscriptions, that is exactly what was missing.

## How the hierarchy works

Just like management groups, Service Groups form a tree structure. You can place a Service Group directly under the Tenant Root Service Group, or under another Service Group as a child. Every Service Group needs a globally unique name, similar to a storage account name, with a maximum of 250 characters.

At the moment you can create a Service Group and link resources to it through the Azure Portal or through the Azure REST API. There is no dedicated PowerShell cmdlet or specific Azure CLI command yet, so for automation you have to rely on Bicep, ARM templates or direct REST calls.

## Creating a Service Group with Bicep

Below is an example in which a Service Group is created and a storage account is linked to it.

```bicep
param serviceGroupId string
param parentServiceGroupId string
param serviceGroupName string
param storageAccountName string

resource serviceGroup 'Microsoft.Management/serviceGroups@2024-02-01-preview' = {
  scope: tenant()
  name: serviceGroupId
  properties: {
    displayName: serviceGroupName
    parent: {
      resourceId: '/providers/Microsoft.Management/serviceGroups/${parentServiceGroupId}'
    }
  }
}

resource storageAccount 'Microsoft.Storage/storageAccounts@2025-06-01' existing = {
  name: storageAccountName
}

resource rel1 'Microsoft.Relationships/serviceGroupMember@2023-09-01-preview' = {
  scope: storageAccount
  name: 'childResource1'
  properties: {
    targetId: serviceGroup.id
  }
}
```

What happens here: the Service Group itself is created at tenant scope, with a parent that points to the root Service Group or to another Service Group. To link an existing resource, in this case a storage account, you use the resource type Microsoft.Relationships/serviceGroupMember. The existing keyword gives you a reference to the resource without deploying it again.

One detail to watch out for: in Bicep, the scope property of serviceGroupMember does not accept a resourceId as a string. If you do want that, you have to fall back on an ARM template, where it is possible through the scope property at resourceId level.

## My first impression

Service Groups is still in public preview, so do not count on the APIs and tooling being final yet. Expect changes on the way to general availability. But for anyone managing medium-sized or large Azure tenants who recognises the problem of workloads cutting across your governance hierarchy, this is definitely worth trying out. I will be testing it myself in a few customer environments over the coming period and will share my experiences here once I have made more progress.
