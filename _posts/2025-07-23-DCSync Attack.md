---
title: DCSync Attack - MITRE ATT&CK T1003.006
date: 2025-07-24 10:32:00 
categories: [Purple Team,Active Directory & Kerberos Abuse]
tags: [active directory,asrep roasting,hashcat]     # TAG names should always be lowercase
comments: true
image:
  path: /./media/post5/kerberos.png
---
Hi,Welcome again to a new Kerberos abuse blog post. In this blog post, I will demonstrate and simulate a very classic yet interessting Active Directory and Kerberos abuse attack: the **DCSync** attack. So, let's get started!
> In this post, I won’t explain the Kerberos protocol. You can refer to the previous post where I already covered it, or click this link: <a href="https://b2hu.me/posts/AS-REP-Roasting-Attack">here</a>.  
{: .prompt-info }


## DCSync :
**DCSync** is a post-exploitation technique where an attacker impersonates a Domain Controller (DC) and abuses the **Directory Replication Service Remote Protocol (DRSR)** to extract sensitive information, such as **NTLM password hashes, Kerberos tickets, and even KRBTGT account credentials** — the keys to forging Golden Tickets.

In a typical Active Directory environment, replication is the process by which Domain Controllers synchronize directory data (users, groups, passwords) between each other to remain consistent. This replication is normally performed by trusted DCs using special replication permissions.

DCSync exploits this trust model. If an attacker compromises an account that has replication privileges, such as:

- **Domain Admin**

- **Enterprise Admin**

Or any account with these ACLs:

- **DS-Replication-Get-Changes**

- **DS-Replication-Get-Changes-All**

…they can request directory replication data as if they were a DC.
## ATTACK Simulation (DCSync Attack Demonstration) :
In this practical demonstration, we'll simulate a **DCSync Attack** attack in our test environment and identify **key Indicators of Attack (IoAs)** plus any **artifacts** the attack leaves that would alert defenders to such malicious activity.

**Lab Environment Details:**

- Target Domain: **iccn.ma**

- Vulnerable Account: **iccn.ma/jdoe** 

the **jdoe** user must have **DS-Replication-Get-Changes** and **DS-Replication-Get-Changes-All** in order for this attack to work

![screen_2](/./media/post6/jdoe_drsr.png)

### Attack
to perform DCSync a lot of red team tools can be used whether its for a windows machine (Rebeus,mimikatz), or linux attack machine (impacket scripts), i really like using impacket scripts so for this blog post we'll use the impacket script **secretsdump.py**:
```shell
impacket-secretsdump iccn/jdoe@10.0.2.16
```
you will be prompted to enter the user's password (as said befor DCSync is a post exploitation technique),the output wil be something similair to this
```shell
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

Password:
[-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied 
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b43.....
Guest:501:aad3b435b51404eeaad3b435b51404e.....
krbtgt:502:aad3b435b51404eeaad3b435b51404.....
iccn.ma\jdoe:1103:aad3b435b51404eeaad3b43.....
....
```
from the result above we can further exeute post exploitation or persistence attacks like child-domain escalationn or golden/silver tickets
## Detect & respond to the Attack Using _Wazuh_

Detecting DCSync is easy because each Domain Controller replication generates an event
with the ID **4662** . We can pick up abnormal requests immediately by monitoring for this
event ID and checking whether the initiator account is a Domain Controller (all machine accounts in active direcory end with **$**). Here's the event
generated from earlier when we ran the impacket script ; it serves as a flag that a user account is
performing this replication attempt:

![screen_3](/./media/post6/event.png)

The goal is to detect and alert when a **non-Domain Controller account** attempts to perform **directory replication**, which is a strong indicator of a DCSync attack. Using **Wazuh**, we monitor for Event ID **4662** with **replication-related GUIDs**, and trigger alerts if the account name does not end with **$**, meaning it isn’t a DC. This helps identify unauthorized attempts to extract credential data from Active Directory.

Let’s analyze the following rule:

```xml
  <!-- This rule detects DCSync attacks -->
  <rule id="110003" level="12">
    <if_sid>60103</if_sid>
    <field name="win.system.eventID">^4662$</field>
    <field name="win.eventdata.properties" type="pcre2">{1131f6aa-9c07-11d1-f79f-00c04fc2dcd2}|{19195a5b-6da0-11d0-afd3-00c04fd930c9}</field>
    <match name="win.eventdata.SubjectUserName" negate="yes">\$$</match>    
    <options>no_full_log</options>
    <description>Directory Service Access. Possible DCSync attack from Username: $(win.eventdata.SubjectUserName)</description>
    <mitre>
        <id>T1003.006</id>
    </mitre>
    <info type="link">https://attack.mitre.org/techniques/T1003/006/</info>
  </rule>
````
This child rule (ID: 110003) is triggered when its parent rule (60103) is matched. It includes conditions aligned with our detection objective:

1. It only triggers if the **event properties** matches properties related to replication services.
2. It ensures the **SubjectUserName** doesn't end with a **$** sign (non DC account).

The generated alert includes useful metadata like the MITRE ATT&CK technique ID and a link for further reading.

Next, we simulate the attack from a Linux machine with the IP address 10.0.2.20:

![screen_4](/./media/post5/ifconfig.png)
After using the Impacket script (as in the previous attack simulation). In the Wazuh Dashboard, we can observe that an alert was successfully generated:
![screen_5](/./media/post6/alert.png)
we can further inspect the content of the alert using drilldowns 
![screen_6](/./media/post6/alert_dd.png)

## Conclusion
This detection strategy effectively leverages Wazuh’s rule customization capabilities to identify **DCSync** attacks. By creating a child rule that monitors for directory replication access **(Event ID 4662)** using replication-related extended rights, and filtering out legitimate Domain Controllers (accounts ending with $), we generate precise and meaningful alerts. This approach allows for targeted detection of unauthorized replication attempts without modifying Wazuh’s built-in rules, preserving maintainability and flexibility. As demonstrated, a simulated DCSync attempt from a non-DC account successfully triggered an alert, confirming the effectiveness of this method in detecting credential extraction activity in Active Directory environments.

Thank you for your attention, if you have any suggestion or question feel free to write it in the comments, or send me an email.

