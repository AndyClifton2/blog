---
Author: "Andy Clifton"
title: "Dataverse does not automatically sync users from an Entra ID group"
description: "Dataverse does not automatically synchronise group members from an Entra ID security group. Learn how to force the sync with PowerShell and Add-AdminPowerAppsSyncUser."
date: 2026-06-11
tags: ["Azure", "PowerShell", "CAF", "Terraform", "Governance"]
thumbnail: "/img/dataverse-sync-thumbnail.png"
---

## Introduction
When you create a Power Platform environment, you can link an Entra ID security group to it. You would expect the users in that group to be synchronised to Dataverse automatically so they have access right away. In practice, that is not always the case. Dataverse does not automatically synchronise group members at the moment the environment is created, or when new users are added to the group.
This article shows how to solve this problem with a PowerShell script that forces the synchronisation manually.

<ins>The problem: missing users in Dataverse</ins>

When you create a Power Platform environment and link an Entra ID security group to it, you expect the users in that group to be available in Dataverse immediately. Microsoft does document that the security group restricts access to the environment, but the step where users are actually registered as a SystemUser in Dataverse is a separate synchronisation action.
That action does not happen automatically and immediately. The result: users cannot sign in to the environment, apps do not work, and administrators get error messages that do not immediately point to the cause.
It is a well known phenomenon when managing DEV and TEST environments, where you want to grant access quickly after creating the environment or after adding new team members.

<ins>The solution: manual synchronisation via PowerShell</ins>
The Power Apps Administrator module contains the cmdlet Add-AdminPowerAppsSyncUser. With it you force the synchronisation of a specific user to a Power Platform environment. By combining this cmdlet with the Microsoft Graph module, you can synchronise all members of an Entra ID group in one go.
Prerequisites
Make sure the following modules are available and that you are connected:
powershell# Microsoft Graph (to retrieve the group members)
Connect-MgGraph -Scopes "GroupMember.Read.All", "User.Read.All"

# Power Apps Administrator (for the sync)
Connect-PowerAppsAccount
The script
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
What does the script do?
The script goes through the following steps:

Retrieve group members: with Get-MgGroupMember all members of the specified Entra ID group are retrieved.
Retrieve user details: for each member the display name and UPN are retrieved so the output is readable.
Force synchronisation: Add-AdminPowerAppsSyncUser registers the user as a SystemUser in Dataverse.
Show the result: the HTTP status code and description are displayed. A code 200 means success; other codes deserve further investigation.


<ins>Practical use</ins>
This script is particularly useful in the following situations:

Right after creating a new Power Platform environment that is linked to an Entra ID group.
After new colleagues have been added to the Entra ID group and need access immediately.
As part of an onboarding or environment provisioning runbook.

The $groupId and $environmentId are easy to turn into variables or to load from a configuration file if you want to include this in a pipeline or broader automation.

<ins>Background: why doesn't Dataverse synchronise automatically?</ins>
Dataverse manages its own user table (SystemUser) separately from Entra ID. The link via a security group in Power Platform determines who is allowed to have access, but does not proactively add users to the Dataverse table. The synchronisation normally happens lazily: only when a user signs in to the environment for the first time is his or her record created.
In automated or managed environments, where you want to assign users or security roles in advance, this behaviour is impractical. Triggering it manually via Add-AdminPowerAppsSyncUser bypasses that waiting time.

Wrapping up
Linking an Entra ID group to a Power Platform environment gives you control over who has access, but does not guarantee that those users are immediately active in Dataverse. With the script above you force that synchronisation, so users can get started right away and you can assign security roles immediately.
A small but impactful step towards streamlined environment management.
