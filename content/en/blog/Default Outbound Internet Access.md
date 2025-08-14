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

## Hoe kunen we checken welke machines afhankelijk zijn van Outbound access

Het is belangrijk om te weten welke virtuele machines in je omgeving afhankelijk zijn van outbound internettoegang, zodat je tijdig maatregelen kunt nemen. Wij gebruiken hiervoor Powershell. Met PowerShell kun je bijvoorbeeld opvragen welke VM’s een public IP-adres hebben. Door deze informatie te verzamelen, kun je gericht bepalen welke workloads extra aandacht nodig hebben bij het uitfaseren van default outbound access.  
Voer onderstaande Powershell script uit om een output te krijgen van de servers die je moet aanpassen.

````
# Login to Azure
Connect-AzAccount

# Select subscription
$subscriptionId = "<your-subscription-id>"
Select-AzSubscription -SubscriptionId $subscriptionId

# Resultaat array
$results = @()

# Get all VMs
$vms = Get-AzVM

foreach ($vm in $vms) {
    $vmName = $vm.Name
    $rgName = $vm.ResourceGroupName
    $hasPublicIp = $false
    $hasOutboundNSG = $false
    $hasInternetRoute = $false

    # Get NIC
    $nicId = $vm.NetworkProfile.NetworkInterfaces[0].Id
    $nicName = ($nicId -split "/")[-1]
    $nic = Get-AzNetworkInterface -Name $nicName -ResourceGroupName $rgName

    # Check for public IP
    foreach ($ipConfig in $nic.IpConfigurations) {
        if ($ipConfig.PublicIpAddress) {
            $hasPublicIp = $true
            $publicIpName = ($ipConfig.PublicIpAddress.Id -split "/")[-1]
            $publicIpRg = ($ipConfig.PublicIpAddress.Id -split "/")[4]
            $publicIp = Get-AzPublicIpAddress -Name $publicIpName -ResourceGroupName $publicIpRg
        }
    }


    # Check NSG outbound rules
    if ($nic.NetworkSecurityGroup) {
        $nsgName = ($nic.NetworkSecurityGroup.Id -split "/")[-1]
        $nsgRg = ($nic.NetworkSecurityGroup.Id -split "/")[4]
        $nsg = Get-AzNetworkSecurityGroup -Name $nsgName -ResourceGroupName $nsgRg
        $outboundRules = $nsg.SecurityRules | Where-Object { $_.Direction -eq "Outbound" -and $_.Access -eq "Allow" }
        if ($outboundRules.Count -gt 0) {
            $hasOutboundNSG = $true
        }
    }


    # Check route table
    $subnetId = $nic.IpConfigurations[0].Subnet.Id
    $vnetRg = ($subnetId -split "/")[4]
    $vnetName = ($subnetId -split "/")[8]
    $vnet = Get-AzVirtualNetwork -Name $vnetName -ResourceGroupName $vnetRg
    $subnetName = ($subnetId -split "/")[-1]
    $subnet = $vnet.Subnets | Where-Object { $_.Name -eq $subnetName }

    if ($subnet.RouteTable) {
        $routeTable = Get-AzRouteTable -ResourceId $subnet.RouteTable.Id
        $internetRoute = $routeTable.Routes | Where-Object { $_.AddressPrefix -eq "0.0.0.0/0" -and $_.NextHopType -eq "Internet" }
        if ($internetRoute) {
            $hasInternetRoute = $true
        }
    }

    # Output to console
    Write-Host "VM: $vmName"
    Write-Host "  ✅ Public IP: $hasPublicIp"
    Write-Host "  🔄 NSG Outbound Access: $hasOutboundNSG"
    Write-Host "  🌐 Internet Route: $hasInternetRoute"
    Write-Host "---------------------------------------------"

    # Add to results
    $results += [PSCustomObject]@{
        VMName           = $vmName
        ResourceGroup    = $rgName
        PublicIP         = $hasPublicIp
        NSGOutboundAllow = $hasOutboundNSG
        InternetRoute    = $hasInternetRoute
    }
}

# Export to CSV
$results | Export-Csv -Path "OutboundAccessReport.csv" -NoTypeInformation
Write-Host "✅ Rapport opgeslagen als 'OutboundAccessReport.csv'"

````

We hebben nu onderzocht welke machines een Public IP hebben,NSG Rules hebben of een Internet Route.
Nu moeten we de gekozen optie implementeren. Het aanmaken van een Azure Firewall zullen we later nog eens bespreken,
Public IP toevoegen spreekt voor zich, dus we gaan nu het aanmaken van een Azure NAT Gateway aanmaken.

## Aanmaken van een Azure NAT Gateway.
![Image](/Images/Outbound/Nat.png)

Azure NAT Gateway is de aanbevolen oplossing om veilige en schaalbare outbound internettoegang te bieden voor je virtuele machines en andere resources in Azure. Met NAT Gateway kun je outbound verkeer centraliseren en beheren, zodat je precies weet welke IP-adressen worden gebruikt voor internettoegang. Dit maakt het eenvoudiger om firewallregels toe te passen, monitoring in te richten en je netwerkarchitectuur te beveiligen. Daarnaast zorgt NAT Gateway voor hoge beschikbaarheid en ondersteunt het duizenden gelijktijdige verbindingen, waardoor het geschikt is voor zowel kleine als grote omgevingen. Hieronder lees je hoe je een NAT Gateway aanmaakt en configureert.
Maar hoe configureer je dit nu:

Log in op de Azure portal en klik op **Create a resource**.

![Image](/Images/Outbound/NatGateway.png)

Vul in **NAT Gateway** en klik op **Create**.

Vul in en klik op **Next**
~~~
Subscription = Subscription waar connectivity aan verbonden is
Resource Group = RG voor Connectivity

NAT Gateway Name = Naam
Region = Regio waar jullie alles op deployen. (West Europe in dit geval)
Availability Zone = Geef aan of je dit wilt gebruiken in geval van issues in het datacenter
TCP Ide timeout = Tijd tussen 4 en 120 minuten voordat er een timeout optreed en de connectie word gesloten.
~~~

![Image](/Images/Outbound/NatGateway1.png)


Nu moeten we de outbound adressen gaan configureren.

Klik eerst op **Create a new public IP address**

~~~
Name = naam voor de publieke IP.
SKU = Basic
Assignment = Static
~~~

Klik op **Ok**

![Image](/Images/Outbound/pip.png)
Vervolgens configureren we de Prefix. (dus hoeveel Public IP's kunnen er maximaal in deze NAT Gateway nog aangemaakt worden.)

Klik  op **Create a new public IP prefix**

~~~
Name = naam voor de publieke prefix.
Prefix Size= Pak de grootte die je nodig hebt. In deze test pak ik een /31 omdat ik niet meer als 2 externe adressen nodig heb.
~~~
Klik op **Ok**

![Image](/Images/Outbound/pip2.png)

klik op **Next**

We zijn nu aangekomen bij het configureren van de subnets die deze NAT gaan gebruiken.
Dus selecteer het Virtual network en daarna de subnets.

![Image](/Images/Outbound/NAT1.png)

klik op **Review+Create** en daarna op **Create**

![Image](/Images/Outbound/passed.png)

We kunnen dit ook opbouwen via Powershell dat doe je via onderstaande Powershell script.

````
# Set the variables for the NAT Gateway.
$rg = "RG-Andy-Test2"
$Location = "Westeurope"
$sku = "Standard"
$PublicIpname = "pip"
$Publicprefixname = "PIPPrefix"
$NatGatewayname="NATGateway"


#create Rsource group
New-AzResourceGroup -Name $rg -Location $Location 

#create Standard SKUP public IP
$publicIP = New-AzPublicIpAddress -Name $PublicIpname -ResourceGroupName $rg -AllocationMethod Static -Location $Location -Sku $sku

#create  IP prefix
$publicIPPrefix = New-AzPublicIpPrefix -Name $Publicprefixname -ResourceGroupName $rg -Location $Location -PrefixLength 29

#Create NAT gateway
$natGateway = New-AzNatGateway -Name $NatGatewayname -ResourceGroupName $rg -PublicIpAddress $publicIP -PublicIpPrefix $publicIPPrefix -Location $Location -Sku $sku -IdleTimeoutInMinutes 4
$natGateway  | Select-Object Name, ResourceGroupName, IdleTimeoutInMinutes , SKuText | Format-table -autosize –wrap

$virtualNetwork = Get-AzVirtualNetwork | Out-GridView -PassThru -Title "Kies je VNET"
$NATSubnet = Get-AzVirtualNetworkSubnetConfig -VirtualNetwork $virtualNetwork | Out-GridView -PassThru -Title "Kies de Subnets"

````

Nu is de NAT Gateway gedeployed en geconfigureerd het enige wat nu nog moet gebeuren, is inloggen op de VM en testen of outbound verkeer werkt.

## Wil je nu al zorgen voor een uniforme aanpak? Pak dan meteen de standaard internettoegang in bestaande VNets aan.

Dat doe je zo:

🔒 *Blokkeer ongewenst internetverkeer*
Maak een outbound NSG-regel die al het verkeer naar het internet tegenhoudt — behalve via jouw gekozen route.

🚧 *Stuur het verkeer bewust de juiste kant op*
Koppel expliciet een NAT Gateway of Firewall aan je VNet en zorg dat andere routes verdwijnen uit het plaatje.

Zo houd je de touwtjes stevig in handen en voorkom je dat verkeer stiekem via de achterdeur naar buiten glipt.

