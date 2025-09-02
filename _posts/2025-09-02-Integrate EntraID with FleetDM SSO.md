---
title: Integrate EntraID with FleetDM SS0
date: 2025-09-02 17:00:00 
last_modified_at: 2025-09-01 17:00:00
categories: [Fleet DM, osquery, Cybersecurity Engineering, EntraID, AAD, Azure, IAM]
tags: [fleet dm,osquery,entra id,iam,aad] # TAG names should always be lowercase
comments: true
image:
  path: ../media/post9/
  alt: "FleetDM and osquery deployment"
author: Ahmed BAHYA
excerpt: "Learn how to integrate your FleetDM server with already existing Entra ID tenant for seamless SSO login of you security and incident response team"
description: "Comprehensive guide to FleetDM and osquery. Learn how to deploy FleetDM with Docker, manage osquery agents, and integrate with your cybersecurity monitoring stack for enhanced visibility and defense."
keywords: [fleetdm, osquery, docker, cybersecurity, endpoint monitoring, purple team, siem, detection, response]
canonical_url: https://b2hu.me/posts/FleetDM-and-Osquery
---

Hello! After setting up FleetDM in our security architecture, the next critical step is managing identities and access — deciding who can log in and what they can access. That’s where an Identity Provider (IdP) comes in. Since I’m currently preparing for the SC-300 certification, I’ll be using Entra ID (formerly Azure AD) as the IdP for this guide.

> go back to the previous post to learn how to install and deploy FleetDM and its agents accross your org [previous post](https://b2hu.me/posts/FleetDM-and-osquery/) for background.  
{: .prompt-info }

## Why Use EntraID SSO :
Imagine you’re running a large organization with thousands of employees and hundreds of applications. You’ve just deployed FleetDM to control and manage all your devices — great! But now comes the challenge: creating and managing all the appropriate user accounts directly in FleetDM. Doing this manually, even just 50 times, quickly becomes a nightmare.

This is where Entra ID SSO comes in. Instead of recreating accounts, your teams can log in to FleetDM using their existing corporate identities. With Entra ID, you also gain centralized control through features like Multi-Factor Authentication (MFA), Conditional Access policies, and fine-grained access management, ensuring only the right people have the right access under the right conditions.

## Integration

### Register FleetDM as an Enterprise Application in Entra ID

- Go to your Entra ID tenant : [Entra ID Admin Portal](https://entra.microsoft.com/)
- Access Entreprise Application and Add a new Application
- Click **New Application**, insert you App name and select **Integrate any other application you don't find in the gallery (Non-gallery)** then create.
- After creating the App Go to single sign-on and choose **SAML** as the sso method 
- In Edit mode of **Basic SAML Configuration** make sure :
  
  1. Add Identifier as **fleet**
  2. Add Reply URL as ***https://fleet.b2hu.me:8080/api/v1/fleet/sso/callback*** (change besed on you server config)
  3. Copy the **Metadata URL**

N.B : if your server url is not a domain name (like my case make sure you add it in hosts file)

![prof](/./media/post9/entr_app.png)

---
### Fleet Server Configuration

- Navigate to **Settings > Integration > Single sign-on options**
- Enable SSO and replace IdP name with **Entra ID** and Entity ID with **fleet** and image url is optional.
- Paste the copied metadata url and save

overall the config will look like this : 

![config](/./media/post9/fleet.png)

## JIT Provisioning and tenant user assignement 

When a security team member tries to log in to Fleet for the first time (without a pre-existing account), Fleet can use Just-In-Time (JIT) user provisioning to automatically create their profile during the login process. However, this feature is only available in Fleet Premium.

Since we’re working with the Fleet Free edition in this blog post, we’ll take a different approach: we’ll manually create the user in Fleet first, then mark that account as an SSO user. This ensures that authentication is delegated to Entra ID while account creation remains under our control.

But before we can link FleetDM with Entra ID, we need to make sure our tenant users are assigned to the Enterprise Application.
- In EntraID Navigate to **Entreprise App > FleetDM > Users and Groups**
- add you security Team users or groups hat will be allowed to access **FleetDM**

This step ensures that when the assigned Entra ID users attempt to log in, Fleet recognizes their accounts and routes authentication through SSO.

then in the Fleet server go to **Manage Users > Add User** :

![add_user](/./media/post9/add_user.png)

## SSO Test
the Sign-in options : 

![ss](/./media/post9/sso.png)

after choosing to sign-in with Entra you'll be redirected to a MS login portal, and upon a successful login we'll be redirected again to the fleet dashboard : 

![succ_ss](/./media/post9/succ_ss.png)

---

Thank you for Reading! feel free to contact me if you have a question.