---
Author: "Andy Clifton"
title: "Password Protection"
description: "Password Protection"
date: 2023-04-18
tags: ["emoji"]
thumbnail: "/img/hugo.jpg"
---

# Password Protection

### ==> Disclaimer: work in progress <==

## Introduction

Complex passwords are a basic requirement to protect your systems against a cyber attack.
Even today, most cyber attacks could be prevented if users would use a more complex and, above all, less guessable password.
Cyber attackers always go for the low hanging fruit first. So with a password protection policy you can at least make it a lot harder for them.
In an on-premises environment we can enforce complex passwords via Group Policy, but with this method we cannot block specific password combinations. That is possible with password protection.
Azure AD now supports a banned password list and smart lockout for both Azure AD and on-premises AD in a hybrid environment. (Smart lockout uses cloud intelligence to detect password spraying.)

Let's take a look at the setup of Azure AD password protection to secure the hybrid environment.


## Setup Azure AD Password Protection

Log in to the Azure portal and click **Azure Active Directory**

![Image](/Images/PasswordProtection/AAD.png)

Open **Authentication methods**
.

![Image](/Images/PasswordProtection/AuthenticationMethods.png)

4)	A new window now opens where you can configure the password protection settings.

Fill in: 
~~~
Lockout Threshold = how many attempts before the account is locked (by default 10)
Lockout duration in seconds = time during which you cannot log in (by default 60 seconds)

In Custom banned password :
Enforce custom list  = Yes
Custom banned password list = here you enter all the passwords you do not want to be allowed, 
                              think of password, welcome, company name, first names, last names, postal codes, etc.

~~~

![Image](/Images/PasswordProtection/customsmartlockout.png)

5)	To also apply the policy to your on-premises AD environment, fill in the following:

~~~
Enable password protection on Windows Server Active Directory = Yes
Mode = Enforced or Audit (in audit mode it will only create log entries)
~~~
![Image](/Images/PasswordProtection/PPAD.png)

Click **Save** to save all changes.

![Image](/Images/PasswordProtection/save.png)


## Installing the Azure agents on on-premises servers.

To apply password protection to your on-premises environment as well, we need to install 2 Azure components.

1. Azure AD password protection proxy service
2. Azure AD password protection DC agent

How these agents work is explained perfectly in the picture below.
![Image](/Images/PasswordProtection/adpp.png)

Before we install the Azure AD password protection service, we need to keep the following in mind.
~~~
-	At the moment you are allowed to install 2 proxy servers per forest.
-	Installing it on a DC is supported, but that server must have an internet connection.
-	To register the service you need Domain Admin rights.
-	Proxy servers must have a continuous communication path to the DC agent.
-	You need a Tenant administrator account to configure the proxies in Azure.
~~~
Start of the installation:


1. **Log in** to the server as Domain Admin
2. Go [here](https://www.microsoft.com/download/details.aspx?id=57071) and download AzureADPasswordProtectionProxy.msi
![Image](/Images/PasswordProtection/download.png)

3.Double-click the msi file and start the installation.

  ![Image](/Images/PasswordProtection/msi.png)

4. To check whether the service is running: open PowerShell and enter the following lines:

````
Import-Module AzureADPasswordProtection
Get-Service AzureADPasswordProtectionProxy | fl

````
![Image](/Images/PasswordProtection/powershell.png)

5. Now we need to register the proxy. Do this in PowerShell with the following line:

````
$ADadmin = Get-Credential ( You need your Global admin account for this)
Register-AzureADPasswordProtectionProxy -AzureCredential $ADadmin.
````
The next step is installing the Azure AD password protection DC agent (this must be installed on a domain controller, and this installation requires a reboot).

6. **Log in** to the server as Domain Admin
7.  Go [here](https://www.microsoft.com/download/details.aspx?id=57071) and download AzureADPasswordProtectionDCAgent.msi.
8.Double-click the msi file and start the installation.
![Image](/Images/PasswordProtection/msi2.png)
9. After the installation you need to reboot

![Image](/Images/PasswordProtection/reboot.png)

The installation is now complete and you can test whether the existing passwords are rejected.


