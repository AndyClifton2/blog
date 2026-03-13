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
Dit blog is bewust technisch ingestoken. Niet om een exploit te beschrijven, maar om inzicht te geven in waar de trust boundaries liggen, hoe deze kwetsbaarheid ontstaat, en wat dit betekent voor de manier waarop we Azure Arc architectonisch en security‑matig moeten benaderen.


## <ins>waarom Azure Arc hier kwetsbaar is:</ins>

Azure Arc‑enabled servers vertrouwen op een agent‑gebaseerd trustmodel:
Belangrijke componenten (Windows)

azcmagent – hoofdagent voor Azure Arc
HIMDS (Hybrid Instance Metadata Service) – lokale metadata endpoint
Guest Configuration service
Arc Proxy service

Deze services draaien lokaal met verhoogde privileges en zijn verantwoordelijk voor:

authenticatie richting Azure
token‑acquisitie voor de machine‑identiteit
uitvoeren van cloud‑geïnitieerde acties (extensions, policies)

De kern van het probleem:

Lokale service‑to‑service communicatie wordt beschouwd als trusted, terwijl die communicatie benaderbaar is door een lokale gebruiker met lage privileges.



## <ins>Technische oorzaak (CWE‑288)</ins>


Microsoft classificeert CVE‑2026‑26117 als:

Authentication Bypass Using an Alternate Path or Channel (CWE‑288) [nvd.nist.gov]

Concreet betekent dit dat:

interne Arc‑services onvoldoende afdwingen wie hen mag aanspreken
authenticatie tussen componenten kan worden omzeild via alternatieve communicatiepaden
responses van interne services gemanipuleerd kunnen worden

Hierdoor kan een niet‑bevoegde lokale gebruiker zich effectief voordoen als een vertrouwde Arc‑component.

## <ins> Aanvalsketen (technisch) </ins>

Stap 1 – Initiële toegang (Local, Low Privilege)
De aanvaller heeft:

een lokale Windows‑account
geen admin‑rechten nodig

Dit kan een service‑account, support‑user of gecompromitteerde applicatie zijn.

![alt](/Images/vulnability/AzureArc7.png)


Voorafgaand aan exploitatie stelt de aanvaller een ARC-gekoppeld apparaat op in zijn eigen door aanvaller gecontroleerde tenant

