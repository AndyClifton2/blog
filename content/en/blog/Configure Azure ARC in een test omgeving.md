---
Author: "Andy Clifton"
title: "Configure Azure ARC in a test environment"
description: "Test environment Azure Arc."
date: 2024-01-03
tags: ["ARC"]
thumbnail: "/img/arc.jpeg"
---




 
# Configure Azure ARC in a test environment

### ==> Disclaimer: work in progress. <==

## Foreword

Sometimes it is handy to quickly spin up a test environment so you can try things out. However, it is not always possible to set up an entire Hyper-V environment.
How do you do that quickly? What do you need for it? And can this be done in Azure? These are questions I get asked every now and then.
In this blog I will explain how you can build a Hyper-V environment in Azure and how to install Azure ARC on it afterwards, so you can manage the machines through the Azure Portal.

<ins>What is Azure Arc?:</ins>

With Azure Arc you can manage Windows and Linux machines that are hosted outside Azure from within Azure. This means you can use Azure services on physical servers or on servers running in other cloud providers' services.

Azure Arc became generally available in September 2020.


## Build a new server that is suitable for Hyper-V (Portal)

We are now going to build a new machine through the portal that meets the requirements for installing Hyper-V. (Many machines in Azure do not have the right specs to run a Hyper-V environment.)

Go to http://portal.azure.com and log in with credentials that allow you to build a machine.

Go to **Create a Resource.**
![alt](/Images/AzureArc/Resource.JPG)



Then search for **Windows Server** and click **Create**

![Image](/Images/AzureArc/WindowsServer.jpg)

Then choose **Windows Server 2019 Datacenter** and press **Create**


![Image](/Images/AzureArc/WindowsServer1.jpg)


-----
Next, fill in the details below:


~~~
Subscription = "de subscription die je gebruikt om machines te installeren"
Resource Group = selecteer een bestaande Resource group of maak een nieuwe aan.
Virtual Machine name = HV01
Region = kies de regio die het dichtst bij je zit. In ons geval is dat West-Europe.
Availablility Options = No Infrastructure redundancy required (kies niet hiervoor als je een live omgeving ervan gaat maken)
Security Type = Trusted launch virtual machines
Image = Windows Server 2019 Datacenter - x64 Gen1 (let op dit Moet **GEN1** zijn anders werkt het niet)
Size = Dit moet een V3 versie zijn om Hyper-V te kunnen installeren. Dus de Stansdard_D8s_v3

Username = Zelf in te vullen
Password = Zelf in te vullen

De rest mag default blijven
~~~
![Image](/Images/AzureArc/WindowsServer2.jpg)

Click **Review & Create** 

After the review you can click **Create**

![Image](/Images/AzureArc/Deploy.JPG)

After a while the machine is ready and we are going to connect and log in to it.

Go to the resource we just created and click **Connect**.

![Image](/Images/AzureArc/connect.JPG)


If you have configured a different method than Native RDP, connect that way. Otherwise click **select** under Native RDP and an rdp file will be downloaded.


![Image](/Images/AzureArc/RDP.jpg)


Open the RDP file and log in to the new server with the admin credentials you entered earlier.


## Build a new server that is suitable for Hyper-V (PowerShell)


~~~~
# Variables
$resourceGroupName = "YourResourceGroupName"
$vmName = "YourVMName"
$location = "East US" # Change this to the desired Azure region
$adminUsername = "YourAdminUsername"
$adminPassword = "YourAdminPassword"
$vmSize = "Standard_D8s_v3" (nodig voor een Gen1 machine)
$imagePublisher = "MicrosoftWindowsServer"
$imageOffer = "WindowsServer"
$imageSku = "2019-Datacenter"
$nicName = "YourNICName"
$vnetName = "YourVNetName"
$subNetName = "YourSubnetName"

# Create a new resource group
New-AzResourceGroup -Name $resourceGroupName -Location $location

# Create a new virtual network
$subnet = New-AzVirtualNetworkSubnetConfig -Name $subNetName -AddressPrefix "10.0.1.0/24"
$vnet = New-AzVirtualNetwork -ResourceGroupName $resourceGroupName -Name $vnetName -Location $location -AddressPrefix "10.0.0.0/16" -Subnet $subnet

# Create a public IP address
$publicIp = New-AzPublicIpAddress -ResourceGroupName $resourceGroupName -Name "YourPublicIP" -AllocationMethod Dynamic -Location $location

# Create a network security group (optional)
$nsg = New-AzNetworkSecurityGroup -ResourceGroupName $resourceGroupName -Name "YourNSG" -Location $location

# Allow RDP traffic (optional)
$rdpRule = New-AzNetworkSecurityRuleConfig -Name "Allow-RDP" -Description "Allow RDP" -Access Allow -Protocol Tcp -Direction Inbound -Priority 100 -SourceAddressPrefix "*" -SourcePortRange "*" -DestinationAddressPrefix "*" -DestinationPortRange 3389
$nsg | Add-AzNetworkSecurityRuleConfig -NetworkSecurityRule $rdpRule
$nsg | Set-AzNetworkSecurityGroup

# Create a network interface
$nic = New-AzNetworkInterface -Name $nicName -ResourceGroupName $resourceGroupName -Location $location -SubnetId $vnet.Subnets[0].Id -PublicIpAddressId $publicIp.Id -NetworkSecurityGroupId $nsg.Id

# Create the virtual machine configuration
$vmConfig = New-AzVMConfig -VMName $vmName -VMSize $vmSize
$vmConfig = Set-AzVMOperatingSystem -VM $vmConfig -Windows -ComputerName $vmName -Credential (New-Object PSCredential -ArgumentList $adminUsername, (ConvertTo-SecureString -AsPlainText -Force $adminPassword))

$vmConfig = Set-AzVMSourceImage -VM $vmConfig -PublisherName $imagePublisher -Offer $imageOffer -Skus $imageSku -Version "latest"
$vmConfig = Add-AzVMNetworkInterface -VM $vmConfig -Id $nic.Id

# Create the VM
New-AzVM -ResourceGroupName $resourceGroupName -Location $location -VM $vmConfig


~~~~


Log in to your machine.

![Image](/Images/AzureArc/Login.JPG)




## Install Hyper-V and install Windows 2019 on a Hyper-V machine.

Once you are logged in to the machine, we will start with the installation of the Hyper-V features.
Open Server Manager, click **Manage** and then **Add Roles and Features**

![IMage](/Images/AzureArc/HyperV1.JPG)

On the Before you begin screen, click **Next**

![Image](/Images/AzureArc/HyperV2.JPG)

Select **Role-Based or Feature-Based Installation**

![Image](/Images/AzureArc/HyperV3.JPG)

You can now choose which server you want to perform the installation on. We want to install this on the newly built server.
So we can click **Next**

![Image](/Images/AzureArc/HyperV4.JPG)


Now we arrive at the screen to install the role. Click **Hyper-V**

![Image](/Images/AzureArc/HyperV5.JPG)

A pop-up will then appear showing the features that are required to use Hyper-V.
Click **Add Features**

![Image](/Images/AzureArc/HyperV6.JPG)

Hyper-V is now selected and we can continue. Click **Next**

![Image](/Images/AzureArc/HyperV7.JPG)

On the Features page, click **Next** again
We now move on to the configuration of Hyper-V

You will now land on the start screen of the Hyper-V configuration. Click **Next**

![Image](/Images/AzureArc/HyperV8.JPG)

Next, select a network adapter that you have specifically chosen for Hyper-V and click **Next**

![Image](/Images/AzureArc/HyperV9.JPG)

After that we arrive at the Migration tab. If you have multiple Hyper-V nodes, you can choose to enable Live Migration.
For now this is just a test, so we only have 1 node. We will therefore not enable Migration here. So click **Next**

![Image](/Images/AzureArc/HyperV10.JPG)

Fill in the default locations for the VMs and click **Next**

![Image](/Images/AzureArc/HyperV11.JPG)

Everything is now configured and we can click **Install**.

![Image](/Images/AzureArc/HyperV12.JPG)


~~~~
Mocht je Hyper-V willen deployen via Powershell gebruik dan onderstaande regel:

Install-WindowsFeature -Name Hyper-V -IncludeManagementTools

~~~~

Now build 2 virtual Windows Servers. We will use these to show how you can configure Azure Arc.

## Configure your VM so you can install Azure Arc on an Azure VM.

Normally, of course, you would not install Azure ARC on an Azure VM.
That is why we need to tweak the machine a little bit to be able to play with Azure Arc.

To do this, open PowerShell and enter the following: 

~~~~
[System.Environment]::SetEnvironmentVariable("MSFT_ARC_TEST",'true', [System.EnvironmentVariableTarget]::Machine)

Set-Service WindowsAzureGuestAgent -StartupType Disabled -Verbose
Stop-Service WindowsAzureGuestAgent -Force -Verbose

~~~~

If necessary, you can also create a firewall rule to allow the connection to be made.
New-NetFirewallRule -Name BlockAzureIMDS -DisplayName "Block access to Azure IMDS" -Enabled True -Profile Any -Direction Outbound -Action Block -RemoteAddress 169.254.169.254



## Configure Azure Arc

You can onboard Linux and Windows servers using the Azure Arc Center in the Azure Portal.

Once you are in the Azure Portal, search for **Azure Arc**.

![Image](/Images/AzureArc/Arc1.JPG)

Then click **Infrastructure** and then **ADD** under Servers.

![Image](/Images/AzureArc/Arc2.JPG)

You can now choose between Single, Multiple or Update management.


![Image](/Images/AzureArc/Arc3.JPG)

~~~~
Single = 1 machine per keer installeren
Multiple = meerdere machines installeren via een Service Principal, hiermee kun je het script geautomatiseerd laten lopen zonder tussenkomst op meerdere machines.
Update Management = Machines die in Azure Update Management staan worden door middel van een Service principal toegevoegd aan Azure ARC
~~~~

Since we are going to use this for a test, I would choose Single if you want to quickly add the 2 machines, or Multiple if you already have a service principal.

So we now choose **Generate Script** under Add a single server

![Image](/Images/AzureArc/ArcServer1.JPG)

Review the Prerequisites and then click **Next**

![Image](/Images/AzureArc/ArcServer2.JPG)

Now we fill in the Resource details

~~~~
Subscription = de bestaande Subscription
Resource group = een resource group die je gemaakt hebt voor Azure ARC

Region = Regio het dichtst in de buurt is van je locatie. 
Operating System = Linux of Windows

Connectivity method = Public or Private endpoint afhankelijk wat je geselecteerd hebt.
~~~~
Then click **Next**

![Image](/Images/AzureArc/ArcServer3.JPG)

After that you can specify Tags for the servers if needed. For example for Backup or responsible groups, location, etc.
Then click **Next**

![Image](/Images/AzureArc/ArcServer4.JPG)

You will then see the script. You can copy it or download the ps1. Download the PS1 and copy it to the Hyper-V server.

![Image](/Images/AzureArc/ArcServer5.JPG)

Log in to the Hyper-V server and open PowerShell in Admin mode.

I placed the ps1 in the folder C:\Scripts. You can now run it. 
At the end, a pop-up will appear asking for your Azure credentials. Make sure you have Enhanced mode for IE turned off.

![Image](/Images/AzureArc/ArcServer6.JPG)

Enter your email address for Azure here and click **Next**

![image](/Images/AzureArc/ArcServer7.JPG)

Enter your password and click **Next**

![image](/Images/AzureArc/ArcServer8.JPG)

PowerShell will then indicate that the script has completed successfully

![Image](/Images/AzureArc/ArcServer9.JPG)

If you then go to Azure Arc in Azure and then to Servers, you will see this machine listed there

![Image](/Images/AzureArc/Arc4.JPG)


This completes the installation of Azure Arc, and you can use the machines you have to apply Azure Policies, Update management, Logic Apps or other things to them.
Later I will write another blog about how to use Update Manager on your Azure Arc machines.

