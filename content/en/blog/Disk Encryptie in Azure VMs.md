---
Author: "Andy Clifton"
title: "Disk Encryption in Azure VMs"
description: "Disk Encryption in Azure VMs."
date: 2023-02-16
tags: ["AzureB2B"]
thumbnail: "/img/DiskEncryption.png"
---

# Disk Encryption in Azure VMs

### ==> Disclaimer: work in progress <==

## Introduction
In the past, the process of encrypting a virtual machine in Microsoft Azure required an AD App account to do the work.
Nowadays we have a simpler way to encrypt our VMs in Microsoft Azure, and this new method is integrated into the Azure Portal.
In this article we will cover the basics of enabling encryption using the new method, including creating an Azure Key Vault.
We will also walk through the steps that can be performed in the Azure Portal and the PowerShell functions that support the encryption process.

![Image](/Images/Encryptie/Diagram.png)

## Setting up the Key Vault
The first step is creating a Key Vault.
Log in to the Azure portal and click **Create a resource**.

![Image](/Images/Encryptie/CreateResource.png)

Enter **Key vault** and click **Create**.

![Image](/Images/Encryptie/CreateKeyvault.png)

A new wizard will appear. Here we can configure all the basic settings we need to start using the new Key Vault with disk encryption.

Fill in: 
~~~
Key vault name = Name 
Region = Region
Pricing Tier = Standard or Premium
~~~
![Image](/Images/Encryptie/KeyvaultSettings.png)

Click **Next** and we go to Access Policy. 
Make sure the checkbox **Azure Disk encryption for volume encryption** is ticked. 

![Image](/Images/Encryptie/ResourceAccess.png)

Click **Review + create** to start the deployment of the Key Vault.


> **_Note:_** If you are using a hub and spoke topology, using a Key Vault in your shared infrastructure in the hub is the right way to go.
This way all your encrypted VMs use the centralized keys to store and encrypt their disks.

After creating the Key Vault, our next step is creating our first set of keys. Open the newly created Key Vault and click **Keys** under Objects.

Click **Generate/Import**.

![Image](/Images/Encryptie/Keys.png)

On the Create a Key page select the following:
~~~
Options = Generate
Name = Enter a logical name
Key type = RSA
RSA key size = minimum 2048
Enabled = Yes
~~~

![Image](/Images/Encryptie/CreateKeys.png)

## Encrypting a VM via the Azure Portal.


Nowadays the Azure Portal offers a simple way to encrypt your VM with Azure Key Vault integration.
In the VM whose disk we want to encrypt, click Disks (number 1) and a list of all disks attached to the given VM is displayed.
At number 3 we can see the current encryption status for a particular disk.
To start the encryption process we click Encryption (number 2).

![Image](/Images/Encryptie/EncryptDisk.png)

In the new tab we have three options: None (no encryption), OS disk (only the disk with the operating system), or OS and data disk (all disks on this VM).

On the Encryption settings page select the following:

~~~~
Disk to Encrypt = OS Disk
Key Vault =  the Key Vault you created
Key =  Select the Key.
Version = Version 
~~~~
![Image](/Images/Encryptie/Encryptionsettings.png)

Select the Key we just created and then click Save.

As soon as you click Save, a dialog box is displayed asking for confirmation. Click Yes.
This process causes the VM to restart, so make sure you do this during planned maintenance or when the users are aware of it.

As part of the restart, the AzureDiskEncryption extension will be added to the VM.

![Image](/Images/Encryptie/AzureDiskEncryptionExtensie.png)


## Managing disks and encryption scenarios.

Life is good, the operating system disk is being encrypted.
One thing is certain though, there will be changes!
Let's run through a number of possible scenarios when managing disk encryption, to see how you should act when these situations occur.

Let's start with a requirement to add a new data disk to the same VM we have been working on throughout this article.
The most important thing (before you even think about encryption) is to understand that the disk must be present and connected within the operating system for the encryption to work.
If you add the disk via the Azure Portal, the disk shows up on the VM as uninitialized and it will not be encrypted.
Make sure you initialize, format and assign a drive letter to the disk.

After you have presented the new disk to the operating system,
go back to the Encryption button, change the setting to OS and data disks and click Save.

At this point a restart is not required, and after completion the new disks will be encrypted.

![Image](/Images/Encryptie/Encryptdatadisk.png)


We can see the encryption in action in the image below.
We just added a new volume (drive letter D:). After running the procedure above, the new disk has been encrypted as part of the process.
![Image](/Images/Encryptie/DataDisk.png)

Whenever you add new disks to the VM, we have to repeat the process to encrypt the new disks as well. (Keep this in mind.)


## Using Disk Encryption with PowerShell

With all that information, the PowerShell cmdlet (**Set-AzVMDiskEncryptionExtension**) would look similar to the cmdlet below.

```
Set-AzVmDiskEncryptionExtension -ResourceGroupName <ResourceGroupName> -VMName <VMName> -DiskEncryptionKeyVaultId  <Key Vault Resource ID> -DiskEncryptionKeyVaultUrl <Key Vault URL> -KeyEncryptionKeyVaultId <Key Vault Resource ID> -KeyEncryptionKeyUrl <Key URL> -VolumeType "All"
```
I have to admit that gathering all that information is certainly not an easy job, because you have to retrieve it using 2 cmdlets and put the pieces together to build the cmdlet above.
I found a function for this to automate the process further. You need to provide the Key Vault name, the Key name and the VM name.
The function collects all the required information in variables and starts the encryption of all disks on the given VM.
Here is the code for the function. To use it in a PowerShell session, enter EncryptVM -Keyvaultname,KeyName,VMname


````
Function EncryptVM($KeyVault,$KeyName,$VM){
$kv = Get-azKeyVault -VaultName $KeyVault
$key = get-Azkeyvaultkey -Name $KeyName -VaultName $KeyVault
Set-AzVmDiskEncryptionExtension -ResourceGroupName $kv.ResourceGroupName `
-DiskEncryptionKeyVaultId  $kv.ResourceID `
-DiskEncryptionKeyVaultUrl $kv.VaultURI `
-VMName $VM -KeyEncryptionKeyVaultId $kv.ResourceID `-KeyEncryptionKeyUrl $key.id -SkipVmBackup -VolumeType "All"}

````

More information about disk encryption can be found [here](https://learn.microsoft.com/en-us/azure/virtual-machines/windows/disk-encryption-faq#can-i-encrypt-both-boot-and-data-volumes-with-azure-disk-encryption)


