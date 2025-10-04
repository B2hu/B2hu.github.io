---
title: Cortex MCP Server - The Future of Threat Intelligence
date: 2025-10-03 08:00:00 
last_modified_at: 2025-10-03 01:00:00
categories: [Suricata]
tags: [suricata,soc,ips,nsm,network security,wazuh] # TAG names should always be lowercase
comments: true
image:
  path: /media/post12/suricata.jpg
  alt: "Suricata IPS Mode"
author: Ahmed BAHYA
keywords: suricata,soc,ips,nsm,network security,wazuh]
---
Hi everyone,

Lately I’ve been digging into the world of IDS, IPS, and NSM. While researching, I noticed that there are plenty of tutorials and guides explaining how to run Suricata as an IDS (Intrusion Detection System). But when it comes to configuring it as an IPS (Intrusion Prevention System), the resources are surprisingly limited.

Since IPS is where detection turns into active defense, I thought it would be valuable to share what I’ve learned. In this post, I’ll walk you through how to set up Suricata in IPS mode — from the initial configuration to actually blocking malicious traffic in real time.

## A Refresher : IDS Vs IPS

- **IDS**: Monitors traffic and generates alerts when suspicious activity is detected.

- **IPS**: Goes a step further by blocking or dropping malicious traffic before it reaches its target.

This distinction is important because Suricata can work in both modes. (only one mode)

### Suricata IPS modes

besed on suricata docs [here](https://docs.suricata.io/en/suricata-8.0.1/ips/setting-up-ipsinline-for-linux.html), there are two operation modes :
```text
The inline operations are either on layer 2 (bridge, for example using AF_PACKET or DPDK) or on layer 3 (routing, for example in NFQueue or IPFW).
```
- **Layer 2 IPS Mode** :

**Suricata** attaches directly to a network interface (using **AF-PACKET** or **DPDK**) and inspects packets at the ***Ethernet frame level***. This allows it to drop malicious packets before they reach the kernel network stack, making it high-performance and suitable for full traffic inspection on busy networks. **Layer 2 IPS** is commonly used in bridged setups, such as transparent firewalls or network taps.

- **Layer 3 IPS Mode** :

works by having Suricata receive packets from **iptables** or **nftables queues** (***NFQUEUE***). It inspects them at the IP layer and decides whether to accept or drop them. Only traffic explicitly queued by the firewall is inspected, giving more selective control over what to block. This mode is easier to deploy on a host without bridging interfaces and is ideal for host IPS or lab/testing environments.

--- 
In this blog post, we will focus on Layer 3 IPS using NFQUEUE, because it allows for more selective packet inspection. Only traffic explicitly routed through iptables will be processed by Suricata, making it ideal for controlled, host-based, or lab environments where you want to block specific flows without affecting all network traffic.

## Setting Up Suricata as an IPS

you need to already have an instance of suricata up and running(use this [link](https://docs.suricata.io/en/latest/install.html) to install it). Once Suricata is installed, we can configure it to operate in Layer 3 IPS mode using NFQUEUE. This involves two main steps: setting up iptables rules to queue traffic and configuring Suricata to process that queue and enforce drop or reject rules.

### Create Suricata Drop/Reject Rules

To make Suricata act as an IPS, you need to define rules that actively drop or reject malicious traffic. Suricata rules are stored inside `/var/lib/suricata/rules/local.rules` (make sure to indicate local rules in the config file).

>drop rules only drop packets, which may result in timeouts for the client. On the other hand, reject rules actively send ICMP or TCP RST responses back to the client, immediately informing them that the connection was blocked.
{: .prompt-tip}

for this blog, we'll create a rule that blocks anonymous ftp logins

```text
reject ftp any any -> any 21 (msg:"FTP anonymous login attempt"; flow:to_server,established; ftp.command ;content:"USER"; ftp.command_data; content:"anonymous";nocase; classtype:attempted-recon; sid:99999; rev:1;)
```
As you can see, this rule will reject any **FTP connection** that attempts to log in as **anonymous**, regardless of whether it’s legitimate or not. The rule works by inspecting the **FTP USER** command and sending a TCP reset back to the client, effectively stopping the connection.

### Create NFQUEUE

Setting up an NFQUEUE allows the kernel to send selected packets from iptables or nftables to user-space applications, like Suricata. Once in user space, Suricata can inspect each packet and decide whether to accept it (letting it continue to its destination) or drop/reject it (blocking the traffic). This mechanism enables selective, layer 3 IPS functionality, giving you fine-grained control over which traffic is processed and blocked without affecting the entire network interface.

To forward FTP traffic to Suricata for inspection, we can create an NFQUEUE rule in iptables:

```shell
iptables -I INPUT -p tcp --dport 21 -j NFQUEUE
```
This rule tells the kernel to send any incoming FTP connection to the NFQUEUE. Once queued, Suricata running in IPS mode can decide whether to accept or reject the connection based on your rules.

You can verify the rule is in place using:

```shell
iptables -L INPUT 
```
sample output :
```text
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         
NFQUEUE    tcp  --  anywhere             anywhere             tcp dpt:ftp NFQUEUE num 0
```
This confirms that all incoming FTP traffic on port 21 is now being queued for Suricata to process.

### Enable Suricata nfq 

To allow Suricata to process packets from the NFQUEUE, you need to enable NFQ mode in its configuration. Open the main Suricata configuration file: `/etc/suricata/suricata.yaml` and change nfq mode to accept, like this :
```yaml
nfq:
  mode: accept
```
confirm by running ***suricata --build-info***.

## Start IPS Mode 
Normaly Suricata is run with **--af-packet** argument on the service file, and to start with nfq mode simply Suricata needs to be started with **-q <NFQ-N°>**.
run
```shell
suricata -q 0 # many nfqueue can be passed
```
## IPS Testing 

let's try to access the ftp server using anonymous (enable anonymous login if you haven't) :

```shell
root@ahmedbh-VirtualBox:~# ftp localhost 
Connected to localhost.
220 (vsFTPd 3.0.5)
Name (localhost:root): anonymous
331 Please specify the password.
Password: 
421 Service not available, remote server has closed connection.
ftp: Login failed
ftp> 
```
we notice that the login failed due to a closed connection, which indicates that suricata detected the login attempt and sent a **tcp reset** packet to us.
let's check the `fast.log` file to see the alert :

```shell
cat /var/log/suricata/fast.log 

10/04/2025-02:54:21.140929  [wDrop] [**] [1:99999:1] FTP anonymous login attempt [**] [Classification: Attempted Information Leak] [Priority: 2] {TCP} 127.0.0.1:42822 -> 127.0.0.1:21
```
Thank for Reading.