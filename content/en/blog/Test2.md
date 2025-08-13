---
Author: "Andy Clifton"
title: "Test"
description: "Test omgeving Azure Arc."
date: 2025-08-12
tags: ["ARC"]
thumbnail: "/img/arc.jpeg"
---



# Run Powershell Script met Hybrid Worker Groups

### ==> Disclaimer work in progress. <==

## Voorwoord Tset

Als je de on-premises omgeving wilt automatiseren, is Azure Arc Server een geweldige aanbieding voor het onboarden van Azure-beheerservices zoals Azure Monitor, Sentinel, maar ook Defender en IAM.
Een van de andere voordelen die we nog hebben is het gebruik van Hybrid Workers. Hierdoor kun je Runbooks, Logic Apps of Power Automate gebruiken om het beheer of deploy van dingen op je on-premise vm mogelijk te maken.

In deze blog laat ik je zien hoe je een Automation account maakt, hoe je Connect met een server via Azure ARC, het creëren van een Hybrid Workers group en daarna het creëren en deployen van een runbook op basis van Powershell.

![Image](/Images/RunPowershellHybrid/Praatplaat.JPG)

## Maken van een Azure Automation Account.

Naar mijn idee kun je het Automation Account het beste in een eigen RG stoppen. Hierdoor houd je overzichtelijk wat hoort bij dit Automation account. Hou er netwerk technisch rekening mee dat dit account en alles wat er onder staat bij de omgevingen kan komen.

In dit document beschrijf ik niet hoe je een Resource group moet maken omdat ik er vanuit ga dat dit al bekend is.