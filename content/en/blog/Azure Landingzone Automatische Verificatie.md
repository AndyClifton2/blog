---
Author: "Andy Clifton"
title: "Azure Landingzone verificatie met PowerShell (CAF Checklist)"
description: "Automatisch controleren of jouw Azure CAF-implementatie volledig en correct is."
date: 2026-05-01
tags: ["Azure", "PowerShell", "CAF", "Terraform", "Governance"]
thumbnail: "/img/caf-thumbnail.png"
---

## Voorwoord

Veel organisaties kiezen ervoor om hun Azure-omgeving in te richten volgens het **Cloud Adoption Framework (CAF)** van Microsoft. Dat is een verstandige keuze: CAF biedt een bewezen structuur voor governance, security, netwerk en identiteitsbeheer. Maar er zit een addertje onder het gras.

De inrichting zelf — via Terraform, Bicep of handmatig — is slechts de eerste stap. De echte vraag is: *staat alles ook echt zoals het hoort?* Zijn alle policies actief? Zijn de juiste RBAC-rollen aanwezig? Draait de firewall correct? Zijn de DNS-zones aangemaakt?

Handmatig door de Azure Portal klikken om dit te controleren is tijdrovend, foutgevoelig en — boven alles — niet herhaalbaar. Elke keer dat je een omgeving uitrolt of een wijziging doorvoert, begin je opnieuw.

Dit artikel beschrijft een aanpak waarbij een PowerShell-script die verificatierol overneemt. Het script doorloopt automatisch 30+ controlepunten, verdeeld over tien domeinen, en geeft per checkpunt een duidelijk ✅, ❌ of ⚠️ met een verklarende detail-regel. De namen van de checks in dit artikel verwijzen naar de CAF-referentiecode die je in het script terugvindt — zodat je eenvoudig kunt navigeren tussen blog en script.

Het script is generiek opgezet: je past één configuratieblok bovenaan aan voor jouw eigen omgeving, en de rest werkt ongeacht de naam van je organisatie of de specifieke resourcenamen die je gebruikt.


## <ins>Hoe is het script opgebouwd?</ins>

Het script bestaat uit drie lagen: een configuratieblok, een set helper-functies, en de eigenlijke checks per domein.

**Configuratieblok**

Bovenaan het script staat een blok met variabelen die je eenmalig aanpast per omgeving. Denk aan je Tenant ID, de Subscription IDs voor je platform-subscriptions, en de namen van je resource groups. Dit is de enige plek die je hoeft aan te raken:

```powershell
$TenantId           = "<jouw-tenant-id>"
$ConnectivitySubId  = "<subscription-id-connectivity>"
$ManagementSubId    = "<subscription-id-management>"
$RootMgName         = "<naam-van-jouw-root-management-group>"
$HubResourceGroup   = "<naam-van-jouw-hub-resource-group>"
$DnsResourceGroup   = "<naam-van-jouw-dns-resource-group>"
```

**Helper-functies**

De helper-functies zorgen voor herbruikbaarheid en consistentie door het hele script:

- `Write-CheckResult` — toont het resultaat van elke check (✅/❌/⚠️) inclusief referentiecode en details, en houdt de tellers bij voor de eindrapportage
- `Find-ManagementGroup` — zoekt Management Groups op zowel technische `GroupId` als de `DisplayName` die je in de portal ziet, zodat naamverschillen geen valse negatieven opleveren
- `Get-AllMGs` — haalt alle Management Groups eenmalig op en slaat ze op in een cache, zodat het script niet tientallen keren dezelfde Azure API aanroept
- `Test-PolicyAssignmentExists` — controleert of een specifieke policy daadwerkelijk is toegewezen op een bepaalde scope

**Pre-flight check**

Voordat het script ook maar één check uitvoert, verifieert het of je actief bent ingelogd op Azure. Zo voorkom je cryptische foutmeldingen halverwege:

```powershell
$context = Get-AzContext -ErrorAction SilentlyContinue
if (-not $context) {
    Write-Host "⚠️  Niet ingelogd bij Azure. Voer eerst Connect-AzAccount uit."
    exit 1
}
```

Dit klinkt vanzelfsprekend, maar in geautomatiseerde pipelines is een verlopen token één van de meest voorkomende oorzaken van stille fouten.


## <ins>1. Governance & Management Groups</ins>

Management Groups zijn de ruggengraat van elke CAF-implementatie. Ze vormen de hiërarchische structuur waarop je policies, RBAC-rollen en budgetten toepast — en ze bepalen hoe ver die instellingen "zakken" naar onderliggende subscriptions.

Zonder de juiste Management Group-structuur kun je geen consistente governance afdwingen. Policies die je op de root-level toepast, gelden automatisch voor alle onderliggende groepen en subscriptions. Dat is precies de kracht van dit model — maar ook de reden waarom de structuur correct moet zijn.

De CAF-standaard schrijft voor dat je minimaal de volgende groepen aanmaakt onder je root Management Group:

```
Tenant Root Group
└── [Jouw organisatienaam] (root MG)
    ├── Platform
    ├── Landing Zones
    ├── Sandbox
    └── Decommissioned
```

**Waarom deze verdeling?**

- **Platform** bevat de subscriptions voor gedeelde infrastructuur: netwerk, identiteit, beveiliging en monitoring. Dit zijn de resources waarvan alle workloads afhankelijk zijn.
- **Landing Zones** bevat de subscriptions voor de daadwerkelijke workloads van de organisatie, opgesplitst per applicatie of team.
- **Sandbox** is een geïsoleerde speeltuin voor experimenten, expliciet losgekoppeld van productie. Peering met andere omgevingen is hier geblokkeerd via policy.
- **Decommissioned** bevat subscriptions die worden uitgefaseerd. Door ze niet direct te verwijderen maar te verplaatsen, houd je audit trails intact.

<ins>G01 – Verificatie van de MG-structuur</ins>

Het script zoekt alle verwachte Management Groups op — zowel op `GroupId` als op `DisplayName`. Dit onderscheid is belangrijk: Azure toont in de portal soms een andere naam dan de technische identifier die in de API wordt gebruikt.

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

<ins>G05 – Maximaal 2 Owners op de Tenant Root Group</ins>

De Tenant Root Group is het hoogste niveau in je Azure-tenant. Wie hier Owner is, heeft in principe toegang tot alles. Dat mogen uitsluitend zogeheten *break-glass accounts* zijn — noodaccounts die je alleen gebruikt als alle andere toegang is mislukt. Het script controleert dat er niet meer dan twee Owner-assignments op dit niveau staan.

<ins>G06 – Alleen audit-policies op root-niveau</ins>

Op de root Management Group mogen uitsluitend *audit-only* policies staan — geen Deny-policies. De reden: Deny-policies op het hoogste niveau kunnen onverwacht workloads breken in subscriptions die je nog niet volledig in beeld hebt. Deny-beleid hoort thuis op het niveau van de Landing Zones MG of lager, waar je volledige controle hebt over de impact.

<ins>G15 & G16 – Key Vaults en Resource Locks op platform-resources</ins>

Elke platform-subscription heeft een Key Vault nodig voor het beheer van secrets, certificaten en sleutels. Daarnaast controleert het script of er `CanNotDelete`-locks op de kritieke platform resource groups staan. Deze locks voorkomen dat iemand per ongeluk — of met een fout Terraform-commando — de netwerk- of logging-infrastructuur verwijdert waarvan alle workloads afhankelijk zijn.


## <ins>2. RBAC & Toegangsbeheer</ins>

Role-Based Access Control (RBAC) bepaalt wie wat mag doen in jouw Azure-omgeving. Een veelgemaakte fout is om RBAC te beschouwen als een eenmalige configuratie — iets wat je instelt bij de initiële inrichting en daarna niet meer aanraakt. In de praktijk sluipt er zonder actief beheer al snel *privilege creep* in: accounts die meer rechten hebben dan ze nodig hebben, of rechten die permanent zijn terwijl ze tijdelijk zouden moeten zijn.

Het script controleert drie aspecten van RBAC:

<ins>G08 – Beheerders op de root Management Group</ins>

Op de root Management Group moeten Contributor-rollen zijn toegewezen aan de beheerders van de omgeving. Ontbreken die, dan heeft niemand de juiste rechten om de platform-infrastructuur te beheren. Het script controleert of er minimaal één Contributor-assignment bestaat op dit niveau.

<ins>G31 – Privileged Identity Management (PIM)</ins>

Admin-rollen horen nooit permanent te zijn. Met Entra PIM (Privileged Identity Management) maak je rollen *eligible* in plaats van actief: een beheerder vraagt tijdelijk toegang aan, geeft een reden op, en de rol wordt voor een beperkte tijd geactiveerd. Dit verkleint het aanvalsoppervlak enorm — een gecompromitteerd account heeft zonder actieve PIM-activatie geen admin-rechten.

Het script controleert via `Get-AzRoleEligibleChildResource` of er PIM-eligible assignments zijn geconfigureerd. Lukt de query niet door te lage rechten of een verouderde module, dan geeft het script een ⚠️ skip — geen onterechte ❌.

> **Let op:** PIM vereist een Entra ID P2-licentie. Wie die licentie niet heeft, loopt een reëel risico: permanente admin-rechten zonder tijdslimiet of audittrail.

<ins>G34/35 – Drie standaard RBAC-groepen per subscription</ins>

Op elke subscription worden drie rollen verwacht: Owner, Contributor en Reader. Dit klinkt minimaal, maar het is de basisstructuur die het mogelijk maakt om toegang gelaagd te regelen. Ontbreekt één van de drie, dan is de RBAC-structuur incompleet en moet je handmatig toewijzen — wat foutgevoelig is en slecht te auditen.

```powershell
$missing = @("Owner","Contributor","Reader") | Where-Object { $_ -notin $foundRoleNames }
```


## <ins>3. Azure Policy & Compliance</ins>

Azure Policy is het mechanisme waarmee je afdwingt dat resources voldoen aan jouw organisatiestandaarden — automatisch, op schaal, zonder handmatige controle. Het verschil met RBAC is subtiel maar belangrijk: RBAC regelt *wie* iets mag doen, Policy regelt *wat* er überhaupt mag bestaan.

Denk aan voorbeelden als: alle resources moeten een bepaalde tag hebben, publieke endpoints zijn niet toegestaan, of VM-schijven moeten versleuteld zijn. Zonder policies vertrouw je erop dat mensen de juiste keuzes maken. Met policies dring je het af.

<ins>G22 – Microsoft Defender for Cloud</ins>

Defender for Cloud is het centrale security-platform van Azure. Het combineert vulnerability management, threat protection en compliance monitoring in één dashboard. Het is verdeeld in meerdere plans per resourcetype — en elk plan dat op de gratis *Free*-tier staat, biedt geen actieve bescherming.

Het script controleert welke plans actief zijn en welke nog op Free draaien:

```powershell
$defenderPlans = Get-AzSecurityPricing -ErrorAction SilentlyContinue
$activePlans   = $defenderPlans | Where-Object { $_.PricingTier -ne "Free" }
$g22ok         = ($activePlans).Count -gt 0
```

Activeer minimaal de plans voor `VirtualMachines`, `SqlServers`, `StorageAccounts` en `KeyVaults`. Dit zijn de resourcetypen waarop aanvallers het vaakst inzetten.

<ins>G23 – Custom Policy Initiatives</ins>

Een *policy initiative* is een verzameling van meerdere policies die je als één pakket toewijst. De CAF-standaard schrijft voor dat je minimaal vier initiatives aanmaakt:

- **Tagging** — afdwingen dat alle resources de vereiste tags bevatten (bijv. `Environment`, `Owner`, `CostCenter`)
- **Landing Zone** — basisregels voor workload-subscriptions, zoals het verbieden van bepaalde VM-groottes of regio's
- **Platform Connectivity** — regels voor netwerkconfiguratie, zoals het verbieden van VPN-gateways buiten de connectivity-subscription
- **Security** — baseline security-regels, zoals het verplichten van HTTPS en versleuteling

Het script controleert of deze initiatives aanwezig zijn op de juiste Management Group-scope.

<ins>G18 – Deny publieke endpoints op Landing Zones</ins>

Op de Landing Zones Management Group moet een *Deny*-policy actief zijn die het aanmaken van publieke endpoints blokkeert. Zonder deze policy kan een ontwikkelaar een Storage Account of Key Vault aanmaken met publieke toegang — ook al is dat niet de bedoeling. De policy maakt het onmogelijk, niet alleen onwenselijk.

<ins>G13 – Log Analytics retentie</ins>

Logdata is waardeloos als je hem niet lang genoeg bewaart. Voor incident response en forensisch onderzoek heb je minimaal 30 dagen nodig — bij voorkeur 90 dagen of meer. Het script controleert de retentie-instelling op alle Log Analytics Workspaces en rapporteert welke onder de grens zitten.


## <ins>4. Connectivity & Netwerk</ins>

Het netwerkfundament van een CAF-omgeving bepaalt hoe verkeer stroomt: tussen workloads onderling, naar on-premises, en naar het internet. Een verkeerde netwerkinrichting heeft directe gevolgen voor security — niet als theorie, maar als aanvalsoppervlak.
Hub & Spoke of Virtual WAN?
Voor de meeste organisaties is Hub & Spoke de aanbevolen netwerktopologie binnen CAF. In dit model beheer je zelf een centraal hub-VNet, koppel je spoke-VNets via VNet-peering, en plaats je gedeelde services zoals Azure Firewall, DNS en VPN Gateway in de hub. Je hebt volledige controle over de configuratie, de kosten zijn voorspelbaar, en de opzet is goed te begrijpen en te beheren — ook voor kleinere teams.

Virtual WAN (vWAN) is een alternatief dat Microsoft aanbiedt voor organisaties met complexere vereisten: meerdere regio's, grote aantallen branch-locaties, of de wens om het routebeheer volledig over te laten aan het Azure-platform. vWAN beheert routing automatisch en schaalt moeiteloos, maar brengt hogere kosten met zich mee en minder granulaire controle over de netwerkinrichting.
De keuze tussen Hub & Spoke en Virtual WAN hangt af van de schaal, het beheermodel en de budgetruimte van de organisatie. Er is geen universeel juist antwoord — wél een juist antwoord voor jouw situatie.
In het geval van dit voorbeeld koos de klant bewust voor Virtual WAN. De omgeving omvat meerdere locaties en de klant wilde routebeheer niet handmatig onderhouden. Het script verifieert daarom specifiek de vWAN-componenten. Werk je zelf met Hub & Spoke, dan zul je de connectivity-checks in het script aanpassen naar de relevante VNet- en peering-resources.

<ins>C02 – Virtual WAN & Secured Hub</ins>

Het script verifieert dat zowel de Virtual WAN-resource als de Secured Hub aanwezig zijn in de Hub resource group. Ontbreekt de hub, dan heeft de rest van de netwerkarchitectuur geen fundament.

<ins>C05 – Platform VNets</ins>

Minimaal drie platform-VNets worden verwacht:

- **Connectivity** — voor de WAN-koppeling en externe verbindingen
- **PrivateEndpoints** — voor het centraal hosten van private endpoint DNS-koppelingen
- **Management** — voor beheertoegang en monitoring-resources

De reden voor aparte VNets is *segmentatie*: een probleem in één VNet beïnvloedt de anderen niet automatisch.

<ins>C10 – Azure Firewall Policy</ins>

Een Azure Firewall zonder Policy is een firewall die alles doorlaat of alles blokkeert — er zijn geen regels. Een Firewall Policy is het document waarin je de regels definieert: welk verkeer is toegestaan, welk wordt geblokkeerd, en wat wordt gelogd. Het script controleert of er een policy aanwezig is in de Hub resource group.

<ins>C11 – Blokkade van spoke-to-spoke peering</ins>

In een vWAN-topologie loopt al het verkeer via de hub. Directe peering tussen spokes omzeilt de firewall en het centrale verkeersbeleid. Het script controleert of er een policy actief is die spoke-to-spoke peering blokkeert.

<ins>C12 – Internet breakout via Azure Firewall</ins>

Al het uitgaande internetverkeer moet via de Azure Firewall in de hub lopen. Dit heet een *forced internet breakout*. Zonder dit kunnen workloads direct communiceren met het internet, zonder inspectie of logging. Het script controleert of de Virtual WAN Hub, de Azure Firewall én Routing Intent aanwezig zijn:

```powershell
$vhubs         = Get-AzVirtualHub -ResourceGroupName $HubResourceGroup
$afw           = Get-AzResource -ResourceType "Microsoft.Network/azureFirewalls" `
                     -ResourceGroupName $HubResourceGroup
$routingIntent = Get-AzResource `
                     -ResourceType "Microsoft.Network/virtualHubs/routingIntent"

$c12ok = ($vhubs).Count -gt 0 -and ($afw).Count -gt 0
```

Routing Intent is de schakel die vWAN instrueert om al het verkeer via de firewall te sturen. Zonder Routing Intent heeft de firewall wel regels, maar ziet hij het verkeer niet.


## <ins>5. DNS & Private Endpoints</ins>

DNS is de stille infrastructuur die bepaalt of private connectivity daadwerkelijk privaat is. Een veelgemaakte fout: private endpoints aanmaken, maar de bijbehorende DNS-configuratie vergeten. Het gevolg: resources die via de public DNS worden bereikt in plaats van via de private IP — waardoor verkeer onbedoeld buiten het privénetwerk loopt.

<ins>C07 – DNS Private Resolver</ins>

In hybride omgevingen — waar on-premises systemen via ExpressRoute of VPN verbinding maken met Azure — is een **DNS Private Resolver** nodig. Deze verzorgt *conditional forwarding*: DNS-queries voor `privatelink.*`-domeinen worden doorgestuurd naar Azure's private DNS, zodat on-premises clients de private IP-adressen van Azure-resources kunnen resolven.

Ontbreekt de Private Resolver, dan kunnen on-premises systemen private endpoints simpelweg niet bereiken.

<ins>C09a – Verplichte Private DNS Zones</ins>

Voor elke Azure-service die via private endpoints wordt aangeboden, moet een bijbehorende Private DNS Zone aanwezig zijn. Het script controleert minimaal vier zones die in vrijwel elke omgeving voorkomen:

```powershell
$requiredZones = @(
    "privatelink.blob.core.windows.net",
    "privatelink.file.core.windows.net",
    "privatelink.vaultcore.azure.net",
    "privatelink.database.windows.net"
)
$missingZones = $requiredZones | Where-Object { $dnsZonesCheck.Name -notcontains $_ }
```

Ontbreekt een zone, dan valt het private endpoint voor die service buiten DNS-resolutie. Afhankelijk van de netwerkconfiguratie leidt dat tot één van twee problemen: de resource is onbereikbaar, of de client valt terug op de publieke DNS — en maakt zo alsnog een publieke verbinding.

<ins>C08 – Private Endpoints op platform-resources</ins>

Het script verifieert ook of platform-resources zelf via private endpoints bereikbaar zijn — niet via publieke internet-endpoints. Dit geldt met name voor Key Vaults en Storage Accounts in de platform-subscriptions.


## <ins>6. Security & Monitoring</ins>

Een Azure-omgeving zonder goede monitoring is een omgeving waarvan je pas achteraf leert dat er iets is misgegaan. Security en observability zijn geen afzonderlijke disciplines — ze zijn twee kanten van dezelfde medaille.

<ins>S01 – Security policies via EPAC</ins>

**EPAC** staat voor *Enterprise Azure Policy as Code* — een Microsoft-open-source framework voor het beheren van Azure Policy als versie-gecontroleerde code. Het script controleert of er EPAC-gebaseerde security policies actief zijn op de root Management Group. Dit omvat minimaal audit-policies voor compliance en security-baselines.

Waarom EPAC? Omdat handmatig beheerde policies in de portal niet auditeerbaar zijn, niet herhaalbaar zijn, en snel uit sync raken met de werkelijkheid. Policy-as-code lost dat op.

<ins>IAM-3 – User-assigned Managed Identities</ins>

Service principals met wachtwoorden zijn een risico: wachtwoorden verlopen, worden gedeeld, of lekken uit. **Managed Identities** zijn het alternatief: een Azure-resource krijgt een identiteit die automatisch wordt beheerd door het platform, zonder wachtwoord of secret. Het script controleert of er user-assigned Managed Identities aanwezig zijn als signaal van een volwassen identity-strategie.

<ins>M03 – Twee Log Analytics Workspaces</ins>

Het scheiden van infrastructuur-logging en security-logging is een bewuste architectuurkeuze. Infrastructuurlogs bevatten performance- en diagnostische data die breed toegankelijk mag zijn. Security-logs bevatten gevoelige informatie — loginpogingen, policy-wijzigingen, privileged access — die alleen beschikbaar mag zijn voor het security-team. Door ze in aparte workspaces te bewaren, kun je toegangsrechten per workspace regelen.

Het script verwacht minimaal twee workspaces in de Management subscription: één voor infra, één voor security (gekoppeld aan Sentinel).

<ins>M09/10 – AMBA Alert Rules</ins>

**AMBA** staat voor *Azure Monitor Baseline Alerts* — een door Microsoft gepubliceerde set aanbevolen drempelwaarden voor standaard Azure-resources. Denk aan alerts op hoge CPU-gebruik, ophopende queues, Key Vault-fouten, of onverwachte uitval van VPN Gateways. Het script controleert zowel *Metric Alerts* (gebaseerd op meetwaarden) als *Activity Log Alerts* (gebaseerd op acties, zoals het verwijderen van een resource).

<ins>G14 – Action Groups</ins>

Een alert zonder Action Group is een stil alarm. Een Action Group definieert *wat er gebeurt* als een alert afgaat: een e-mail sturen, een webhook aanroepen, een runbook starten. Het script controleert of er minimaal één Action Group geconfigureerd is. Zonder Action Group genereren je alerts niets actiefs.


## <ins>7. Landing Zones & Business Continuity</ins>

De voorgaande secties draaien om de platform-infrastructuur — de gedeelde fundament waarop workloads draaien. Deze laatste sectie kijkt naar de landing zones zelf: de omgevingen waar applicaties en services daadwerkelijk worden gehost.

<ins>LZ02 – Prod én Non-Prod per landing zone</ins>

Elke applicatie of team krijgt een eigen landing zone, met twee Management Groups: Prod en Non-Prod. Dit maakt het mogelijk om per omgeving verschillende policies toe te passen. In Prod wil je strikte beperkingen; in Non-Prod meer ruimte voor experimenten. Het script zoekt recursief door de MG-hiërarchie om te verifiëren dat deze splitsing aanwezig is.

<ins>LZ-std – Standaard resources per landing zone</ins>

Elke landing zone moet minimaal twee resources bevatten:

- Een **Log Analytics Workspace** — zodat applicatie-logs van workloads in deze landing zone centraal worden verzameld
- Een **Key Vault** — voor het veilig opslaan van secrets, API-sleutels en certificaten die workloads in deze landing zone nodig hebben

Door dit als standaard af te dwingen, voorkom je dat teams secrets in code zetten of logs nergens naartoe sturen.

<ins>B22 – Zone-Redundant Storage (ZRS)</ins>

Storage Accounts die op LRS (Locally Redundant Storage) draaien, slaan data op in één datacenter. Valt dat datacenter uit — door hardware of een zoneprobleem — dan is de data tijdelijk niet beschikbaar. ZRS (Zone-Redundant Storage) repliceert data over drie availabilityzones binnen dezelfde regio, waardoor dit risico wordt geëlimineerd. Het script controleert alle Storage Accounts en rapporteert welke nog op LRS of GRS draaien:

```powershell
$nonZrsSAs = $allSAs | Where-Object { $_.Sku.Name -notlike "*ZRS*" }
$b22ok     = ($allSAs).Count -gt 0 -and ($nonZrsSAs).Count -eq 0
```

<ins>B15-18 – Resource Locks op kritieke resources</ins>

`CanNotDelete`-locks op de DNS- en Hub resource groups zijn een simpele maar effectieve maatregel. Ze voorkomen dat een beheerder — of een foutief Terraform-commando met `destroy` — de netwerk- of DNS-infrastructuur verwijdert waarvan alle workloads afhankelijk zijn. Een lock kost niets en voorkomt potentieel uren aan herstelwerk.


## <ins>Het script uitvoeren</ins>

Het script heeft minimale vereisten en draait op elke machine waar Azure PowerShell beschikbaar is.

**Vereisten:**

- Az PowerShell module: `Install-Module Az -Force`
- Azure CLI (optioneel, voor budget-checks): `winget install Microsoft.AzureCLI`
- Rechten: minimaal *Reader* op Management Group scope + *Security Reader* op de subscriptions

**Uitvoeren:**

```powershell
# Stap 1 — inloggen
Connect-AzAccount

# Stap 2 — pas het configuratieblok bovenin het script aan voor jouw omgeving

# Stap 3 — uitvoeren
.\CAF.ps1
```

**Wat je te zien krijgt:**

Per check verschijnt een regel met het resultaat, de referentiecode en een detail-melding:

```
  ✅  G01    Management group structuur aanwezig
       → Alle MG's gevonden: Platform | Landing Zones | Sandbox | Decommissioned

  ❌  G06    Audit-only policies op root MG
       → Geen audit-only policies gevonden op scope [jouw-root-mg]

  ⚠️  G31    Admin-rollen via PIM
       → Onvoldoende rechten voor PIM-query of Az.Resources module te oud (vereist 6.x+)
```

Aan het einde verschijnt een samenvatting:

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

Rode ❌-items bevatten altijd een detail-regel (→) die exact aangeeft wat er ontbreekt of niet klopt. Begin daar bij het oplossen van bevindingen.


## Tot slot

CAF is geen doel op zich — het is een middel om Azure op een beheersbare, veilige en schaalbare manier in te richten. De waarde ervan zit niet in het volgen van het framework, maar in het consequent handhaven ervan.

Dat is precies waar dit script bij helpt. Het verifieert niet of je ooit de juiste dingen hebt gedaan, maar of ze *nu* nog correct staan. Configuraties driften. Mensen maken wijzigingen. Terraform runs overschrijven soms onbedoeld bestaande instellingen. Een verificatiescript dat je na elke deployment — of periodiek als scheduled task — uitvoert, geeft je de zekerheid dat de werkelijkheid overeenkomt met de bedoeling.

Wie Azure serieus inricht, doet er goed aan om niet alleen te vragen "Hebben we alles gedeployed?", maar vooral "Staat het ook correct, volledig en aantoonbaar?"

Dit script helpt bij die laatste vraag.

Het volledige script is beschikbaar op GitHub: [CAF.ps1 →](https://github.com/AndyClifton2/Scripts/tree/master/CAF)

## Azure-governance is pas effectief als je hem ook actief verifieert. Niet eenmalig, maar structureel.
