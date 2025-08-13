---
Author: "Andy Clifton"
title: "Inplace Upgrade een Server van 2019 naar 2022"
description: "Upgrade VM."
date: 2024-03-08
tags: ["Upgrade,2019,2022,VM"]
thumbnail: "/img/server2022.png"
---




## Voorwoord

De vraag die ik regelmatig krijg is: Hoe kan ik een Windows Server upgraden naar de nieuwste versie van Windows nu ik geen iso kan mounten?
Microsoft heeft hier over nagedacht en heeft de mogelijkheid gecreëerd om een inplace upgrade uit te voeren op de machine zelf.
Hieronder leg ik verder uit hoe je deze stappen het beste kunt doen.



### Creëer een Managed Disk met de Iso hierin.

Het crëeren van deze disk kan alleen via Powershell.
Daarop open ISE en creëer onderstaande Powershell script:

Eerst loggen we in bij Azure met:
{{< highlight html >}}
Connect-azaccount

{{< /highlight >}}
![Image](/Images/InplaceUpgrade/connect.JPG)

Login en voer je MFA.(Als je dit hebt ingesteld) Daarna ben je connected.
![Image](/Images/InplaceUpgrade/Connected.JPG)

Daarna maken we connectie met de juiste tenant door aan te geven: 
{{< highlight html >}}

Set-azcontext -subscription **SubscriptionName**

{{< /highlight >}}

![Image](/Images/InplaceUpgrade/Context.JPG)

Daarna kun je onderstaand script draaien en zal er een Managed Disk aangemaakt zijn in de aangemaakte Resource group. (Vergeet niet om alle info in te vullen.)

{{< highlight html >}}
# Resource group of the source VM
$resourceGroup = "" (Vul hier de Resourcegroep in die je wilt gebruiken om de)
 
# Location of the source VM
$location = "" (De locatie van je Resource Group in mijn geval westeurope)
 
# Zone of the source VM, if any
$zone = "" (Maak een keuze uit zone 1,2 of 3)
 
# Disk name for the that will be created
$diskName = "" (Geef de disk een willekeurige naam)
 
# Target version for the upgrade - must be either server2022Upgrade, server2019Upgrade, server2016Upgrade or server2012Upgrade
$sku = "server2022Upgrade"
 
 
# Common parameters
 
$publisher = "MicrosoftWindowsServer"
$offer = "WindowsServerUpgrade"
$managedDiskSKU = "Standard_LRS"


# Get the latest version of the special (hidden) VM Image from the Azure Marketplace
 
$versions = Get-AzVMImage -PublisherName $publisher -Location $location -Offer $offer -Skus $sku | sort-object -Descending {[version] $_.Version	}
$latestString = $versions[0].Version
 
 
# Get the special (hidden) VM Image from the Azure Marketplace by version - the image is used to create a disk to upgrade to the new version
 
 
$image = Get-AzVMImage -Location $location -PublisherName $publisher -Offer $offer -Skus $sku -Version $latestString


# Create Managed Disk from LUN 0

 
if ($zone){
    $diskConfig = New-AzDiskConfig -SkuName $managedDiskSKU -CreateOption FromImage -Zone $zone -Location $location
} else {
    $diskConfig = New-AzDiskConfig -SkuName $managedDiskSKU -CreateOption FromImage -Location $location
    }
 
Set-AzDiskImageReference -Disk $diskConfig -Id $image.Id -Lun 0
 
New-AzDisk -ResourceGroupName $resourceGroup -DiskName $diskName -Disk $diskConfig
{{< /highlight >}}

{{< css.inline >}}

<style>
.emojify {
	font-family: Apple Color Emoji, Segoe UI Emoji, NotoColorEmoji, Segoe UI Symbol, Android Emoji, EmojiSymbols;
	font-size: 2rem;
	vertical-align: middle;
}
@media screen and (max-width:650px) {
  .nowrap {
    display: block;
    margin: 25px 0;
  }
}
</style>

{{< /css.inline >}}

Hierna zul je zien dat de disk is aangemaakt in de Resource Group.

![Image](/Images/InplaceUpgrade/Disk.JPG)

Ga naar de bestaande VM en ga naar disks:

![Image](/Images/InplaceUpgrade/Disk1.JPG)

Daarna klikken we op Attach een disk en selecteren de aangemaakte disk.

![Image](/Images/InplaceUpgrade/Attach.JPG)

Klik daarna op Apply en wacht tot de VM geüpdatet is.

![Image](/Images/InplaceUpgrade/Apply.JPG)

Ga naar Connect en login via RDP op deze machine.

![Image](/Images/InplaceUpgrade/RDP.JPG)

Als je bent ingelogd open je command prompt in Elevated mode, ga dan naar de nieuwe disk en open de folder:

{{< highlight html >}}

Windows Server 2022

{{< /highlight >}} 

![Image](/Images/InplaceUpgrade/cmd.JPG)

Vul daarna het volgende in om de in place upgrade te starten:

{{< highlight html >}}

.\setup.exe /auto upgrade /dynamicupdate disable

{{< /highlight >}} 

Vervolgens word de prepare van de upgrade gestart.

![Image](/Images/InplaceUpgrade/Prepare.JPG)

Als de prepare klaar is kun je een keuze maken welke versie je erop wilt zetten. Bepaal dit aan de hand van je licenties. (Kies wel de Desktop Expierence)
En klik vervolgens op **Next**.

![Image](/Images/InplaceUpgrade/Keuze.JPG)

De installatie gaat starten.

![Image](/Images/InplaceUpgrade/Install1.JPG)

Na een tijdje wordt het scherm blauw en zie je een percentage lopen.
 
![Image](/Images/InplaceUpgrade/Install2.JPG)

Daarna zal de server een paar keer rebooten.Vervolgens is Windows geüpgraded naar Windows 2022.

![Image](/Images/InplaceUpgrade/2022.JPG)

