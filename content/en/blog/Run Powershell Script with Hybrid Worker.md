---
Author: "Andy Clifton"
title: "Run Powershell Script met Hybrid Worker Groups"
date: 2023-08-12
tags: ["emoji"]
thumbnail: "/img/HRW.png"
---

# Run Powershell Script met Hybrid Worker Groups

### ==> Disclaimer work in progress. <==

## Voorwoord

Als je de on-premises omgeving wilt automatiseren, is Azure Arc Server een geweldige aanbieding voor het onboarden van Azure-beheerservices zoals Azure Monitor, Sentinel, maar ook Defender en IAM.
Een van de andere voordelen die we nog hebben is het gebruik van Hybrid Workers. Hierdoor kun je Runbooks, Logic Apps of Power Automate gebruiken om het beheer of deploy van dingen op je on-premise vm mogelijk te maken.

In deze blog laat ik je zien hoe je een Automation account maakt, hoe je Connect met een server via Azure ARC, het creëren van een Hybrid Workers group en daarna het creëren en deployen van een runbook op basis van Powershell.

![Image](/Images/RunPowershellHybrid/Praatplaat.JPG)

## Maken van een Azure Automation Account.

Naar mijn idee kun je het Automation Account het beste in een eigen RG stoppen. Hierdoor houd je overzichtelijk wat hoort bij dit Automation account. Hou er netwerk technisch rekening mee dat dit account en alles wat er onder staat bij de omgevingen kan komen.

In dit document beschrijf ik niet hoe je een Resource group moet maken omdat ik er vanuit ga dat dit al bekend is.

Zoek in de zoekbalk naar Automation account en klik hierop.

![Image](/Images/RunPowershellHybrid/zoekbalk.jpg)

Klik daarna op **Create**

![Image](/Images/RunPowershellHybrid/create.jpg)

Vul in: 
~~~

Resource Group = Aangemaakte Resource group
Automation Account name= Maak een logische naam aan voor dit account.
Region = Zelfde als de RG

~~~

![Image](/Images/RunPowershellHybrid/CreateAA1.jpg)

Klik **Next**

Nu kom je in het Advanced tabblad, hier kun je aangeven van welke identity dit account gebruik moet maken. In dit geval kiezen wij voor een System Managed account.

![Image](/Images/RunPowershellHybrid/CreateAA2.jpg)

De rest hoeven we voor nu niet aan te passen. Mocht je alleen private access toe willen staan (dus de toegang tot dit account uitsluiten van toegang ergens anders vandaan) dan moet je bij networking Private Access aanklikken.

Voor nu klik op **Review + Create**
![Image](/Images/RunPowershellHybrid/CreateAA3.jpg)



Mocht je meer info willen over een Automation Account en hoe te creëren kijk dan hier:

-[Create a standalone Azure Automation account](https://learn.microsoft.com/en-us/azure/automation/automation-create-standalone-account?WT.mc_id=modinfra-0000-thmaure&tabs=azureportal)


## Connect Server in Azure via Azure Arc

Als je een server die on-premises of bij een andere cloudprovider staat wilt verbinden met Azure via Azure Arc, ga je in de Azure Portal naar Azure Arc Center en selecteer je Azure Arc-servers. Hier kun je op de knop **Toevoegen** klikken en kun je de onboarding-wizard doorlopen. Over een paar weken zal ik nog een blog online brengen hoe je Azure Arc configureert voor je on-premise omgeving.

![Image](/Images/RunPowershellHybrid/ArcOnboarding.jpg)


## Creëren van Hybrid Worker Group

Nu kun je Hybrid Worker Group maken en onderhouden voor het uitvoeren van taken die je on-premise ook wilt uitvoeren. Dit biedt veerkracht om taken uit te voeren voor meerdere Hybrid Workers. Met op extensies based Hybrid Workers (preview) kunnen zowel Azure-machines als niet-Azure-machines (via Arc-server) worden beheerd via ARM-sjablonen en -beleid.

Als je een hybrid worker wilt maken (die Windows- en Linux-servers kan zijn) open dan het Automation account en ga naar Hybrid Worker groups.
![Image](/Images/RunPowershellHybrid/HWG1.jpg)

Klik hierna op **Create Hybrid Worker Group**

![Image](/Images/RunPowershellHybrid/HWG3.jpg)

Vul in: 
~~~

Name = Logische naam voor HWG
Use Hybrid Worker Credentials = Custom
Choose Hybrid Worker Credentials = Add Credentials

~~~

![Image](/Images/RunPowershellHybrid/HWG2.jpg)

Gebruik een service account met de juiste rechten die op de machines het script mag uitvoeren.
Ga nu het naar het tabblad Hybrid Workers en klik op **Add Machines**

![Image](/Images/RunPowershellHybrid/HWG4.jpg)

Je krijgt nu je machines te zien.

Selecteer de machine waarop je Hybrid Worker wilt configureren en klik op **Add**

![Image](/Images/RunPowershellHybrid/HWG5.jpg)


Hierna kun je klikken op **Review + Create**

![Image](/Images/RunPowershellHybrid/HWG6.jpg)

Hierna zal de Hybrid Extension automatisch op deze machine geïnstalleerd worden.

![Image](/Images/RunPowershellHybrid/HWG7.jpg)

Mocht je willen, kun je hier ook altijd nog meer servers toevoegen maar wij houden het hier even bij één.

-[Deploy an extension-based Hybrid Runbook Worker in Azure Automation](https://docs.microsoft.com/en-us/azure/automation/extension-based-hybrid-runbook-worker-install?tabs=windows)


## Creëren van een nieuwe Azure Automation Runbook.
Je kunt een nieuw Azure Automation runbook maken, waarin het script wordt gehost dat je gaat uitvoeren. 
Afhankelijk van de versie van runbookruntime die je kiest, moet je ervoor zorgen dat deze versie ook op de Hybrid Worker server staat. (Denk aan Python en Powershell 7.0)

Ga in je Automation account naar Runbooks en klik daar op **Create a Runbook**

![Image](/Images/RunPowershellHybrid/Runbook1.jpg)

Hierna moet je een aantal default gegevens invullen.

Vul in: 
~~~

Name = Logische naam voor het Runbook
Runbook Type = (kies uit Powershell,Python,Powershell Workflow,Graphical Powershell,Graphical Powershell Workflow) Wij kiezen hier voor Powershell.
Runtime Version = (kies uit 7.2,7.1,5.1) Wij kiezen hier voor 5.1 omdat die by default al op servers staan.

~~~
![Image](/Images/RunPowershellHybrid/Runbook2jpg.JPG)

In het scherm wat nu naar voren komt kun je je powershell script invoeren. Wij kiezen er nu hier even voor om een txt bestand te deployen op de desktop.

Vul in: 
~~~

"from azure" | out-file "C:\Users\*username*\Desktop\hybrid.txt"

~~~

hierna klik je eerst op **Save** en daarna op **Publish**

 ![Image](/Images/RunPowershellHybrid/Runbook3.JPG)


## Run en Schedule Powershell Script als Azure Automation Runbook via Hybrid Worker group op een Azure Arc machine.

Je kunt nu je PowerShell-scripts uitvoeren en schedule als Azure Automation-runbook. Dit gaat via de hybrid workergroups die zijn verbonden met Azure Arc. 
Wanneer je het runbook uitvoert, kun je nu selecteren op welke hybrid worker group dit script moet worden uitgevoerd.


 ![Image](/Images/RunPowershellHybrid/Test1.JPG)

 Via Output kun je nu zien wat er gebeurd.

 ![Image](/Images/RunPowershellHybrid/Test2.JPG)

 Je kunt dit runbook nu ook Schedule. Dit is heel eenvoudig door te klikken op **Link to Schedule** en selecteer of creër de schedule met zijn parameters.
 
 
  ![Image](/Images/RunPowershellHybrid/Test3.JPG)

