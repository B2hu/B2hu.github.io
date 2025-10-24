---
title: Wazuh Active Response MCP Server - The Future of Threat Intelligence
date: 2025-10-22 15:00:00 
last_modified_at: 2025-09-23 08:00:00
categories: [MCP, Wazuh]
tags: [mcp,soc,llm,incident response,wazuh,soar] # TAG names should always be lowercase
comments: true
image:
  path: /media/post15/wazuh_mcp.png
  alt: "Wazuh Active Response MCP Server"
author: Ahmed BAHYA
keywords: [mcp,soc,llm,incident response,wazuh,soar]
---
In our recent blog posts, we’ve deployed and built several **MCP servers** designed to handle critical SOC operations such as ***alert investigation, threat intelligence***, and more.
Now, we’re shifting our focus to another crucial domain : **incident response.**

Today’s threats are evolving rapidly, and to keep up, incident response requires not only speed but also intelligence and automation. Security teams are constantly overwhelmed by alerts, leaving little time for proactive defense. That’s where the Model Context Protocol (MCP) and Wazuh Active Response come together — marking a new step toward integrating AI agents into security operations.

## How does it work :

Wazuh has a cool capability called **active response**, it allows executing scripts on selected agents (or all agents by default). This means that when a threat is detected, Wazuh can automatically take action, such as **blocking an IP**, **disabling a user account**, or **isolating a host**. These active response scripts are usually invoked based on certain conditions, like when a specific **rules ID** are flagged.

Active Response also provides a ***RESTful API*** that accepts the **command name** and arguments to pass to the script. This means we can execute any command on demand, regardless of the triggering conditions.
>[More on wazuh active response capability](https://documentation.wazuh.com/current/user-manual/capabilities/active-response/index.html).
{: .prompt-tip}

### Wazuh AR MCP Server tools

In **v1** of this MCP server, the available tools are:
- **Block** an ***IP Address*** on **all agents**
- **Block** an IP address on **selected agents** (by specifying **Agent IDs**)
- **Disble an Account** on **all agents**
- **Disble an Account** Disable an account on **selected agents**

>check out the code [here](https://github.com/B2hu/wazuh_ar_mcp/blob/main/server.py)
{: .prompt-info}

## Integration and Testing

Let’s configure our LLM in **agent mode** and integrate the **MCP server**.
For this demonstration, we’ll use Claude (Sonnet 4.5).
Head to **Settings → Developer**, and add the following configuration to the JSON file:
```json
{
  "mcpServers": {
    "wazuh-ar-mcp": {
      "type": "stdio",
      "command": "/home/ahmedbh/projects/wazuh_ar/venv/bin/python3",
      "args": [
        "/home/ahmedbh/projects/wazuh_ar/server.py"
      ]
    }
  }	
}
```
restart claude, the tools should now appear : 

![claude_tools](/media/post15/claude_tools.png)

Then, ask the LLM to list the available tools to verify input schema recognition:

![claude_test](/media/post15/claude_test.png)

Now, let’s test a few of them.

### Block an IP Address

We’ll start by fetching a few recently reported IPs from **AbuseIPDB** and blocking them across our agents.

- **On All Agents :**

![ip_block_all](/media/post15/ip_block_all.png)

As shown, the command executed successfully on agents **006 and 009**, but failed on **004 and 005** because they were inactive.

The underlying AR command used is **firewall-drop**, the default Wazuh Active Response script that utilizes **firewalld.**
A quick look at the agent’s iptables confirms this:
```shell
$ iptables -L 
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         
DROP       all  --  36.99.192.221        anywhere            

Chain FORWARD (policy DROP)
target     prot opt source               destination         
DROP       all  --  36.99.192.221        anywhere                      
```
The Wazuh **execd daemon** (responsible for executing scripts) created two drop rules — for both the **INPUT and FORWARD** chains.

- **On Selected Agents**

Now, let’s block another malicious IP on **agent 006** only.

![ip_block_selc](/media/post15/ip_block_selec.png)

Here, the LLM passed the IP address as **srcip** and the **agent ID** as an array to the MCP server, parameters that are then used to invoke the AR API.

```shell
$ iptables -L 
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         
DROP       all  --  236.149.216.162.bc.googleusercontent.com  anywhere            
DROP       all  --  36.99.192.221        anywhere            

Chain FORWARD (policy DROP)
target     prot opt source               destination         
DROP       all  --  236.149.216.162.bc.googleusercontent.com  anywhere            
DROP       all  --  36.99.192.221        anywhere            
```

### Disable An Account 

For this test, we’ll use Wazuh’s built-in **disable-account** script.
We’ll create a test user in a Linux environment where the Wazuh agent ID is **006**:

```shell
$ tail -1 /etc/passwd
test:x:1001:1001::/home/test:/bin/sh
```
Check the current account status:

```shell
$ passwd test --status
test P 2025-10-20 0 99999 7 -1
```
Here, **P** means the password is set (account active), and **L** means locked.

If the user exists on all agents, you can disable the account globally — but for now, we’ll target only agent 006.

Now, we instruct the LLM to disable user **test**

![disable_acc](/media/post15/disable_acc.png)

Just like the block-IP tool, the LLM passed the **dstuser** parameter.
Verifying the user’s status again shows:

```shell
$ passwd test --status
test L 2025-10-20 0 99999 7 -1
```
## Final Thoughts
In real-world **SOC operations**, ***threat hunting** often reveals malicious activity that slips past traditional endpoint sensors, activity that isn’t automatically contained or flagged by detection systems.
When that happens, incident response (IR) becomes critical.

This is where the **Wazuh Active Response MCP Server** truly shines.
By giving LLMs the ability to execute **IR actions** directly, based on analysts natural language, such as **blocking IPs**, **disabling compromised accounts**, or **isolating affected hosts**, analysts can react instantly without leaving their investigation workflow.

The result is a tight feedback loop between **detection, investigation, and response**, powered by automation and context awareness.

As we continue exploring MCP integrations, this approach marks a significant step toward building **intelligent, autonomous SOC environments**, where **AI agents** can not only understand incidents but also act decisively to contain them.