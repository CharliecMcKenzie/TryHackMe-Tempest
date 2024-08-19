# TryHackMe Tempest Challenge

## Introduction
This lab aims to introduce the process of analysing endpoint and network logs from a compromised asset. Given the artefacts, we will aim to uncover the incident from the Tempest machine. In this scenario, I will be tasked to be one of the Incident Responders that will focus on handling and analysing the captured artefacts of a compromised machine. 

## Walkthrough
### Initial Access - Malicious Document

Before starting the log analysis, I generated the SHA-256 hash for each file.
<p align="center">
<img src="https://i.imgur.com/X97URoT.png" height="80%" width="80%" alt="hash"/>

In this incident, I will act as an Incident Responder from an alert triaged by one of my Security Operations Center analysts. The analyst has confirmed that the alert has a CRITICAL severity that needs further investigation.
As reported by the SOC analyst, the intrusion started from a malicious document. In addition, the analyst compiled the essential information generated by the alert as listed below:
- The malicious document has a .doc extension.
- The user downloaded the malicious document via chrome.exe.
- The malicious document then executed a chain of commands to attain code execution.

To find the name of the malicious document, I first used EvtxEcmd to parse the sysmon.evtx file into CSV format for analysis in Timeline Explorer. I then searched for '.doc' and identified the malicious document as free_magicules.doc.
<p align="center">
<img src="https://i.imgur.com/BuJyJ3Z.png" height="80%" width="80%" alt="convertformat"/>
<img src="https://i.imgur.com/7ZjrvWh.png" height="80%" width="80%" alt=".doc"/>

Next I needed to know the name of the compromised user and machine.
<p align="center">
<img src="https://i.imgur.com/PW4E0C8.png" height="80%" width="80%" alt="username"/>

I also needed the PID of the Microsoft Word process that opened the malicious document, which I had already identified earlier when locating free_magicules.doc. 
<p align="center">
<img src="https://i.imgur.com/lvYJ2Fw.png" height="80%" width="80%" alt="PID"/>

From the Sysmon logs, I needed to identify the IPv4 address linked to the malicious domain. After reviewing the logs, I found a suspicious DNS query to phishteam.xyz, matching the previously identified PID. With the destination IP address 167.71.199.191. 
<p align="center">
<img src="https://i.imgur.com/vzK8z2J.png" height="80%" width="80%" alt="DNS"/>
<img src="https://i.imgur.com/mibkXio.png" height="80%" width="80%" alt="IP"/>

I identified the Base64-encoded string in the malicious payload executed by the document by searching for 'base64.' This search returned a log with a parent process ID of 496.
<p align="center">
<img src="https://i.imgur.com/ZlZYSPL.png" height="80%" width="80%" alt="Base64"/>

After identifying the payload executed by the document, I noticed that msdt.exe was invoked. A quick Google search revealed the CVE number of the exploit used by the attacker to achieve remote code execution. 
<p align="center">
<img src="https://i.imgur.com/ZoIKzrA.png" height="80%" width="80%" alt="CVE"/>

### Initial Access - Stage 2 execution

Based on the initial findings, we discovered that there is a stage 2 execution:

- The document has successfully executed an encoded base64 command.
- Decoding this string reveals the exact command chain executed by the malicious document.

Decoding the Base64 string using CyberChef revealed the full target path of the payload and the file  written to the system. 
<p align="center">
<img src="https://i.imgur.com/3LJDNw6.png" height="80%" width="80%" alt="cyberchef"/>

The implanted payload is executed when the user logs into the machine. The command triggered upon the compromised user's successful login was identified by filtering the logs for the domain phishteam.xyz, which was extracted from the decoded Base64 string. 
<p align="center">
<img src="https://i.imgur.com/ttggZFU.png" height="80%" width="80%" alt="command"/>

By filtering for phishteam.xyz, I also discovered a malicious binary along with its SHA-256 hash, which was downloaded for stage 2 execution. 
<p align="center">
<img src="https://i.imgur.com/uSbODtD.png" height="80%" width="80%" alt="binaryhash"/>

The stage 2 payload establishes a connection to a C2 server at the domain resolvecyber.xyz on port 80. 
<p align="center">
<img src="https://i.imgur.com/m0u6ibZ.png" height="80%" width="80%" alt="c2domain"/>
<img src="https://i.imgur.com/rDcwcoS.png" height="80%" width="80%" alt="c2port"/>

### Initial Access - Malicious Document Traffic 
Based on the collected findings, we discovered that the attacker fetched the stage 2 payload remotely:
- We discovered the Domain and IP invoked by the malicious document on Sysmon logs.
- There is another domain and IP used by the stage 2 payload logged from the same data source.

I used Wireshark to analyze the capture file, filtering for HTTP packets containing 'GET' requests and the domain 'phishteam.xyz' identified earlier. This led me to find the URL of the malicious payload embedded in the document.
<p align="center">
<img src="https://i.imgur.com/C32ds4S.png" height="80%" width="80%" alt="url"/>

The attacker used Base64 encoding for the C2 connection. The malicious C2 binary sends payloads via the 'q' parameter, which contains the results of executed commands. It connects to the specific URL '/9ab62b5' to retrieve commands for execution, using the 'GET' HTTP method. The user agent string revealed that the binary was compiled using the Nim programming language. 
<p align="center">
<img src="https://i.imgur.com/hhttitd.png" height="80%" width="80%" alt="basec2"/>

### Discovery - Internal Reconnaissance 
Based on the collected findings, we have discovered that the malicious binary continuously uses the C2 traffic:
- We can easily decode the encoded string in the network traffic.
- The traffic contains the command and output executed by the attacker.

The attacker managed to locate a sensitive file on the user's machine. By decoding several of the Base64-encoded commands, I eventually uncovered the password for the file.
<p align="center">
<img src="https://i.imgur.com/Mj9GO53.png" height="80%" width="80%" alt="basepassword"/>

The attacker proceeded to enumerate the list of listening ports on the machine. By further decoding the Base64-encoded commands, I identified the specific listening port that could be used to establish a remote shell on the machine. 
<p align="center">
<img src="https://i.imgur.com/xB164S8.png" height="80%" width="80%" alt="listeningport"/>

The attacker then set up a reverse SOCKS proxy to access internal services hosted on the machine. By searching for 'socks,' I located the command the attacker used to establish the connection. 
<p align="center">
<img src="https://i.imgur.com/tOzEM3m.png" height="80%" width="80%" alt="socks"/>

By using the SHA-256 hash of the binary used by the attacker to establish the reverse SOCKS proxy, I can search VirusTotal to identify the tool's name. 
<p align="center">
<img src="https://i.imgur.com/LgdNXU0.png" height="80%" width="80%" alt="socksvirustotal"/>

The attacker then utilized the harvested credentials from the machine. Based on the subsequent process following the execution of the SOCKS proxy, the attacker authenticated using the WinRM service.
<p align="center">
<img src="https://i.imgur.com/oX2PGqj.png" height="80%" width="80%" alt="winrm"/>

### Privilege Escalation - Exploiting Privileges 
Based on the collected findings, the attacker gained a stable shell through a reverse socks proxy.

After identifying the current user's privileges, the attacker downloaded another binary for privilege escalation. I discovered the binary ‘spf.exe’ and the SHA-256 hash by tracing the child process of wsmprovhost.exe. 
<p align="center">
<img src="https://i.imgur.com/22MHE4z.png" height="80%" width="80%" alt="spf.exe"/>

Using the SHA-256 hash of the binary, I identified the tool as PrintSpoofer.
<p align="center">
<img src="https://i.imgur.com/FQzDXqR.png" height="80%" width="80%" alt="printspoofer"/>

The tool exploits a specific user privilege known as SeImpersonatePrivilege. 
<p align="center">
<img src="https://i.imgur.com/CwoZITO.png" height="80%" width="80%" alt="SeImpersonatePrivilege"/>

The attacker then executed the tool alongside another binary to establish a C2 connection, which I had previously identified when discovering spf.exe. 
<p align="center">
<img src="https://i.imgur.com/t3udRWM.png" height="80%" width="80%" alt="final.exe"/>

The binary connects to port 8080, which is different from the port used in the initial C2 connection.  
<p align="center">
<img src="https://i.imgur.com/2AA3v2W.png" height="80%" width="80%" alt="port8080"/>

### Actions on Objective - Fully-owned Machine 
Now, the attacker has gained administrative privileges inside the machine. I need to find all persistence techniques used by the attacker.

In addition, the unusual executions are related to the malicious C2 binary used during privilege escalation.

After gaining SYSTEM access, the attacker created two new user accounts. I identified these accounts by filtering the Windows logs for Event ID 4720. 
<p align="center">
<img src="https://i.imgur.com/Gr1JuFh.png" height="80%" width="80%" alt="4720"/>

Prior to the successful creation of the accounts, the attacker executed commands that failed in the creation attempt due missing '/add' in the command. 
<p align="center">
<img src="https://i.imgur.com/nGs0p8s.png" height="80%" width="80%" alt="/add"/>

The attacker added one of the accounts to the local administrators group. This action is reflected in Event ID 4732, which signals the addition of a user to a sensitive local group. 
<p align="center">
<img src="https://i.imgur.com/olG0Fyu.png" height="80%" width="80%" alt="admingroup"/>

After creating the accounts, the attacker employed a technique to establish persistent administrative access. While there were two commands executed, only registry events from 'TempestUpdate2' were recorded. 
<p align="center">
<img src="https://i.imgur.com/4e8W3MC.png" height="80%" width="80%" alt="persistancecommand"/>
<img src="https://i.imgur.com/8YduyMf.png" height="80%" width="80%" alt="persistancecommand2"/>

## Summary
Drawing on my experience with Windows Event Logs, Sysmon, Wireshark, and Brim from previous exercises, I found this challenge to be a valuable opportunity to apply and reinforce what I've learned. 
