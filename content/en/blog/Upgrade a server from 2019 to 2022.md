---
Author: "Andy Clifton"
title: "In-place Upgrade a Server from 2019 to 2022"
description: "Upgrade VM."
date: 2024-03-08
tags: ["Upgrade,2019,2022,VM"]
thumbnail: "/img/server2022.png"
---




## Introduction

A question I get regularly is: how can I upgrade a Windows Server to the latest version of Windows now that I cannot mount an ISO?
Microsoft has thought about this and created the option to perform an in-place upgrade on the machine itself.
Below I explain how best to carry out these steps.



### Create a Managed Disk containing the ISO.

Creating this disk is only possible via PowerShell.
So open ISE and create the PowerShell script below:

First we log in to Azure with:
{{< highlight html >}}
Connect-azaccount

{{< /highlight >}}
![Image](/Images/InplaceUpgrade/connect.JPG)

Log in and complete your MFA (if you have it set up). After that you are connected.
![Image](/Images/InplaceUpgrade/Connected.JPG)

Next we connect to the right tenant by specifying: 
{{< highlight html >}}

Set-azcontext -subscription **SubscriptionName**

{{< /highlight >}}

![Image](/Images/InplaceUpgrade/Context.JPG)

After that you can run the script below, and a Managed Disk will be created in the resource group you created. (Do not forget to fill in all the information.)

{{< highlight html >}}
# Resource group of the source VM
$resourceGroup = "" (Enter the resource group you want to use for this)
 
# Location of the source VM
$location = "" (The location of your resource group, in my case westeurope)
 
# Zone of the source VM, if any
$zone = "" (Choose zone 1, 2 or 3)
 
# Disk name for the that will be created
$diskName = "" (Give the disk any name you like)
 
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

You will then see that the disk has been created in the resource group.

![Image](/Images/InplaceUpgrade/Disk.JPG)

Go to the existing VM and go to disks:

![Image](/Images/InplaceUpgrade/Disk1.JPG)

Next we click Attach a disk and select the disk we created.

![Image](/Images/InplaceUpgrade/Attach.JPG)

Then click Apply and wait until the VM has been updated.

![Image](/Images/InplaceUpgrade/Apply.JPG)

Go to Connect and log in to this machine via RDP.

![Image](/Images/InplaceUpgrade/RDP.JPG)

Once you are logged in, open a command prompt in elevated mode, then go to the new disk and open the folder:

{{< highlight html >}}

Windows Server 2022

{{< /highlight >}} 

![Image](/Images/InplaceUpgrade/cmd.JPG)

Then enter the following to start the in-place upgrade:

{{< highlight html >}}

.\setup.exe /auto upgrade /dynamicupdate disable

{{< /highlight >}} 

The preparation of the upgrade will then start.

![Image](/Images/InplaceUpgrade/Prepare.JPG)

When the preparation is finished, you can choose which edition you want to install. Decide this based on your licenses. (Do choose the Desktop Experience.)
Then click **Next**.

![Image](/Images/InplaceUpgrade/Keuze.JPG)

The installation will start.

![Image](/Images/InplaceUpgrade/Install1.JPG)

After a while the screen turns blue and you see a percentage progressing.
 
![Image](/Images/InplaceUpgrade/Install2.JPG)

The server will then reboot a few times. After that, Windows has been upgraded to Windows Server 2022.

![Image](/Images/InplaceUpgrade/2022.JPG)

