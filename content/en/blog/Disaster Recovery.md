---
Author: "Andy Clifton"
title: "Disaster Recovery"
date: 2023-01-16
description: "Disaster Recovery."
tags: ["Disater,Recovery"]
thumbnail: "/img/Disasterrecovery.jpeg"
---


# Disaster Recovery

### ==> Disclaimer: work in progress. <==

## Introduction

If you have a virtual machine running in Azure and you want to make sure your machines are protected against downtime as much as possible, there are several things you want to look at in terms of availability.
You need to make sure you have the required uptime in the region where the VMs are running.
For this you can think of different storage options, such as Premium Storage, which offers an SLA of 99.9% for a single disk. You can also work with Availability Sets and Availability Zones, which let you spread an important server across multiple datacenters or multiple zones within 1 region.
However, we also want to protect the server against the failure of an entire region. For this, Azure Site Recovery (ASR) is a suitable solution. With ASR we replicate the machine asynchronously to another region or regions.

Let's take a look at the setup of ASR.



## Setting up Disaster Recovery for Azure VMs

Setting up Azure Site Recovery for a VM can be fairly straightforward. In our case we have a virtual machine running in the West Europe region. Before we can configure Disaster Recovery, you need to make sure your environment has been configured and designed in the right way.
If you want to know which requirements apply to your architecture, have a look at the links below.

-[Support Matrix for Azure VM disaster recovery between Azure Regions](https://docs.microsoft.com/en-us/azure/site-recovery/azure-to-azure-support-matrix?WT.mc_id=itopstalk-blog-thmaure)

-[Azure to Azure disaster recovery architecture](https://docs.microsoft.com/en-us/azure/site-recovery/azure-to-azure-architecture?WT.mc_id=itopstalk-blog-thmaure)

There are 3 ways to configure disaster recovery for Azure VMs

~~~
- Azure PowerShell/Azure CLI
- Azure Recovery Services vault
- start replication directly from the VM

~~~

Starting replication directly from the VM is the most commonly used method, so that is the one I will discuss further here.

**Go to** the VM via the Azure Portal
**Open Disaster Recovery** under Operations
![Image](/Images/DisasterRecovery/disaster.png)

This part is easy: we only need to fill in the Target Region. In this [list](https://docs.microsoft.com/en-us/azure/availability-zones/cross-region-replication-azure) you can see what the paired cross regions are. It is easier and more common sense to choose one of these as the Target Region.

After that we need to configure a number of Advanced settings. ASR already provides a few suggestions, but you can adjust these to whatever you think is correct.

![Image](/Images/DisasterRecovery/advanced.png)

You can also adjust the Storage and Replication Settings.
Under Storage, for example, you can choose not to use a Premium SSD but just an HDD disk or a Standard SSD disk, in order to save costs.
Under Replication settings you can adjust the Replication Policy and thereby control how the sync runs.

![Image](/Images/DisasterRecovery/replication.png)

Once you have adjusted everything to your liking, you can click Review + Start Replication. It will take a number of hours before this has been created and the replication has run. (This depends on the size of the VM.)

## Failover of an Azure VM to another region.

After we have set up Disaster Recovery and the replication has run, we can now look at some graphical views that clearly show what is happening and how the resources are connected to each other.
![Image](/Images/DisasterRecovery/replication1.jpg)

You can perform a failover, which carries out a production failover of the VM. If the region where the source is located is still available, you can let ASR stop the machine and sync the latest changes, so that we have no data loss.
This is of course only possible if the source region still exists. Otherwise ASR takes the latest existing restore point.
![Image](/Images/DisasterRecovery/restorepoint.jpg)

Being prepared for disaster recovery is excellent. However, we want to be sure that it works when you need it. You do not want to wait until a real disaster strikes to find out whether everything is set up correctly. This is what Test Failover is for. Test Failover gives us the option to fail over the virtual machine to an isolated virtual network in the target region, so we can test the virtual machine and application without any impact on the production deployment.

If you have multiple VMs that you want to fail over in a specific order, and perhaps even want to run some additional scripts to perform a complete failover out of a region, ASR also offers us [Recovery Plans](https://docs.microsoft.com/en-us/azure/site-recovery/recovery-plan-overview?WT.mc_id=itopstalk-blog-thmaure). Recovery Plans are better suited for more complex scenarios than just a single virtual machine.


## Azure Site Recovery via PowerShell

If you have a lot of machines that need to be added, it could be easier to do this via PowerShell.
Below is a link to Microsoft Learn that explains how to enable ASR and start replications via PowerShell.

~~~
https://learn.microsoft.com/en-us/azure/site-recovery/azure-to-azure-powershell
~~~
