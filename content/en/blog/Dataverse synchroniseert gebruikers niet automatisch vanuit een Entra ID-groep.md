---
Author: "Andy Clifton"
title: "Dataverse synchroniseert gebruikers niet automatisch vanuit een Entra ID-groep"
description: "Dataverse synchroniseert groepsleden niet automatisch vanuit een Entra ID-beveiligingsgroep — leer hoe je de sync afdwingt met PowerShell en Add-AdminPowerAppsSyncUser.."
date: 2026-06-11
tags: ["Azure", "PowerShell", "CAF", "Terraform", "Governance"]
thumbnail: "/img/dataverse-sync-thumbnail.png"
---

## Voorwoord
Bij het aanmaken van een Power Platform-omgeving kun je een Entra ID-beveiligingsgroep koppelen. Logisch zou zijn dat gebruikers die in die groep zitten automatisch gesynchroniseerd worden naar Dataverse — zodat ze meteen toegang hebben. In de praktijk blijkt dat niet altijd het geval te zijn. Dataverse synchroniseert groepsleden niet automatisch op het moment dat de omgeving wordt aangemaakt of wanneer er nieuwe gebruikers aan de groep worden toegevoegd.
Dit artikel laat zien hoe je dit probleem oplost met een PowerShell-script dat de synchronisatie handmatig afdwingt.

<ins>Het probleem: ontbrekende gebruikers in Dataverse</ins>

Wanneer je een Power Platform-omgeving aanmaakt en daarbij een Entra ID-beveiligingsgroep koppelt, verwacht je dat de gebruikers in die groep direct beschikbaar zijn in Dataverse. Microsoft documenteert ook dat de beveiligingsgroep de toegang tot de omgeving beperkt — maar de stap waarbij gebruikers daadwerkelijk als SystemUser in Dataverse worden geregistreerd, is een aparte synchronisatieactie.
Die actie vindt niet automatisch en direct plaats. Het gevolg: gebruikers kunnen zich niet aanmelden in de omgeving, apps werken niet, en beheerders krijgen foutmeldingen die niet meteen naar de oorzaak wijzen.
Het is een bekend fenomeen bij het beheer van DEV- en TEST-omgevingen, waarbij je snel toegang wilt verlenen na omgevingscreatie of na het toevoegen van nieuwe teamleden.

<ins>De oplossing: handmatig synchroniseren via PowerShell</ins>
De Power Apps Administrator-module bevat de cmdlet Add-AdminPowerAppsSyncUser. Hiermee forceer je de synchronisatie van een specifieke gebruiker naar een Power Platform-omgeving. Door deze cmdlet te combineren met de Microsoft Graph-module kun je alle leden van een Entra ID-groep in één keer synchroniseren.
Vereisten
Zorg dat de volgende modules beschikbaar zijn en dat je verbonden bent:
powershell# Microsoft Graph (voor groepsleden ophalen)
Connect-MgGraph -Scopes "GroupMember.Read.All", "User.Read.All"

# Power Apps Administrator (voor de sync)
Connect-PowerAppsAccount
Het script
powershell$groupId       = "e948d229-3b58-49cb-bb0a-6fb1038dfc65"
$environmentId = "60a5220e-4f56-ece5-980f-5c7ebfe1ec58"

$members = Get-MgGroupMember -GroupId $groupId -All

foreach ($member in $members) {
    $user = Get-MgUser -UserId $member.Id -Property DisplayName, UserPrincipalName
    Write-Host "Syncing: $($user.DisplayName)" -ForegroundColor Cyan

    $result = Add-AdminPowerAppsSyncUser `
        -EnvironmentName $environmentId `
        -PrincipalObjectId $member.Id

    Write-Host "  Code: $($result.Code) - $($result.Description)" `
        -ForegroundColor $(if ($result.Code -eq 200) { "Green" } else { "Yellow" })
}

Write-Host "`n✅ Sync compleet" -ForegroundColor Green
Wat doet het script?
Het script doorloopt de volgende stappen:

Groepsleden ophalen — via Get-MgGroupMember worden alle leden van de opgegeven Entra ID-groep opgehaald.
Gebruikersgegevens ophalen — voor elk lid wordt de weergavenaam en UPN opgehaald zodat de uitvoer leesbaar is.
Synchronisatie forceren — Add-AdminPowerAppsSyncUser registreert de gebruiker als SystemUser in Dataverse.
Resultaat tonen — de HTTP-statuscode en beschrijving worden getoond. Een code 200 betekent succes; andere codes verdienen nader onderzoek.


<ins>Praktische toepassing</ins>
Dit script is met name nuttig in de volgende situaties:

Direct na het aanmaken van een nieuwe Power Platform-omgeving die gekoppeld is aan een Entra ID-groep.
Nadat nieuwe collega's zijn toegevoegd aan de Entra ID-groep en zij direct toegang nodig hebben.
Als onderdeel van een onboarding- of omgevingsprovisioning-runbook.

De $groupId en $environmentId zijn eenvoudig te variabiliseren of uit een configuratiebestand te laden als je dit in een pipeline of bredere automatisering wilt opnemen.

<ins>Achtergrond: waarom synchroniseert Dataverse niet automatisch?</ins>
Dataverse beheert zijn eigen gebruikerstabel (SystemUser) los van Entra ID. De koppeling via een beveiligingsgroep in Power Platform bepaalt wie toegang mag hebben, maar voegt gebruikers niet proactief toe aan de Dataverse-tabel. De synchronisatie gebeurt normaliter lazy: pas op het moment dat een gebruiker voor de eerste keer inlogt in de omgeving, wordt zijn of haar record aangemaakt.
In geautomatiseerde of beheerde omgevingen — waarbij je gebruikers of beveiligingsrollen vooraf wilt toewijzen — is dit gedrag onpraktisch. Het handmatig triggeren via Add-AdminPowerAppsSyncUser omzeilt die wachttijd.

Tot slot
Het koppelen van een Entra ID-groep aan een Power Platform-omgeving geeft je controle over wie toegang heeft, maar garandeert niet dat die gebruikers meteen actief zijn in Dataverse. Met bovenstaand script dwing je die synchronisatie af, zodat gebruikers direct aan de slag kunnen en je beveiligingsrollen meteen kunt toewijzen.
Een kleine maar impactvolle stap in gestroomlijnd omgevingsbeheer.