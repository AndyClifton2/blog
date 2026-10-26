---
title: "Azure Arc gateway: één endpoint voor al je hybride verkeer"
date: 2026-09-30
author: Andy Clifton
tags: [Azure, Azure Arc, Hybride cloud, Netwerk, Azure Local]
thumbnail: "/img/gateway.jpg"
---

# Azure Arc gateway: één endpoint voor al je hybride verkeer

![Azure Arc gateway: tientallen servers die via één gateway met Azure praten](/Images/arcgateway/arc-gateway-hero.png)

## Waarom netwerkteams zuchten bij Azure Arc?

Wie ooit een Azure Arc onboarding heeft begeleid bij een organisatie met een strikt netwerkbeleid, kent het gesprek. Je levert de lijst met benodigde endpoints aan en het netwerkteam kijkt je aan alsof je net hebt gevraagd of de firewall even helemaal open mag. Tientallen FQDN's, deels met wildcards, deels regio afhankelijk, en bij elke nieuwe extensie komen er weer een paar bij. Azure Monitor Agent, Update Manager, Defender, Machine Configuration, Key Vault sync: allemaal praten ze met hun eigen set endpoints.

Het resultaat is voorspelbaar. De change duurt weken, er sluipen typefouten in de allowlist en zes maanden later ontdek je dat een extensie stilletjes niet meer werkt omdat Microsoft een endpoint heeft toegevoegd die niemand heeft gezien. Bij overheidsorganisaties en waterschappen, waar iedere uitgaande verbinding onderbouwd moet worden is dit vaak de grootste vertragende factor in een hybride cloud traject. Niet de techniek, maar de allowlist.

![Zonder gateway loopt elke server naar tientallen endpoints door de firewall; met gateway bundelt de Arc proxy alles op één regel naast zes onveranderde basisendpoints](/Images/arcgateway/arc-gateway-voor-na.png)

De Azure Arc gateway is Microsofts antwoord hierop. In plaats van tientallen endpoints open je er nog zeven, waarvan er één jouw eigen gateway URL is waar vrijwel al het Arc verkeer doorheen loopt. In deze blog leg ik uit hoe de gateway werkt, wat er het afgelopen jaar is veranderd, hoe je hem inricht, en vooral: waar de grenzen liggen. Want "één endpoint" klinkt mooi, maar het is niet het hele verhaal.

## Hoe de Arc gateway werkt:

De Arc gateway bestaat uit twee onderdelen die samenwerken. Aan de Azure kant staat de **gateway resource**, een beheerobject van het type `Microsoft.HybridCompute/gateways`. Bij het aanmaken krijgt die een unieke URL in de vorm `<prefix>.gw.arc.azure.com`. Deze URL is een front end op basis van Azure Front Door en fungeert als gezamenlijke voordeur voor het Arc verkeer. Aan de serverkant zit de **Arc proxy**, een component die al in de Connected Machine agent zit en die zich gedraagt als forward proxy voor de agent zelf en voor alle extensies. Je hoeft die proxy niet apart te installeren of te configureren; zodra een server aan een gateway gekoppeld is, gaat de agent hem gebruiken.

De verkeersstroom wordt daarmee: Arc agent en extensies, naar de lokale Arc proxy op `localhost:40343`, eventueel via je eigen enterprise proxy, naar de Arc gateway, en van daaruit naar de eigenlijke Azure dienst. Voor je firewall ziet het er dus uit alsof de server slechts met een handvol adressen praat, terwijl achter de gateway de tientallen Azure endpoints gewoon nog bestaan.


Een belangrijk detail voor security teams is hoe TLS hierin zit. De Arc proxy zet een TLS sessie op naar de gateway en stuurt daarbinnen een geneste HTTP CONNECT naar het doel endpoint. Als dat doel zelf ook TLS gebruikt, wat vrijwel altijd zo is, ontstaat er een tweede, end to end versleutelde sessie binnen de eerste. Een TLS inspecterende proxy ziet die geneste CONNECT wel, maar kan de binnenste sessie niet openbreken zonder geneste TLS terminatie, en dat kan geen standaard proxy. Microsoft adviseert daarom expliciet om `<prefix>.gw.arc.azure.com` uit te sluiten van TLS inspectie. Wie een beleid heeft dat al het uitgaande verkeer geïnspecteerd moet worden, moet hier dus een bewuste uitzondering voor maken.

![Geneste TLS: de buitenste sessie loopt van de Arc proxy naar de gateway URL, de binnenste end to end sessie naar het Azure endpoint kan een inspecterende proxy niet openen](/Images/arcgateway/arc-gateway-geneste-tls.png)

De regio die je bij het aanmaken kiest bepaalt alleen waar de resource in het control plane leeft. De gateway zelf is een globale dienst; een server in Nederland die tegen een gateway in West Europe praat, loopt niet via een langere route dan nodig.

## Wat er het afgelopen jaar is veranderd:

De Arc gateway is niet nieuw; de public preview dateert al van eind 2024. Wat wel nieuw is, is dat het inmiddels een volwassen product is dat je bij klanten in productie kunt inzetten voor alle drie de Arc resourcetypes. In een jaar tijd is de gateway van preview naar GA gegaan en heeft hij er een aantal functies bij gekregen die in de praktijk het verschil maken.

![Tijdlijn van public preview in november 2024 tot de bypass list in agent 1.64 van mei 2026](/Images/arcgateway/arc-gateway-tijdlijn.png)

| Wanneer | Wat | Status |
| --- | --- | --- |
| Juli 2026 (agent 1.66) | Arc proxy laat entries in de gateway bypass list correct buiten de gateway om lopen | GA |
| Mei 2026 (agent 1.64) | Gateway bypass list: FQDN's die je zelf opgeeft gaan via je enterprise proxy of direct, in plaats van via de gateway | GA |
| Maart 2026 | Arc gateway GA voor Arc enabled Kubernetes | GA |
| Oktober 2025 (agent 1.58) | Preview vlag van `connection.type` verwijderd | GA |
| September 2025 | Arc gateway GA voor Arc enabled servers en voor Azure Local | GA |

De bypass list is voor mij de belangrijkste toevoeging. In de eerste GA versie was het alles of niets: zat een server aan de gateway, dan ging al het Arc verkeer erdoorheen, en de klassieke `proxy.bypass` instelling van de agent werkte niet meer. Dat was lastig voor organisaties die bepaalde diensten juist bewust via hun eigen proxy willen laten lopen, bijvoorbeeld omdat daar logging of DLP op zit, of omdat een endpoint intern via een private endpoint wordt aangeboden. Sinds agent 1.64 kun je die uitzonderingen benoemen. Verderop kom ik daar op terug.

Voor Azure Local geldt één belangrijke beperking die je vooraf moet weten: de gateway kan alleen bij een nieuwe deployment (versie 2506 of hoger) worden ingeschakeld. Een bestaande Azure Local instance achteraf op de gateway zetten wordt niet ondersteund. Voor servers en Kubernetes clusters kan dat wel.

## Hands on: gateway aanmaken en servers koppelen

Je hebt de rol **Azure Arc gateway manager** nodig op de subscription of resource group waar de gateway komt. Het aanmaken zelf is één commando en duurt ongeveer tien minuten. Ik zet de gateway bij voorkeur in de connectivity of management resource group van de landing zone, naast de andere gedeelde netwerkcomponenten.

```powershell
New-AzConnectedMachineGateway -ResourceGroupName "rg-hybrid-prd-weu-01" `
    -Location "westeurope" `
    -Name "arcgw-prd-weu-01"
```

Of met de Azure CLI:

```bash
az arcgateway create --resource-group rg-hybrid-prd-weu-01 \
    --location westeurope \
    --name arcgw-prd-weu-01

az arcgateway list --query "[].{name:name, url:gatewayEndpoint}" -o table
```

Het tweede commando geeft je de gateway URL, de `<prefix>.gw.arc.azure.com` die het netwerkteam in de allowlist moet zetten. Doe dat vóórdat je de eerste server koppelt, anders gaat de agent op de gateway staan wachten en verlies je de verbinding.

### Nieuwe servers onboarden

Bij het genereren van het onboarding script in de portal kies je bij *Connectivity method* voor **Public endpoint** en selecteer je onder *Gateway resource* de zojuist aangemaakte gateway. Het script bevat dan de parameter `--gateway-id` met de volledige resource ID:

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

Heb je een enterprise proxy, dan zet je die zoals altijd met `azcmagent config set proxy.url` vóór het `connect` commando. De Arc proxy gebruikt die vervolgens als upstream.

### Controleren of het werkt

Twee commando's vertellen je alles. `azcmagent show` moet bij *Using HTTPS Proxy* de waarde `http://localhost:40343` laten zien, met daaronder je enterprise proxy als *Upstream Proxy*. En `azcmagent check` laat per endpoint zien of het bereikbaar is; bij een goed werkende gateway staat `connection.type` op `gateway` en staat elke regel op *Reachable: true*.

```powershell
azcmagent show
azcmagent check
azcmagent config get connection.type
```

De logs van de Arc proxy vind je na `azcmagent logs` in het zip bestand onder `ProgramData\AzureConnectedMachineAgent\Log\arcproxy.log` (Windows) of `/var/opt/azcmagent/log/arcproxy.log` (Linux). Dat is de eerste plek om te kijken als een extensie na de overstap niet meer wil.

### Kubernetes clusters

Voor Arc enabled Kubernetes is het patroon hetzelfde, met `--gateway-resource-id` bij `az connectedk8s connect` voor nieuwe clusters en `az connectedk8s update` voor bestaande. Met `az connectedk8s show --query gateway.enabled` controleer je of de koppeling actief is. Microsoft adviseert om na het inschakelen minimaal een uur te wachten voordat je de oude endpoints uit de firewall haalt.

## De bypass list: bewust verkeer buiten de gateway om

Sinds agent 1.64 kun je per server een lijst FQDN's opgeven die de Arc proxy niet via de gateway stuurt, maar via je enterprise proxy of rechtstreeks. Het is het spiegelbeeld van de oude `proxy.bypass` instelling: die bepaalde wat je enterprise proxy oversloeg, de gateway bypass list bepaalt wat de gateway overslaat. De twee sluiten elkaar uit, want `proxy.bypass` wordt genegeerd zodra een server op `connection.type gateway` staat.

Wanneer wil je dit? Drie situaties uit de praktijk:

1. **Private endpoints voor Log Analytics of Key Vault.** Sommige organisaties hebben een Azure Monitor Private Link Scope of een Key Vault met private endpoint, bereikbaar via ExpressRoute of VPN. Dat verkeer wil je intern houden en niet via een publieke gateway laten lopen. Zet `<workspace-id>.ods.opinsights.azure.com` of `<vault>.vault.azure.net` op de bypass list en de agent gebruikt de interne route.
2. **Logging of DLP op de enterprise proxy.** Als het securitybeleid eist dat specifieke categorieën verkeer, bijvoorbeeld alles richting storage accounts, zichtbaar blijven op de eigen proxy, hou je die endpoints buiten de gateway.
3. **Troubleshooting.** Werkt een extensie niet meer sinds de overstap naar de gateway, dan kun je het endpoint van die extensie tijdelijk op de bypass list zetten om te bepalen of de gateway het probleem is.

De bypass list stel je in met `azcmagent config set`. Omdat de exacte property naam per agentversie kan verschillen en de documentatie hier nog achterloopt, is `azcmagent config info` je vriend. Dat commando toont alle beschikbare properties voor de geïnstalleerde agentversie, inclusief de gateway gerelateerde. Controleer daarna met `azcmagent check` dat de endpoints op de bypass list inderdaad via je proxy of direct lopen en niet meer via de gateway.

Let wel: alles wat je op de bypass list zet, moet je netwerkteam weer expliciet openzetten. Elke uitzondering knabbelt aan de winst van de gateway. Gebruik de lijst dus gericht en documenteer waarom een endpoint erop staat.

Samengevat ziet de routekeuze van de agent er zo uit:

![Beslisboom: zonder gateway via de bedrijfsproxy naar Azure met volledige allowlist; met gateway gaan FQDN's op de bypass list via de bedrijfsproxy en al het overige via de Arc proxy en de gateway](/Images/arcgateway/arc-gateway-beslisboom.png)

## Wat de gateway niet afdekt:

"Eén endpoint" is de marketingversie. De eerlijke versie is: zeven endpoints voor de basis, plus een aantal extra afhankelijk van wat je op de servers draait.

![De zeven basisendpoints die altijd nodig zijn, naast de extra endpoints die extensies als Azure Monitor Agent, Update Manager en Defender buiten de gateway om vragen](/Images/arcgateway/arc-gateway-allowlist.png)

Dit is de minimale allowlist voor Arc enabled servers met gateway:

| Endpoint | Doel |
| --- | --- |
| `<prefix>.gw.arc.azure.com` | Je eigen gateway URL |
| `management.azure.com` | Azure Resource Manager |
| `login.microsoftonline.com` | Microsoft Entra ID |
| `<regio>.login.microsoft.com` | Microsoft Entra ID, regionaal |
| `gbl.his.arc.azure.com` | Arc agent cloud service |
| `<regio>.his.arc.azure.com` | Arc control channel, regionaal |
| `download.microsoft.com` of `packages.microsoft.com` | Agent installatie (Windows respectievelijk Linux) |

Voor Kubernetes komen daar `mcr.microsoft.com`, `*.data.mcr.microsoft.com` en optioneel `<regio>.obo.arc.azure.com` voor cluster connect bij. Azure Local heeft bovendien nog bootstrap endpoints, de Key Vault en het storage account voor de cloud witness, en de OCSP endpoints voor certificaatcontrole nodig.

Daarnaast zijn er extensies die gewoon hun eigen endpoints blijven eisen, gateway of niet:

| Scenario | Extra endpoints |
| --- | --- |
| Azure Monitor Agent | `<workspace-id>.ods.opinsights.azure.com` |
| Arc enabled data services | `*.ods.opinsights.azure.com`, `*.oms.opinsights.azure.com`, `*.monitoring.azure.com` |
| Key Vault certificaat sync | `<vault>.vault.azure.net` |
| Automation Hybrid Runbook Worker | `*.azure-automation.net` |
| Azure Update Manager (Windows) | Alle Windows Update endpoints |
| Microsoft Defender for Servers | Alle Defender endpoints |

Wat wél volledig via de gateway loopt zonder extra werk: SSH via Arc, Extended Security Updates en de Azure extension voor SQL Server. Juist die drie zijn bij veel klanten de reden om Arc überhaupt uit te rollen, dus dat is goed nieuws.

Er zijn nog drie randvoorwaarden om te onthouden. De gateway werkt alleen in de Azure public cloud, niet in sovereign clouds. Je mag maximaal vijf gateway resources per subscription aanmaken. En Microsoft geeft een vuistregel voor capaciteit per regio: tel (servers ÷ 20) + (Kubernetes clusters ÷ 10) + (Azure Local instances ÷ 10). Blijft dat onder de 100, dan volstaat één gateway. Voor de meeste Nederlandse organisaties is dat ruim voldoende; pas bij zo'n tweeduizend servers in één regio ga je aan een tweede gateway denken.



## Bestaande Arc servers migreren naar de gateway:

De meeste organisaties hebben al Arc servers draaien op de klassieke manier, met de volledige allowlist. Die hoef je niet opnieuw te onboarden. De koppeling aan een gateway is een eigenschap van de Azure resource, en de agent pikt die zelf op.

```powershell
Set-AzConnectedMachine -ResourceGroupName "rg-servers-prd-weu-01" `
    -Name "srv-app-01" `
    -GatewayResourceId "/subscriptions/<subscription-id>/resourceGroups/rg-hybrid-prd-weu-01/providers/Microsoft.HybridCompute/gateways/arcgw-prd-weu-01"
```

Voor grotere aantallen loop je met een `Get-AzConnectedMachine` over een resource group of een tag en roep je `Set-AzConnectedMachine` per machine aan. In de portal kan het ook: open de gateway resource, ga naar *Associated resources*, kies *Add* en selecteer de servers.

Draait er nog een agent van versie 1.50 of ouder, dan moet je daarnaast lokaal `azcmagent config set connection.type gateway` uitvoeren. Vanaf 1.51 regelt de agent dat zelf. Gezien het supportbeleid, alleen agents jonger dan een jaar worden ondersteund, is dit meteen een goed moment om oude agents bij te werken of automatische upgrades aan te zetten.

![Migratie in vier stappen: gateway aanmaken, pilot koppelen, uitrollen in batches, en pas opruimen na minimaal een dag meekijken in de proxy logs](/Images/arcgateway/arc-gateway-migratie.png)

Een aanpak die bij mij goed werkt:

1. Maak de gateway aan en laat het netwerkteam de gateway URL toevoegen aan de bestaande allowlist. Haal nog niets weg.
2. Koppel eerst een handvol servers, bij voorkeur met de zwaarste extensieset (Azure Monitor Agent, Update Manager, Defender), en controleer met `azcmagent check` en in de portal dat alle extensies gezond blijven.
3. Rol de koppeling daarna in batches uit over de rest, bijvoorbeeld per resource group of per applicatieteam.
4. Wacht minimaal een dag, controleer de proxy logs op verkeer naar de oude endpoints, en laat het netwerkteam pas dan de overbodige regels verwijderen.

Terugdraaien kan altijd. Zet `azcmagent config set connection.type direct` op de server en verwijder de koppeling met `Set-AzConnectedMachine` zonder gateway ID of via *Remove* in de portal. Vanaf dat moment gelden de volledige netwerkvereisten weer, dus haal de oude regels pas weg als je zeker weet dat je niet terug hoeft.

## Valkuilen uit de praktijk:



**TLS inspectie op de gateway URL.** Dit is met afstand de meest voorkomende oorzaak van een niet werkende gateway. De geneste TLS sessie kan niet door een inspecterende proxy heen. Symptoom: de agent staat op *Connected*, maar extensies falen bij installatie of rapporteren niet meer. Oplossing: `<prefix>.gw.arc.azure.com` uitsluiten van inspectie. Regel dit vóór de eerste koppeling, niet erna.

**Gateway URL vergeten in de allowlist.** Klinkt flauw, gebeurt toch. De onboarding slaagt vaak wel omdat `login.microsoftonline.com` en `management.azure.com` al open stonden, en de server verschijnt in de portal. Daarna gaat de agent alle overige verkeer via de gateway sturen, en dat verkeer strandt. Controleer altijd met `azcmagent check` na de koppeling.

**Te vroeg opruimen.** De verleiding is groot om direct na de migratie de oude firewall regels weg te halen. Sommige extensies praten maar eens per uur of per dag met hun endpoint. Kijk minstens een dag mee in de proxy logs en wacht bij Kubernetes het door Microsoft geadviseerde uur.

**Verwachten dat Defender en Update Manager "gratis" meegaan.** Beide hebben hun eigen endpointvereisten die buiten de gateway vallen. Wie de gateway verkoopt als "daarna hoeft er niets meer open", krijgt dat gesprek met het netwerkteam alsnog.

**Azure Local achteraf.** Voor bestaande Azure Local instances is de gateway geen optie; alleen nieuwe deployments vanaf versie 2506 ondersteunen hem. Neem dit mee in de planning als je Azure Local en Arc servers samen in één netwerkontwerp wilt onderbrengen. Het aanmaken van de gateway resource kan bij Azure Local bovendien tot twee uur duren.

**Oude agents.** Onder 1.51 moet je `connection.type` handmatig zetten, en onder 1.64 is er geen bypass list. Combineer de gateway uitrol daarom met een agent upgrade ronde. Alles ouder dan een jaar valt sowieso buiten support.

## Conclusie:

De Arc gateway lost het probleem op dat hybride cloud trajecten het vaakst vertraagt: niet de techniek, maar de allowlist. Van tientallen endpoints naar zeven, met één gateway URL die al het Arc verkeer draagt, is een verschil dat netwerkteams direct begrijpen. Nu de gateway GA is voor servers, Kubernetes en Azure Local, en met de bypass list de scherpe randjes van de eerste versie zijn weggehaald, is er weinig reden om nieuwe Arc uitrollen nog zonder gateway te doen.

Wat het niet is, is een toverstaf. Defender, Update Manager en Azure Monitor Agent houden hun eigen endpoints, TLS inspectie op de gateway URL is een harde no go, en Azure Local kan alleen bij een nieuwe deployment mee. Wie die grenzen vooraf benoemt, houdt het gesprek met het netwerkteam eerlijk en voorkomt dat de gateway later als "werkt toch niet helemaal" te boek staat.

Mijn advies voor organisaties die al Arc draaien: maak de gateway aan, koppel in batches, kijk een paar dagen mee in de logs en ruim dan pas op. Voor nieuwe uitrollen: begin er direct mee en neem de zeven basisendpoints plus de extensie endpoints als één pakket mee in de change.

## Bronnen:

* [Simplify network configuration requirements with Azure Arc gateway](https://learn.microsoft.com/en-us/azure/azure-arc/servers/arc-gateway), Microsoft Learn
* [Azure Arc gateway for Arc enabled Kubernetes](https://learn.microsoft.com/en-us/azure/azure-arc/kubernetes/arc-gateway-simplify-networking), Microsoft Learn
* [Overview of Azure Arc gateway for Azure Local](https://learn.microsoft.com/en-us/azure/azure-local/deploy/deployment-azure-arc-gateway-existing-cluster), Microsoft Learn
* [What's new with Azure Connected Machine agent](https://learn.microsoft.com/en-us/azure/azure-arc/servers/agent-release-notes), Microsoft Learn (versies 1.58, 1.64 en 1.66)
* [Announcing the General Availability of the Azure Arc gateway for Arc enabled Servers](https://techcommunity.microsoft.com/blog/azurearcblog/announcing-the-general-availability-of-the-azure-arc-gateway-for-arc-enabled-ser/4456356), Azure Arc Blog, 25 september 2025
* [Announcing the General Availability of Arc gateway for Azure Local](https://techcommunity.microsoft.com/blog/azurearcblog/announcing-the-general-availability-of-arc-gateway-for-azure-local/4456256), Azure Arc Blog, september 2025
* [Announcing the General Availability of the Azure Arc gateway for Arc enabled Kubernetes](https://techcommunity.microsoft.com/blog/azurearcblog/announcing-the-general-availability-of-the-azure-arc-gateway-for-arc-enabled-kub/4498561), Azure Arc Blog, 2 maart 2026
