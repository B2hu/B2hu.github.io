---
title: Kerberoast Attack - MITRE ATT&CK T1558.003
date: 2025-08-12 10:32:00 
last_modified_at: 2025-08-12 10:32:00
categories: [Purple Team,Active Directory & Kerberos Abuse]
tags: [active directory,kerberoast,hashcat,kerberos,mitre att&ck,cybersecurity,penetration testing,purple team,service principal names,spn]     # TAG names should always be lowercase
comments: true
image:
  path: /./media/post5/kerberos.png
  alt: "Kerberoast Attack - Kerberos Service Ticket Exploitation"
author: Ahmed BAHYA
excerpt: "Learn about Kerberoast attack (MITRE ATT&CK T1558.003), a post-exploitation technique to extract service account credentials from Active Directory. Complete walkthrough with detection and response using Wazuh SIEM."
description: "Comprehensive guide to Kerberoast attack (MITRE ATT&CK T1558.003). Learn how attackers exploit Service Principal Names to extract service account credentials, practical demonstration, and detection using Wazuh SIEM for cybersecurity defense."
keywords: [kerberoast, active directory, mitre att&ck, t1558.003, cybersecurity, penetration testing, purple team, service principal names, spn, service tickets, hashcat, john the ripper, wazuh, siem, detection, response]
canonical_url: https://b2hu.me/posts/Kerberoast-Attack
---
Hello everyone, welcome back! In this blog post, I will demonstrate and simulate another classic Active Directory and Kerberos abuse attack: the **Kerberoast** attack. This is a post-exploitation technique that targets service accounts (or any account with SPN/SAN if you know what i mean 😏) in Active Directory environments. So, let's get started!

> In this post, I won't explain the Kerberos protocol in detail. You can refer to the previous post where I already covered it, or click this link: <a href="https://b2hu.me/posts/AS-REP-Roasting-Attack" target="_blank">here</a>.  
{: .prompt-info }

## What is Kerberoast?

**Kerberoast** is a post-exploitation attack technique that targets **Service Principal Names (SPNs)** in Active Directory. Unlike AS-REP Roasting which targets user accounts with pre-authentication disabled, Kerberoast focuses on service accounts that are configured with SPNs.

### Service Principal Names (SPNs)

SPNs are unique identifiers for services running on servers. They follow the format: `service/hostname:port/realm`. Common examples include:

- `HTTP/webserver.iccn.ma`
- `MSSQLSvc/sqlserver.iccn.ma:1433`
- `CIFS/fileserver.iccn.ma`
- `LDAP/dc.iccn.ma`

Kerberoast follows this attack chain :
1. **Initial Access**: The attacker already has a foothold in the domain (compromised user account)
2. **SPN Discovery**: The attacker enumerates service accounts with SPNs
3. **Service Ticket Request**: The attacker requests service tickets for discovered SPNs using their compromised account
4. **Offline Cracking**: The encrypted service tickets are extracted and cracked offline to reveal service account passwords
5. **Privilege Escalation**: The cracked credentials can be used for lateral movement or privilege escalation

## Attack Simulation (Kerberoast Attack Demonstration)
### Rubeus

In this practical demonstration, we'll simulate a **Kerberoast** attack in our test environment, unlike previous posts where i have used linux machines today i practice with a compromised windows machine, we'll also identify **key Indicators of Attack (IoAs)** aka **Artifacts** that would alert defenders to such malicious activity.

The **mssqlsvc** account must be configured with an SPN(use setspn.exe), after this run the tool Rubeus as shown below:

![Service Account SPN Configuration](/./media/post7/Rubeus_exec.png)

### Offline Cracking

The output will contain encrypted service tickets. Save them to a file and crack using Hashcat or John the Ripper:

```shell
# Using Hashcat
hashcat -m 13100 -a 0 kerberoast_tickets.txt /usr/share/wordlists/rockyou.txt

# Using John the Ripper
john --wordlist=/usr/share/wordlists/rockyou.txt kerberoast_tickets.txt --format=krb5tgs
```

Example output:

![jtr](/./media/post7/jtr.png)

### Post-Exploitation Tactics

With the cracked service account credentials, the attacker can now:

1. **Lateral Movement**: Access the service (e.g., SQL Server, file shares)
2. **Privilege Escalation**: Exploit service account privileges
3. **Persistence**: Create additional accounts or modify existing ones
4. **Data Exfiltration**: Access sensitive data through the compromised service

## Detect & Respond to the Attack Using Wazuh

As mensioned in previous posts every service ticket request to the Domain Controller is logged with Event ID **4769**. This event provides crucial information for detecting Kerberoast attacks.

### Understanding Event ID 4769

Event ID 4769 logs service ticket requests and includes:
- **TargetUserName**: The service account being targeted
- **ServiceName**: The SPN being requested
- **TicketEncryptionType**: The encryption type used
- **IpAddress**: Source IP address of the request
- **Status**: Success or failure of the ticket request

### Detection Strategy

While Analyzing the Atrifacts left by Rubeus the ST requests are all RC4 encrypted leaving a pattern of abnormal activity that could be monitored. 
### Wazuh Detection Rule

```xml
<!-- This rule detects Kerberoast attacks using windows security event on the domain controller -->
<rule id="110004" level="12">
  <if_sid>60103</if_sid>
  <field name="win.system.eventID">^4769$</field>
  <field name="win.eventdata.TicketEncryptionType" type="pcre2">0x17</field>
  <options>no_full_log</options>
  <description>RC4 encrypted kerberos ST was requested - Possible Kerberoast attack to service account $(win.eventdata.ServiceName)</description>
  <mitre>
      <id>T1558.003</id>
  </mitre>
  <info type="link">https://attack.mitre.org/techniques/T1558/003/</info>
</rule>
```

### Attack Simulation Results

After executing the Kerberoast attack from IP 10.0.2.20, Wazuh generates alerts:

![Kerberoast Alert](/./media/post7/alert.png)
rule description :
![Kerberoast Alert 2](/./media/post7/alert_2.png)

The alert details show:
- **Event ID**: 4769 (Service Ticket Request)
- **Target Account**: mssqlsvc (service account)
- **Service Name**: MSSQLSVC/dc01.iccn.ma
- **Source IP**: 10.0.2.20 (attacker machine)
- **Status**: Success (0x0)

### Additional Detection Considerations

1. **Baseline Monitoring**: Establish normal patterns of service ticket requests
2. **Rate Limiting**: Monitor for excessive ticket requests from single sources
3. **Password Policy**: Ensure service accounts follow strong password policies
4. **Enforce AES Encryption**: Configure domain policy to require AES encryption for Kerberos tickets and disable RC4 support to prevent Kerberoast attacks
7. **Implement gMSA**: Use **Group Managed Service Accounts (gMSA)** instead of traditional service accounts, as gMSAs have automatically managed complex passwords that change frequently, making them significantly more resistant to offline cracking attacks

## Conclusion

Kerberoast attacks continue to pose a significant threat to Active Directory environments. By implementing the detection strategies outlined in this post using Wazuh SIEM/EDR, organizations can effectively identify and respond to these attacks. The key to success lies in monitoring RC4-encrypted service ticket requests and implementing preventive measures such as enforcing AES encryption and using gMSA service accounts.

I hope you have enjoyed reading this blog post. If you have any questions or need help implementing a similar setup, feel free to ask!


