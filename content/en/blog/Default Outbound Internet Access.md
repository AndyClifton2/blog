---
Author: "Andy Clifton"
title: "Default Outbound Internet Access"
description: "Default Outbound Internet Access niet langer by default allowed."
date: 2025-08-13
tags: ["Networking"]
thumbnail: "/img/outbound.jpeg"
---



![alt](/Images/Outbound/Outbound.Jpeg)
 
# Default Outbound Internet Access wordt uitgeschakeld

### ==> Disclaimer work in progress. <==

## Voorwoord
Tot nu toe gaf Microsoft standaard outbound internettoegang aan elk nieuw Virtual Network.
Dat gebeurde automatisch via een default route naar het internet, zonder dat je daar iets voor hoefde te doen.
Deze standaardinstelling gaf automatisch internettoegang zonder dat je dit expliciet hoefde te configureren.
Dat leek handig, maar het bracht risico’s met zich mee: ongecontroleerd verkeer, moeilijk te monitoren, en kwetsbaar voor datalekken.
Securityteams hadden minder grip op outbound flows, wat niet paste binnen Zero Trust principes.
Vanaf nu moet je zelf expliciet outbound access regelen via NAT Gateway of andere oplossingen.

![alt](/Images/Outbound/network.jpeg)
<ins>Wat Zijn de gevolgen?</ins>

Tot nu toe gaf Azure je automatisch internettoegang als je een nieuw Virtual Network aanmaakte.
Handig, toch? Je hoefde niks extra’s te doen—je VM kon gewoon het internet op.
Maar vanaf 30 september is dat verleden tijd. Microsoft zet deze automatische toegang uit.

<ins>Wat merk jij ervan?</ins>

- Nieuwe Virtual Networks hebben geen automatische internettoegang meer.  
- Applicaties die afhankelijk zijn van outbound internet moeten expliciet worden geconfigureerd.
- Je moet zelf een NAT Gateway of andere outbound oplossing instellen.
- Deployment scripts en templates kunnen falen als ze uitgaan van standaard internettoegang.
- Monitoring en logging worden belangrijker om outbound verkeer te beheren.
- Security wordt strakker: geen ongecontroleerde verbindingen meer.
- Je krijgt meer controle, maar ook meer verantwoordelijkheid.
- Kosten kunnen stijgen door het gebruik van NAT Gateways.
- Legacy-architecturen moeten mogelijk worden aangepast.
- Het dwingt teams om bewuster om te gaan met netwerkbeveiliging en outbound flows.

**Voor bestaande Vnet's veranderd er op dit moment nog niks**

## Welke oplossingen zijn er om dit dan goed te regelen.

![alt](/Images/Outbound/optie.jpeg)


**Azure NAT Gateway**

Dit is de aanbovolen oplossing voor de meeste omgevingen door Microsoft waarom:

*Verbeterde Security*

NAT Gateway biedt gecontroleerde outbound internettoegang, waardoor je precies kunt bepalen welke resources toegang hebben tot het internet. Dit sluit ongecontroleerde verbindingen uit en past beter bij Zero Trust principes.

*Logging en Monitoring*

Met NAT Gateway kun je outbound verkeer beter monitoren en loggen. Dit maakt het eenvoudiger om verdachte activiteiten te detecteren en te auditen.

*Schaalbaarheid*

NAT Gateway is ontworpen voor hoge schaalbaarheid en ondersteunt duizenden gelijktijdige verbindingen, wat essentieel is voor grote workloads.

*Statische IP-adressen*

Je kunt één of meerdere statische publieke IP-adressen toewijzen aan je outbound verkeer, waardoor je eenvoudiger firewallregels en toegangsbeleid kunt beheren.

*Kostenbeheer*

NAT Gateway maakt het mogelijk om outbound verkeer te centraliseren, waardoor je kosten kunt optimaliseren en onverwachte uitgaven door ongecontroleerde outbound flows voorkomt.


**Azure Load Balancer (Outbound Rules)**

Azure Load Balancer kan ook worden gebruikt om outbound internettoegang te regelen, vooral in scenario’s waar je al een Load Balancer gebruikt voor inbound verkeer. Waarom is dit een goede vervanging?

*Flexibele Outbound Configuratie*

Met outbound rules kun je precies bepalen welke resources outbound internettoegang krijgen, en hoe het verkeer wordt afgehandeld.

*Statische IP-adressen*

Je kunt publieke IP-adressen koppelen aan je outbound verkeer, zodat je altijd weet vanaf welk IP je resources het internet op gaan.

*Schaalbaarheid*

Azure Load Balancer ondersteunt grote aantallen gelijktijdige verbindingen en is geschikt voor zowel kleine als grote workloads.

*Integratie met bestaande infrastructuur*

Als je al een Load Balancer gebruikt voor inbound verkeer, kun je eenvoudig outbound access toevoegen zonder extra componenten.

*Kostenbesparing*

Voor sommige scenario’s kan Load Balancer goedkoper zijn dan NAT Gateway, vooral bij kleinere workloads of als je al een Load Balancer hebt draaien.

**Azure Firewall**

Azure Firewall is een geavanceerde oplossing voor outbound internettoegang, vooral geschikt voor omgevingen waar security en compliance centraal staan. Waarom is dit een goede vervanging?

*Geavanceerde Security*

Azure Firewall biedt diepgaande inspectie van outbound verkeer, inclusief filtering op applicatie- en netwerklaag. Je kunt strikte regels instellen voor toegestane en geblokkeerde verbindingen.

*Logging, Monitoring en Threat Intelligence*

Firewall logs zijn zeer gedetailleerd en kunnen worden geïntegreerd met Azure Monitor en Sentinel. Je krijgt inzicht in alle outbound flows en kunt dreigingen snel detecteren.

*Schaalbaarheid*

Azure Firewall schaalt automatisch mee met je workload en ondersteunt grote aantallen gelijktijdige verbindingen.

*Statische en meerdere publieke IP-adressen*

Je kunt één of meerdere publieke IP-adressen toewijzen aan outbound verkeer, wat handig is voor toegangsbeheer en compliance.

*Centralisatie en segmentatie*

Met Azure Firewall kun je outbound access centraal beheren voor meerdere subnetten en vnets, en segmentatie toepassen voor extra security.
Het minpunt van Azure Firewall is dat het relatief gezien een vrij dure oplossing is.

**Public IP op de VM**

Het toewijzen van een Public IP-adres direct aan een VM is een eenvoudige manier om outbound internettoegang te regelen. Waarom kan dit een alternatief zijn?

*Directe internettoegang*

De VM heeft direct toegang tot het internet zonder tussenliggende componenten, wat configuratie eenvoudig maakt.

*Statisch IP-adres*

Je kunt een statisch Public IP-adres toewijzen, zodat je altijd weet vanaf welk IP je VM het internet op gaat.

*Eenvoudige troubleshooting*

Problemen met outbound verkeer zijn makkelijker te traceren omdat het verkeer direct vanaf de VM komt.

*Geen extra kosten voor NAT of Load Balancer*

Je betaalt alleen voor het Public IP-adres, niet voor extra netwerkcomponenten.

*Geschikt voor kleine workloads*

Voor test- of ontwikkelomgevingen kan dit een snelle en goedkope oplossing zijn.

Let op: deze optie is minder veilig en minder schaalbaar dan NAT Gateway, Load Balancer of Firewall. Gebruik dit alleen als security geen grote rol speelt.
