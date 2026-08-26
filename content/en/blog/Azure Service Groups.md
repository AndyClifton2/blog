---
Author: "Andy Clifton"
title: "Azure Service Groups"
description: "Azure Service Groups (public preview) laten je resources over meerdere subscriptions heen logisch groeperen, zonder ze te verplaatsen. Een overzicht van wat het is, waarom het relevant is en hoe je het opzet met Bicep."
date: 2026-08-24
tags: [""]
thumbnail: "/img/azure-service-groups-cover.png"
---

# Azure Service Groups: eindelijk grip op workloads die over meerdere subscriptions verspreid zitten

Als je al een tijdje met Azure governance bezig bent, herken je dit vast. Management groups geven je een mooie hiërarchie om subscriptions in te ordenen en daar policies en RBAC op toe te passen. Maar zodra een workload zelf over meerdere subscriptions verspreid staat, wordt het lastiger. Denk aan een netwerkteam dat gateways, load balancers en andere netwerkcomponenten beheert die in vijf verschillende subscriptions staan. Je wilt die resources als één geheel kunnen bekijken en beheren, zonder ze allemaal te verplaatsen naar één subscription. Microsoft heeft daar nu een antwoord op, in public preview: Azure Service Groups.

## Wat is een Service Group precies

Een Service Group is een nieuw type logisch object, los van resource groups, subscriptions en management groups, maar wel vergelijkbaar in hoe je ermee werkt. Het is eigenlijk een container die metadata bijhoudt over een verzameling resources, ongeacht in welke subscription of resource group die resources daadwerkelijk staan. De resources zelf blijven gewoon staan waar ze staan. Je koppelt ze alleen aan de Service Group.

Dat betekent dat je eindelijk een logische groepering kunt maken die past bij hoe een workload er in de praktijk uitziet, in plaats van hoe je toevallig je subscriptions had ingedeeld toen je begon.

## Waarom dit relevant is

Voor mij is het meest interessante punt niet eens de groepering zelf, maar wat je erdoor terugkrijgt. Zodra resources aan een Service Group hangen, krijg je automatisch:

* Een inventarisatie van alle resources binnen die groep
* Een overzicht van issues die spelen bij die resources
* Application maps en beschikbaarheidstests, als Application Insights gekoppeld is
* Monitoring aanbevelingen en dekking voor VM's en AKS clusters

Daarnaast kun je met de rol Service Group Reader iemand toegang geven tot precies die verzameling resources, zonder dat diegene overal in je tenant hoeft rond te kijken. Voor een team dat verantwoordelijk is voor één specifieke workload, maar geen volledige toegang tot subscriptions nodig heeft, is dat precies wat ontbrak.

## Hoe de hiërarchie werkt

Service Groups vormen net als management groups een boomstructuur. Je kunt een Service Group direct onder de Tenant Root Service Group plaatsen, of onder een andere Service Group als child. Elke Service Group heeft een unieke globale naam nodig, vergelijkbaar met een storage account naam, met een maximum van 250 tekens.

Op dit moment kun je een Service Group aanmaken en resources eraan koppelen via de Azure Portal of via de Azure REST API. Een dedicated cmdlet in PowerShell of een specifieke Azure CLI opdracht is er nog niet, dus voor automatisering ben je aangewezen op Bicep, ARM templates of directe REST calls.

## Een Service Group aanmaken met Bicep

Hieronder een voorbeeld waarin een Service Group wordt aangemaakt en een storage account eraan gekoppeld wordt.

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

Wat hier gebeurt: de Service Group zelf wordt aangemaakt op tenant scope, met een parent die verwijst naar de root Service Group of naar een andere Service Group. Voor het koppelen van een bestaande resource, in dit geval een storage account, gebruik je het resource type Microsoft.Relationships/serviceGroupMember. Met het keyword existing haal je een referentie naar de resource op, zonder die opnieuw uit te rollen.

Let op één detail: in Bicep accepteert de scope property van serviceGroupMember geen resourceId als string. Wil je dat wel doen, dan moet je terugvallen op een ARM template, waar dat via de scope property op resourceId niveau wel kan.

## Mijn eerste indruk

Service Groups is nog public preview, dus reken er niet op dat de API's en tooling nu al definitief zijn. Verwacht wijzigingen richting general availability. Maar voor iedereen die middelgrote of grote Azure tenants beheert, en het probleem herkent van workloads die je governance hiërarchie doorkruisen, is dit zeker de moeite van het uitproberen waard. Ik ga het zelf de komende tijd testen binnen een paar klantomgevingen en deel hier mijn ervaringen zodra ik verder ben.