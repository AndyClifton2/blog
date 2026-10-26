---
Author: "Andy Clifton"
title: "Run PowerShell Scripts with Hybrid Worker Groups"
date: 2023-08-12
tags: ["emoji"]
thumbnail: "/img/HRW.png"
---

# Run PowerShell Scripts with Hybrid Worker Groups

### ==> Disclaimer: work in progress. <==

## Introduction

If you want to automate your on-premises environment, Azure Arc-enabled servers are a great offering for onboarding Azure management services such as Azure Monitor and Sentinel, but also Defender and IAM.
One of the other advantages we get is the use of Hybrid Workers. These let you use Runbooks, Logic Apps or Power Automate to manage or deploy things on your on-premises VMs.

In this blog I will show you how to create an Automation account, how to connect a server via Azure Arc, how to create a Hybrid Worker group and, after that, how to create and deploy a PowerShell-based runbook.

![Image](/Images/RunPowershellHybrid/Praatplaat.JPG)

## Creating an Azure Automation Account.

In my opinion it is best to put the Automation Account in its own resource group. That way you keep a clear overview of what belongs to this Automation account. From a networking point of view, keep in mind that this account and everything underneath it needs to be able to reach the environments.

In this document I will not describe how to create a resource group, because I assume that is already familiar.

In the search bar, search for Automation account and click on it.

![Image](/Images/RunPowershellHybrid/zoekbalk.jpg)

Then click **Create**

![Image](/Images/RunPowershellHybrid/create.jpg)

Fill in: 
~~~

Resource Group = The resource group you created
Automation Account name= Choose a logical name for this account.
Region = Same as the resource group

~~~

![Image](/Images/RunPowershellHybrid/CreateAA1.jpg)

Click **Next**

You now land on the Advanced tab, where you can specify which identity this account should use. In this case we choose a System Managed identity.

![Image](/Images/RunPowershellHybrid/CreateAA2.jpg)

We do not need to change anything else for now. If you only want to allow private access (meaning access to this account is blocked from anywhere else), you need to select Private Access under networking.

For now, click **Review + Create**
![Image](/Images/RunPowershellHybrid/CreateAA3.jpg)



If you want more information about an Automation Account and how to create one, have a look here:

-[Create a standalone Azure Automation account](https://learn.microsoft.com/en-us/azure/automation/automation-create-standalone-account?WT.mc_id=modinfra-0000-thmaure&tabs=azureportal)


## Connect a Server to Azure via Azure Arc

If you want to connect a server that is located on-premises or at another cloud provider to Azure via Azure Arc, go to Azure Arc Center in the Azure Portal and select Azure Arc servers. Here you can click the **Add** button and walk through the onboarding wizard. In a few weeks I will publish another blog on how to configure Azure Arc for your on-premises environment.

![Image](/Images/RunPowershellHybrid/ArcOnboarding.jpg)


## Creating a Hybrid Worker Group

Now you can create and maintain a Hybrid Worker Group for running tasks that you also want to run on-premises. This provides resilience for running tasks across multiple Hybrid Workers. With extension-based Hybrid Workers (preview), both Azure machines and non-Azure machines (via Arc-enabled servers) can be managed through ARM templates and policies.

If you want to create a hybrid worker (which can be Windows and Linux servers), open the Automation account and go to Hybrid Worker groups.
![Image](/Images/RunPowershellHybrid/HWG1.jpg)

Then click **Create Hybrid Worker Group**

![Image](/Images/RunPowershellHybrid/HWG3.jpg)

Fill in: 
~~~

Name = Logical name for the HWG
Use Hybrid Worker Credentials = Custom
Choose Hybrid Worker Credentials = Add Credentials

~~~

![Image](/Images/RunPowershellHybrid/HWG2.jpg)

Use a service account with the right permissions that is allowed to run the script on the machines.
Now go to the Hybrid Workers tab and click **Add Machines**

![Image](/Images/RunPowershellHybrid/HWG4.jpg)

You will now see your machines.

Select the machine on which you want to configure the Hybrid Worker and click **Add**

![Image](/Images/RunPowershellHybrid/HWG5.jpg)


After that you can click **Review + Create**

![Image](/Images/RunPowershellHybrid/HWG6.jpg)

The Hybrid extension will now be installed automatically on this machine.

![Image](/Images/RunPowershellHybrid/HWG7.jpg)

If you want, you can always add more servers here, but we will stick to one for now.

-[Deploy an extension-based Hybrid Runbook Worker in Azure Automation](https://docs.microsoft.com/en-us/azure/automation/extension-based-hybrid-runbook-worker-install?tabs=windows)


## Creating a new Azure Automation Runbook.
You can create a new Azure Automation runbook that hosts the script you are going to run. 
Depending on the runbook runtime version you choose, you need to make sure that version is also present on the Hybrid Worker server. (Think of Python and PowerShell 7.0)

In your Automation account, go to Runbooks and click **Create a Runbook**

![Image](/Images/RunPowershellHybrid/Runbook1.jpg)

Next you need to fill in some default details.

Fill in: 
~~~

Name = Logical name for the Runbook
Runbook Type = (choose from PowerShell, Python, PowerShell Workflow, Graphical PowerShell, Graphical PowerShell Workflow) We choose PowerShell here.
Runtime Version = (choose from 7.2, 7.1, 5.1) We choose 5.1 here because it is already present on servers by default.

~~~
![Image](/Images/RunPowershellHybrid/Runbook2jpg.JPG)

In the screen that now appears you can enter your PowerShell script. For now we choose to deploy a txt file on the desktop.

Fill in: 
~~~

"from azure" | out-file "C:\Users\*username*\Desktop\hybrid.txt"

~~~

After that, first click **Save** and then **Publish**

 ![Image](/Images/RunPowershellHybrid/Runbook3.JPG)


## Run and Schedule a PowerShell Script as an Azure Automation Runbook via a Hybrid Worker group on an Azure Arc machine.

You can now run and schedule your PowerShell scripts as an Azure Automation runbook. This works through the hybrid worker groups that are connected via Azure Arc. 
When you run the runbook, you can now select on which hybrid worker group this script should be executed.


 ![Image](/Images/RunPowershellHybrid/Test1.JPG)

 Via Output you can now see what happens.

 ![Image](/Images/RunPowershellHybrid/Test2.JPG)

 You can now also schedule this runbook. This is very easy: click **Link to Schedule** and select or create the schedule with its parameters.
 
 
  ![Image](/Images/RunPowershellHybrid/Test3.JPG)

