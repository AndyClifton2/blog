---
Author: "Andy Clifton"
title: "Installing and configuring SCOM Managed Instance"
description: "Installing and configuring SCOM Managed Instance"
date: 2024-05-07
tags: ["emoji"]
thumbnail: "/Images/SCOMMI/SCOM.png"
---

# Installing and configuring SCOM Managed Instance

### ==> Disclaimer: work in progress <==

## Introduction

SCOM Managed Instance is a cloud-based alternative to System Center Operations Manager as we currently know it on-premises at our customers.
SCOM Managed Instance gives us the opportunity to continuously monitor workloads (so not only VMs or SQL databases) and, where possible, to resolve problems on these workloads with automated solutions (think, for example, of step 1 restart the service, step 2 restart the server, step 3 reinstall the application, and as a final step send an error to the administrator by mail or only via the portal).

Management is not limited to workloads in Azure. It is also possible (as long as there is network connectivity) to add workloads that are running on-premises.

**Biggest advantages**

- The investments you made in your old SCOM environment are preserved. Management Packs can be migrated to the new environment.
- Managing the SCOM infrastructure has become a lot simpler. All cloud-connected components are managed by Microsoft, so you no longer have to  
  worry about anything in your environment except SCOM itself.
- During a migration there is no interruption when you migrate the workloads to the cloud.
- You can migrate from an existing SCOM Management Group setup.
- You can manage your SCOM workloads from anywhere. You no longer necessarily have to be on-premises to manage your workloads.


**SCOM MI Architecture**
![Image](/Images/SCOMMI/architecture.png)

SCOM MI consists of 2 parts.
- Microsoft-Managed part
- Customer-Managed part

Below is a detailed map that explains which part falls under the Microsoft subscription, which part falls under your customer's subscription, and what is managed on-premises.

![Image](/Images/SCOMMI/detailed.png)


## Deploying SCOM Managed Instance.

Log in to the server via RDP.



![Image](/Images/ShrinkOS/rdp.png)

Once you are logged in, go to Disk Management.
.

![Image](/Images/ShrinkOS/disk.png)

When Disk Management is open, check how much free space you have on the C drive. Make sure you keep enough room for growth (roughly 15 to 20 percent free is sufficient).

![Image](/Images/ShrinkOS/diskc.png)

When we look in Disk Management we see the disk number shown below. Remember or write this down, we will need it later on.

![Image](/Images/ShrinkOS/disk0.png)
Next, open PowerShell and enter the line below to shrink your disk.

~~~
Get-Partition -DiskNumber 0  (the disk number is the same number you found above.)

~~~


![Image](/Images/ShrinkOS/disknumber.png)

In the example above we see that partition number 4 is the one we need to use, because that is where the C drive lives.
In our example we shrink the disk to 90 GB. But of course you can choose whatever suits you best.
Enter the following line in PowerShell: 

~~~
Get-Partition -DiskNumber 0 -PartitionNumber 4 | Resize-Partition -Size 90GB

~~~


Sometimes you get the error below:

![Image](/Images/ShrinkOS/error.png)
~~~
The specified shrink size is too big and will cause the volume to be smaller than the minimum volume size
~~~

The solution is actually quite simple: try the PowerShell line another 2 or 3 times and it will work. This happens because a query is being run and sometimes files in a certain block are in use.

![Image](/Images/ShrinkOS/disknew.png)

Turn off the machine via Shutdown:

![Image](/Images/ShrinkOS/shutdown.png)

Once the machine is off, we also need to deallocate it.
Go to the Azure portal and search for Virtual Machines.

![Image](/Images/ShrinkOS/VirtualMachines.png)

Search for the virtual machine and click Stop.
![Image](/Images/ShrinkOS/stop.png)

Once the machine has stopped, you will also see that it shows as deallocated.
![Image](/Images/ShrinkOS/stopped.png)

Now we start PowerShell ISE from your own machine and run a script there.
I will first walk through the parts of the script step by step, and at the end I will copy the complete script once more.

First we install the Azure modules
~~~
Install-Module -name Az
~~~
![Image](/Images/ShrinkOS/Azmodule.jpg)

After that we collect the information we will need later in the script

~~~
# Variables
$DiskID = "# eg. "/subscriptions/203bdbf0-69bd-1a12-a894-a826cf0a34c8/resourcegroups/rg-server1-prod-1/providers/Microsoft.Compute/disks/Server1-Server1"
$VMName = "eg. Vmname"
$DiskSizeGB = 90 (eq. DiskSize)
$AzSubscription = "eq. Subscription name"
~~~

You can retrieve the DiskId by opening the disk of the VM and going to Properties.
![Image](/Images/ShrinkOS/disktype.png)

After that we connect to Azure.
~~~
# Login to Azure with your Admin Account:
Connect-AzAccount
~~~
![Image](/Images/ShrinkOS/login.png)

You will then get a message that the authentication was successful. Close your browser and go back to PowerShell ISE.
![Image](/Images/ShrinkOS/loginsuccess.png)

In ISE you will then see that a connection has been made.



![Image](/Images/ShrinkOS/loginise.png)

Now we select the subscription the VM is running under. (This step is only necessary if you have multiple subscriptions.)


~~~
#Provide the subscription Id of the subscription where VM is created
Select-AzSubscription -Subscription $AzSubscription
~~~

![Image](/Images/ShrinkOS/selectsubscription.png)

After that we retrieve all the information about the VM in question.

~~~
# VM to resize disk of
$VM = Get-AzVm | ? Name -eq $VMName
~~~

![Image](/Images/ShrinkOS/outputvm.png)

Now we extract some information from the VM and its disk.

~~~
#Provide the name of your resource group where VM is created
$resourceGroupName = $VM.ResourceGroupName

# Get Disk from ID
$Disk = Get-AzDisk | ? Id -eq $DiskID

# Get VM/Disk generation from Disk
$HyperVGen = $Disk.HyperVGeneration

# Get Disk Name from Disk
$DiskName = $Disk.Name

~~~

The output looks like this:

For $disk:
![Image](/Images/ShrinkOS/disk1.png)

For $HyperVGen:
![Image](/Images/ShrinkOS/gen.png)

For $DiskName:

![Image](/Images/ShrinkOS/diskname.png)

Next we create a snapshot and a new, smaller disk based on the information we provided.
For this we grant SAS Read access.

~~~
# Get SAS URI for the Managed disk
$SAS = Grant-AzDiskAccess -ResourceGroupName $resourceGroupName -DiskName $DiskName -Access 'Read' -DurationInSecond 600000;

#Provide storage account name where you want to copy the snapshot - the script will create a new one temporarily
$storageAccountName = "shrink" + [system.guid]::NewGuid().tostring().replace('-','').substring(1,18)

#Name of the storage container where the downloaded snapshot will be stored
$storageContainerName = $storageAccountName

#Provide the name of the VHD file to which snapshot will be copied.
$destinationVHDFileName = "$($VM.StorageProfile.OsDisk.Name).vhd"

#Create the context for the storage account which will be used to copy snapshot to the storage account 
$StorageAccount = New-AzStorageAccount -ResourceGroupName $resourceGroupName -Name $storageAccountName -SkuName Standard_LRS -Location $VM.Location
$destinationContext = $StorageAccount.Context
$container = New-AzStorageContainer -Name $storageContainerName -Permission Off -Context $destinationContext

#Copy the snapshot to the storage account and wait for it to complete
Start-AzStorageBlobCopy -AbsoluteUri $SAS.AccessSAS -DestContainer $storageContainerName -DestBlob $destinationVHDFileName -DestContext $destinationContext
while(($state = Get-AzStorageBlobCopyState -Context $destinationContext -Blob $destinationVHDFileName -Container $storageContainerName).Status -ne "Success") { $state; Start-Sleep -Seconds 20 }
$state

# Revoke SAS token
Revoke-AzDiskAccess -ResourceGroupName $resourceGroupName -DiskName $DiskName

# Emtpy disk to get footer from
$emptydiskforfootername = "$($VM.StorageProfile.OsDisk.Name)-empty.vhd"

$diskConfig = New-AzDiskConfig `
    -Location $VM.Location `
    -Zone $VM.Zones `
    -CreateOption Empty `
    -DiskSizeGB $DiskSizeGB `
    -HyperVGeneration $HyperVGen

$dataDisk = New-AzDisk `
    -ResourceGroupName $resourceGroupName `
    -DiskName $emptydiskforfootername `
    -Disk $diskConfig

$VM = Add-AzVMDataDisk `
    -VM $VM `
    -Name $emptydiskforfootername `
    -CreateOption Attach `
    -ManagedDiskId $dataDisk.Id `
    -Lun 63

Update-AzVM -ResourceGroupName $resourceGroupName -VM $VM

$VM | Stop-AzVM -Force


# Get SAS token for the empty disk
$SAS = Grant-AzDiskAccess -ResourceGroupName $resourceGroupName -DiskName $emptydiskforfootername -Access 'Read' -DurationInSecond 600000;

# Copy the empty disk to blob storage
Start-AzStorageBlobCopy -AbsoluteUri $SAS.AccessSAS -DestContainer $storageContainerName -DestBlob $emptydiskforfootername -DestContext $destinationContext
while(($state = Get-AzStorageBlobCopyState -Context $destinationContext -Blob $emptydiskforfootername -Container $storageContainerName).Status -ne "Success") { $state; Start-Sleep -Seconds 20 }
$state

# Revoke SAS token
Revoke-AzDiskAccess -ResourceGroupName $resourceGroupName -DiskName $emptydiskforfootername

# Remove temp empty disk
Remove-AzVMDataDisk -VM $VM -DataDiskNames $emptydiskforfootername
Update-AzVM -ResourceGroupName $resourceGroupName -VM $VM

# Delete temp disk
Remove-AzDisk -ResourceGroupName $resourceGroupName -DiskName $emptydiskforfootername -Force;

# Get the blobs
$emptyDiskblob = Get-AzStorageBlob -Context $destinationContext -Container $storageContainerName -Blob $emptydiskforfootername
$osdisk = Get-AzStorageBlob -Context $destinationContext -Container $storageContainerName -Blob $destinationVHDFileName

$footer = New-Object -TypeName byte[] -ArgumentList 512
write-output "Get footer of empty disk"

$downloaded = $emptyDiskblob.ICloudBlob.DownloadRangeToByteArray($footer, 0, $emptyDiskblob.Length - 512, 512)

$osDisk.ICloudBlob.Resize($emptyDiskblob.Length)
$footerStream = New-Object -TypeName System.IO.MemoryStream -ArgumentList (,$footer)
write-output "Write footer of empty disk to OSDisk"
$osDisk.ICloudBlob.WritePages($footerStream, $emptyDiskblob.Length - 512)

Write-Output -InputObject "Removing empty disk blobs"
$emptyDiskblob | Remove-AzStorageBlob -Force


#Provide the name of the Managed Disk
$NewDiskName = "$DiskName" + "-new"

#Create the new disk with the same SKU as the current one
$accountType = $Disk.Sku.Name

# Get the new disk URI
$vhdUri = $osdisk.ICloudBlob.Uri.AbsoluteUri

# Specify the disk options
$diskConfig = New-AzDiskConfig -AccountType $accountType -Location $VM.location -Zone $VM.Zones -DiskSizeGB $DiskSizeGB -SourceUri $vhdUri -CreateOption Import -StorageAccountId $StorageAccount.Id -HyperVGeneration $HyperVGen

# Handle Trusted Launch VMs/Disks
If($Disk.SecurityProfile.SecurityType -eq "TrustedLaunch"){
    $diskconfig = Set-AzDiskSecurityProfile -Disk $diskconfig -SecurityType "TrustedLaunch"
}
~~~

To keep things from getting too long, I will not include an export of everything. But when you run it, you will see the message below running until the copy operation has completed.

![Image](/Images/ShrinkOS/Copy.png)

When the copy operation has fully completed, you get the following message:

![Image](/Images/ShrinkOS/outcome.png)

Next we create a new disk based on the information that was set in the previous part (for example the size, but also the Security Type).

~~~
$NewManagedDisk = New-AzDisk -DiskName $NewDiskName -Disk $diskConfig -ResourceGroupName $resourceGroupName
~~~

This disk is created fairly quickly and you get an ID containing the new name.

![Image](/Images/ShrinkOS/newdisk.png)

Now we stop the existing VM. It was started again during the script after an Update.

~~~
$VM | Stop-AzVM -Force
~~~

![Image](/Images/ShrinkOS/stopvm.png)

After that we adjust the configuration, attach the disk to the VM and start the machine again.

~~~
# Set the VM configuration to point to the new disk  
Set-AzVMOSDisk -VM $VM -ManagedDiskId $NewManagedDisk.Id -Name $NewManagedDisk.Name

# Update the VM with the new OS disk
Update-AzVM -ResourceGroupName $resourceGroupName -VM $VM

$VM | Start-AzVM

~~~

![Image](/Images/ShrinkOS/klaar.png)

We have now built a pause of 3 minutes into the script to make sure the disk is properly synced and the machine has fully started.

The last step in the script is removing the old disk blob storage and the temporary storage account.

~~~
# Delete old Managed Disk
Remove-AzDisk -ResourceGroupName $resourceGroupName -DiskName $DiskName -Force;

# Delete old blob storage
$osdisk | Remove-AzStorageBlob -Force

# Delete temp storage account
$StorageAccount | Remove-AzStorageAccount -Force
~~~

The machine is now running with a new disk.

![Image](/Images/ShrinkOS/newestdisk.png)

As promised, here is the complete script once more. After that we continue with the configuration inside the VM.
~~~
# Variables
$DiskID = "# eg. "/subscriptions/203bdbf0-69bd-1a12-a894-a826cf0a34c8/resourcegroups/rg-server1-prod-1/providers/Microsoft.Compute/disks/Server1-Server1"
$VMName = "eg. Vmname"
$DiskSizeGB = 90 (eq. DiskSize)
$AzSubscription = "eq. Subscription name"
# Script
# Login to Azure with your Admin Account:
Connect-AzAccount

#Provide the subscription Id of the subscription where VM is created
Select-AzSubscription -Subscription $AzSubscription

# VM to resize disk of
$VM = Get-AzVm | ? Name -eq $VMName

#Provide the name of your resource group where VM is created
$resourceGroupName = $VM.ResourceGroupName

# Get Disk from DiskID
$Disk = Get-AzDisk | ? Id -eq $DiskID

# Get VM/Disk generation from Disk
$HyperVGen = $Disk.HyperVGeneration

# Get Disk Name from Disk
$DiskName = $Disk.Name

# Get SAS URI for the Managed disk
$SAS = Grant-AzDiskAccess -ResourceGroupName $resourceGroupName -DiskName $DiskName -Access 'Read' -DurationInSecond 600000;

#Provide storage account name where you want to copy the snapshot - the script will create a new one temporarily
$storageAccountName = "shrink" + [system.guid]::NewGuid().tostring().replace('-','').substring(1,18)

#Name of the storage container where the downloaded snapshot will be stored
$storageContainerName = $storageAccountName

#Provide the name of the VHD file to which snapshot will be copied.
$destinationVHDFileName = "$($VM.StorageProfile.OsDisk.Name).vhd"

#Create the context for the storage account which will be used to copy snapshot to the storage account 
$StorageAccount = New-AzStorageAccount -ResourceGroupName $resourceGroupName -Name $storageAccountName -SkuName Standard_LRS -Location $VM.Location
$destinationContext = $StorageAccount.Context
$container = New-AzStorageContainer -Name $storageContainerName -Permission Off -Context $destinationContext

#Copy the snapshot to the storage account and wait for it to complete
Start-AzStorageBlobCopy -AbsoluteUri $SAS.AccessSAS -DestContainer $storageContainerName -DestBlob $destinationVHDFileName -DestContext $destinationContext
while(($state = Get-AzStorageBlobCopyState -Context $destinationContext -Blob $destinationVHDFileName -Container $storageContainerName).Status -ne "Success") { $state; Start-Sleep -Seconds 20 }
$state

# Revoke SAS token
Revoke-AzDiskAccess -ResourceGroupName $resourceGroupName -DiskName $DiskName

# Emtpy disk to get footer from
$emptydiskforfootername = "$($VM.StorageProfile.OsDisk.Name)-empty.vhd"

$diskConfig = New-AzDiskConfig `
    -Location $VM.Location `
    -Zone $VM.Zones `
    -CreateOption Empty `
    -DiskSizeGB $DiskSizeGB `
    -HyperVGeneration $HyperVGen

$dataDisk = New-AzDisk `
    -ResourceGroupName $resourceGroupName `
    -DiskName $emptydiskforfootername `
    -Disk $diskConfig

$VM = Add-AzVMDataDisk `
    -VM $VM `
    -Name $emptydiskforfootername `
    -CreateOption Attach `
    -ManagedDiskId $dataDisk.Id `
    -Lun 63

Update-AzVM -ResourceGroupName $resourceGroupName -VM $VM

$VM | Stop-AzVM -Force


# Get SAS token for the empty disk
$SAS = Grant-AzDiskAccess -ResourceGroupName $resourceGroupName -DiskName $emptydiskforfootername -Access 'Read' -DurationInSecond 600000;

# Copy the empty disk to blob storage
Start-AzStorageBlobCopy -AbsoluteUri $SAS.AccessSAS -DestContainer $storageContainerName -DestBlob $emptydiskforfootername -DestContext $destinationContext
while(($state = Get-AzStorageBlobCopyState -Context $destinationContext -Blob $emptydiskforfootername -Container $storageContainerName).Status -ne "Success") { $state; Start-Sleep -Seconds 20 }
$state

# Revoke SAS token
Revoke-AzDiskAccess -ResourceGroupName $resourceGroupName -DiskName $emptydiskforfootername

# Remove temp empty disk
Remove-AzVMDataDisk -VM $VM -DataDiskNames $emptydiskforfootername
Update-AzVM -ResourceGroupName $resourceGroupName -VM $VM

# Delete temp disk
Remove-AzDisk -ResourceGroupName $resourceGroupName -DiskName $emptydiskforfootername -Force;

# Get the blobs
$emptyDiskblob = Get-AzStorageBlob -Context $destinationContext -Container $storageContainerName -Blob $emptydiskforfootername
$osdisk = Get-AzStorageBlob -Context $destinationContext -Container $storageContainerName -Blob $destinationVHDFileName

$footer = New-Object -TypeName byte[] -ArgumentList 512
write-output "Get footer of empty disk"

$downloaded = $emptyDiskblob.ICloudBlob.DownloadRangeToByteArray($footer, 0, $emptyDiskblob.Length - 512, 512)

$osDisk.ICloudBlob.Resize($emptyDiskblob.Length)
$footerStream = New-Object -TypeName System.IO.MemoryStream -ArgumentList (,$footer)
write-output "Write footer of empty disk to OSDisk"
$osDisk.ICloudBlob.WritePages($footerStream, $emptyDiskblob.Length - 512)

Write-Output -InputObject "Removing empty disk blobs"
$emptyDiskblob | Remove-AzStorageBlob -Force


#Provide the name of the Managed Disk
$NewDiskName = "$DiskName" + "-new"

#Create the new disk with the same SKU as the current one
$accountType = $Disk.Sku.Name

# Get the new disk URI
$vhdUri = $osdisk.ICloudBlob.Uri.AbsoluteUri

# Specify the disk options
$diskConfig = New-AzDiskConfig -AccountType $accountType -Location $VM.location -Zone $VM.Zones -DiskSizeGB $DiskSizeGB -SourceUri $vhdUri -CreateOption Import -StorageAccountId $StorageAccount.Id -HyperVGeneration $HyperVGen

# Handle Trusted Launch VMs/Disks
If($Disk.SecurityProfile.SecurityType -eq "TrustedLaunch"){
    $diskconfig = Set-AzDiskSecurityProfile -Disk $diskconfig -SecurityType "TrustedLaunch"
}

#Create Managed disk
$NewManagedDisk = New-AzDisk -DiskName $NewDiskName -Disk $diskConfig -ResourceGroupName $resourceGroupName

$VM | Stop-AzVM -Force

# Set the VM configuration to point to the new disk  
Set-AzVMOSDisk -VM $VM -ManagedDiskId $NewManagedDisk.Id -Name $NewManagedDisk.Name

# Update the VM with the new OS disk
Update-AzVM -ResourceGroupName $resourceGroupName -VM $VM

$VM | Start-AzVM

start-sleep 180
# Please check the VM is running before proceeding with the below tidy-up steps

# Delete old Managed Disk
Remove-AzDisk -ResourceGroupName $resourceGroupName -DiskName $DiskName -Force;

# Delete old blob storage
$osdisk | Remove-AzStorageBlob -Force

# Delete temp storage account
$StorageAccount | Remove-AzStorageAccount -Force

~~~
If you now go to the VM in Azure and then to Disks, you will see that the disk size has also been changed to the 90 GB we specified earlier.
![Image](/Images/ShrinkOS/diskazure.png)


Now log in to the server and open Disk Management. You will see that there is still a very small amount of unallocated storage left.
Click on the C drive and then click Extend Volume.
![Image](/Images/ShrinkOS/ExtendVolume.png)

You will then get the Extend Volume Wizard. Select the available space and click Next.

![Image](/Images/ShrinkOS/Extend1.png)

Then click Finish

![Image](/Images/ShrinkOS/Extend2.png)


You have now successfully made your OS disk smaller. If you can do this on multiple servers, you will have achieved a considerable cost saving for your employer.
