---
Author: "Andy Clifton"
title: "Belangrijke vulnerabilities Azure Arc"
description: "Test omgeving Azure Arc."
date: 2026-03-13
tags: ["ARC"]
thumbnail: "/img/Designer.png"
---





### ==> Disclaimer work in progress. <==

## Voorwoord

Azure Arc wordt door veel organisaties gezien als een logische en relatief “veilige” uitbreiding van Azure‑beheer naar on‑premises en multi‑cloud omgevingen. Het voelt als beheer, niet als identity‑infrastructuur. Precies dát beeld maakt kwetsbaarheden in Azure Arc extra interessant – en gevaarlijk.

De afgelopen jaren zien we een duidelijke trend: steeds meer cloudfunctionaliteit wordt verplaatst naar agents die lokaal draaien, maar namens de cloud handelen. Azure Arc is daar een schoolvoorbeeld van. 

De Arc agent is geen passieve monitor, maar een actief onderdeel van de Azure control plane, met toegang tot identiteiten, configuraties en beheertaken.
In dit artikel kijken we naar CVE‑2026‑26117, een kwetsbaarheid in de Azure Arc agent voor Windows die op het eerste gezicht “slechts” een lokale privilege escalation lijkt. Het onderzoek van Cymulate laat echter zien dat de impact veel verder reikt: van lokale escalatie naar cloud identity takeover.
Deze blog is bewust technisch ingestoken om inzicht te geven in waar de trust boundaries liggen, hoe deze kwetsbaarheid ontstaat, en wat dit betekent voor de manier waarop we Azure Arc architectonisch en security‑matig moeten benaderen.


## <ins>Waarom Azure Arc hier kwetsbaar is:</ins>

Azure Arc‑enabled servers vertrouwen op een agent‑based trustmodel

Belangrijke (Windows) componenten zoals:

- azcmagent – agent voor Azure Arc
- HIMDS (Hybrid Instance Metadata Service) – lokale metadata endpoint
- Guest Configuration service
- Arc Proxy service

Deze services draaien lokaal met verhoogde privileges en zijn verantwoordelijk voor:

- Authenticatie richting Azure
- Token‑acquisitie voor de machine‑identiteit
- Uitvoeren van cloud‑geïnitieerde acties (extensions, policies)

De kern van het probleem:

Lokale service‑to‑service communicatie wordt beschouwd als trusted, terwijl die communicatie benaderbaar is door een lokale gebruiker met lage privileges.



## <ins>Technische oorzaak (CWE‑288)</ins>


Microsoft classificeert CVE‑2026‑26117 als:

Authentication Bypass Using an Alternate Path or Channel (CWE‑288) [nvd.nist.gov]

Concreet betekent dit dat:

- Interne Arc‑services onvoldoende afdwingen wie hen mag aanspreken
- Authenticatie tussen componenten kan worden omzeild via alternatieve communicatiepaden
- Responses van interne services gemanipuleerd kunnen worden

Hierdoor kan een niet‑bevoegde lokale gebruiker zich effectief voordoen als een vertrouwde Arc‑component.

## <ins> Aanvalsketen (technisch) </ins>

<ins> Stap 1 – Initiële toegang (Local, Low Privilege) </ins>
De aanvaller heeft:

- Een lokale Windows‑account
- Geen admin‑rechten nodig

Dit kan een service‑account, support‑user of gecompromitteerde applicatie zijn.

![alt](/Images/vulnability/AzureArc7.png)


<ins>Stap 2 – Interceptie van Arc‑service communicatie </ins>
De aanvaller:

- Onderschept of imiteert communicatie tussen Arc‑services
- Misbruikt zwakke authenticatie tussen HIMDS / Guest Configuration / Arc Proxy

Resultaat:
De aanvaller kan valse responses injecteren die door andere Arc‑componenten als legitiem worden gezien.

![alt](/Images/vulnability/AzureArc8.png)
![alt](/Images/vulnability/AzureArc9.png)

<ins>Stap 3 – Local Privilege Escalation </ins>

Door misbruik van service‑interactie:

- Worden acties uitgevoerd in de context van SYSTEM
- Verkrijgt de aanvaller volledige controle over het OS

Daardoor wordt het heel makkelijk voor een aanvaller om zich door het netwerk te verplaatsen en toegang te behouden
.
![alt](/Images/vulnability/AzureArc11.png)

<ins>Stap 4 – Cloud Identity Impersonation </ins>
Na SYSTEM‑toegang:

- Kan de aanvaller tokens verkrijgen voor de Azure Arc machine‑identiteit
- Deze identiteit kan Azure RBAC‑rechten hebben

Voorbeelden van misbruik:

- Aanpassen van Azure resource properties
- Uitvoeren van Arc extensions
- Uitlezen of wijzigen van configuraties
- Interactie met andere Azure resources binnen RBAC‑scope

![alt](/Images/vulnability/AzureArc13.png)
![alt](/Images/vulnability/AzureArc14.png)

<ins>Stap 5 – Tenant Hijack (worst‑case) </ins>
Cymulate toont aan dat het mogelijk is om:

- De machine opnieuw te laten onboarden
- Richting een door de aanvaller‑gecontroleerde tenant

Hiermee verliest de oorspronkelijke tenant volledig zicht en controle op de machine.

## Impactanalyse

Dit is géén klassieke “local only” kwetsbaarheid, maar een hybride privilege‑escalatie

## Affected scope
Kwetsbaar zijn:

- Alle Azure Arc‑enabled Windows machines
- Met Azure Arc Agent < 1.61
- Ongeacht hostinglocatie (on‑prem, AWS, GCP, edge)

## Oplossingen

<ins> 1. Patchniveau afdwingen </ins>

Microsoft heeft het probleem opgelost in

Azure Arc Agent Services version 1.61

Updaten en agent services herstarten is vereist. (Dit kan gedaan worden via Update Manager of lokaal via Windows Update)


<ins> 2. Azure RBAC minimaliseren </ins>

Controleer:

- Welke Arc‑machine‑identity's rechten hebben.
- Verwijder Contributor‑achtige rollen waar mogelijk
- Gebruik custom roles met minimale permissions

90% van de rechten die uitgedeeld worden zijn overbodig. Met een EPM achtige oplossing kun je goed controleren welke rechten accounts of Identity's ook echt nodig hebben.

<ins> 3. Beschouw Arc als Tier‑0 component </ins>

Behandel Arc‑agents vergelijkbaar met:

- domain controllers
- identity brokers
- management jump hosts

Lokale compromise = cloud impact.
Zet dus ook zoveel mogelijk rechten en mogelijkheden achter PIM rollen.

<ins> 4. Logging & detectie </ins>

Let op:

- Herregistratie‑events van Arc machines
- Token usage van machine identities
- Ongebruikelijke ARM API calls vanuit Arc‑identity's

Dit zijn allemaal dingen die je wilt opvangen in een Application Log en wil laten terugkomen in Azure Monitor of Sentinel.

## Tot slot

Wat deze kwetsbaarheid vooral blootlegt, is een verschuiving in architectuur denken. Waar vroeger identiteit en autorisatie primair in het datacenter of in Azure zelf lagen, verschuift die verantwoordelijkheid steeds vaker naar agents aan de rand van het netwerk.
Azure Arc is daarin geen uitzondering, maar een voorbode.
Wie Azure Arc gebruikt, doet er goed aan om niet alleen te vragen “Wat kan ik ermee beheren?”, maar vooral “Welke vertrouwenspositie geef ik deze agent?”

CVE‑2026‑26117 laat zien dat dat geen theoretische vraag meer is.

## Azure Arc is functioneel krachtig, maar verschuift ook identity en trust naar de edge. Dat vereist een ander beveiligingsmodel dan traditionele on‑prem tooling.
