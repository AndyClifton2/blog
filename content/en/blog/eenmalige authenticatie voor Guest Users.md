---
Author: "Andy Clifton"
title: "One-time passcode for Azure AD B2B Guest Users"
description: "One-time passcode for Azure AD B2B Guest Users"
date: 2024-03-01
tags: ["AzureB2B"]
thumbnail: "/img/B2b.png"
---



# One-time passcode for Azure AD B2B Guest Users

### ==> Disclaimer work in progress <==

## Introduction
Azure AD B2B allows external users to use the applications, services and data of the tenant.
To grant guests access, they need to have an Azure AD account, a Microsoft account or Google Federation (for @gmail.com and @googlemail.com users).
If a user has none of these accounts, a connection can still be made using a one-time passcode.
This code is sent to the guest user's email address and remains valid for 30 minutes.
Once the guest user has authenticated, the session stays valid for 24 hours; after that the user has to request a new code to sign in again.
Users of a one-time passcode must use the links below when they authenticate:

````
https://myapps.microsoft.com/?tenantid=<tenant id> 
https://portal.azure.com/<tenant id>
https://myapps.microsoft.com/<verified domain>.onmicrosoft.com
````
In the above, the tenant id of course needs to be replaced with the tenant id of your own environment. Verified domain in turn needs to be replaced with the domain name of your tenant.
  
  
The option to use one-time passcodes has been available since October 2021 and can be used for existing tenants and tenants yet to be created.
  


## Enable the one-time passcode feature
  
The first step is to enable the one-time passcode feature for guest users.

Sign in to the Azure portal and go to **Azure Active Directory**.

![Image](/Images/OTP/AAD.png)

Now go to **External Identities --> All Identity Providers**

![Image](/Images/OTP/externalidentities.png)
  
In the configured identity providers list, click **Email one time passcode** 
  
![Image](/Images/OTP/passcode.png)

Select **Yes** under Email one-time passcode for guests.
  
![Image](/Images/OTP/passcode2.PNG)
  
We have now enabled the feature, so it is time to test the one-time passcode.

I am going to invite a user and make them a member of the group OCTID-Sales.
  
# Create an Azure AD B2B Guest User
  
 Go to Azure Active Directory -> Users and then click **New User**
 
 Now also click **Invite external user**
  
![Image](/Images/OTP/newuser.png)    

On the new user page, fill in the following details:
  
~~~
  
Template =  Invite user
  
Identity:

Name = Full name of the guest user
Email address = Email address of the guest user
First Name = First name of the guest user
Last Name =  Last name of the guest user
  
~~~
  
![Image](/Images/OTP/newuser1.png) 
  
**Click** Groups, then **select** the group OCTID-Sales and then **click** Save
  
![Image](/Images/OTP/newuser2.png) 
  
**Click** Invite 
  
![Image](/Images/OTP/invite.png) 
  
The user receives an email with an invitation for this.
  
![Image](/Images/OTP/mail.png)
  
The guest user now gets a message that a code will be sent to them.
  
![Image](/Images/OTP/sendcode.png)
  
Within a few seconds you receive an email that looks like the one below:

![Image](/Images/OTP/mail1.png)
  
 
You can enter this code and then you are signed in to the company's My Apps.
This is how you configure the one-time passcode for B2B guests.  
