# Attack Artifacts in Logs — Analysis and Threat Detection in SIEM Systems

---

## 📌 Project Overview

This project documents the design and implementation of a small Security Operations Center lab using Wazuh, Ubuntu Server, Windows 11, Sysmon, VMware Workstation, and Parrot OS. and on **detecting threats using SIEM-class systems**.  
For this purpose, a complete **laboratory environment simulating a SOC** was built, allowing to simulate real cyberattacks, capture their traces, and analyze detection methods.

Main project goals:
- Security monitoring
- Endpoint log collection
- Windows telemetry analysis
- Detection engineering
- File integrity monitoring
- Custom Wazuh rule creation
- Threat hunting
- Alert investigation
- Basic attack simulation in an isolated environment
- Understand how attacks **leave detectable artifacts in logs**
- Map attacker activity to the **MITRE ATT&CK** model

  
The final environment allows activity on a Windows endpoint to be collected by Sysmon and the Wazuh agent, processed by the Wazuh manager, indexed, and displayed in the Wazuh dashboard.
---
## Project Status

### Completed

- Ubuntu Server virtual machine created
- Wazuh all-in-one deployment installed
- Wazuh manager configured
- Wazuh indexer configured
- Wazuh dashboard configured
- Windows 11 virtual machine created
- VMware Tools installed
- Wazuh Windows agent deployed
- Windows agent connected successfully
- Sysmon installed on Windows
- Sysmon event channel added to the Wazuh agent configuration
- Sysmon alerts confirmed in Wazuh
- Custom Notepad process detection rule created
- File integrity monitoring configured
- Real-time directory monitoring enabled

### Planned Improvements

- RDP failed-login detection
- Brute-force correlation rule
- Active response testing
- Additional Sysmon detections
- Email or webhook alerting
- Vulnerability detection
- Security configuration assessment
- Dashboard customization

---

## Lab Architecture

```
                         VMware Workstation

       ┌──────────────────────────────────────────────┐
       │                                              │
       │    ┌───────────────────────────────────┐     │
       │    │ Ubuntu Server                    │     │
       │    │                                   │     │
       │    │ Wazuh Manager                    │     │
       │    │ Wazuh Indexer                    │     │
       │    │ Wazuh Dashboard                  │     │
       │    └───────────────┬───────────────────┘     │
       │                    │                         │
       │          VMware NAT Network                  │
       │                    │                         │
       │    ┌───────────────┴───────────────────┐     │
       │    │ Windows 11                       │     │
       │    │                                   │     │
       │    │ Wazuh Agent                      │     │
       │    │ Sysmon                           │     │
       │    │ File Integrity Monitoring        │     │
       │    └───────────────────────────────────┘     │
       │                                              │
       │    ┌───────────────────────────────────┐     │
       │    │ Parrot OS                        │     │
       │    │                                   │     │
       │    │ Controlled testing system        │     │
       │    └───────────────────────────────────┘     │
       │                                              │
       └──────────────────────────────────────────────┘
```

---
# 1. Installing Ubuntu Server

Ubuntu Server AMD64 was installed on the Wazuh virtual machine.

During installation:

- OpenSSH Server was enabled
- A local administrative user was created
- The VM remained connected to VMware NAT

The Ubuntu IP address was identified with:

```bash
hostname -I
```

The server was then managed from Windows PowerShell through SSH:

---
# 2. Installing the Wazuh Server

The Wazuh all-in-one deployment method was used. It installed:

- Wazuh Manager
- Wazuh Indexer
- Wazuh Dashboard
- Filebeat
- Certificates
- Internal Wazuh users

The installation was performed from the Ubuntu terminal.
## Screenshot
---

# 3. Accessing the Wazuh Dashboard

The Wazuh dashboard was accessed from my Host Windows laptop:

## Screenshot
> Wazuh dashboard successfully accessed from the Windows host machine.

---

# 4. Installing Windows 11

A Windows 11 VM was created in VMware Workstation using the official ISO file from Microsofts website.

---

# 5. Deploying the Wazuh Windows Agent

In the Wazuh dashboard:

```text
Agents Management → Summary → Deploy New Agent
```
## Screenshot
The generated PowerShell installation command was copied and executed in an elevated PowerShell window on the Windows VM.
The Wazuh service was started:

```powershell
Start-Service WazuhSvc
```

Its status was verified:

```powershell
Get-Service WazuhSvc
```

Expected result:

```text
Status: Running
```

The Windows agent appeared as `Active` in the Wazuh dashboard.
## Screenshot
> Windows endpoint successfully enrolled and reporting as an active Wazuh agent.

# 6. Installing Sysmon

A working directory was created:

```powershell
New-Item -ItemType Directory -Path C:\SOC\Sysmon -Force
Set-Location C:\SOC\Sysmon
```

Sysmon was downloaded and extracted:

```powershell
Invoke-WebRequest `
  -Uri "https://download.sysinternals.com/files/Sysmon.zip" `
  -OutFile "C:\SOC\Sysmon\Sysmon.zip"

Expand-Archive `
  -Path "C:\SOC\Sysmon\Sysmon.zip" `
  -DestinationPath "C:\SOC\Sysmon" `
  -Force
```

A community Sysmon configuration was downloaded and used as a starting point (https://github.com/olafhartong/sysmon-modular).

Sysmon was installed:

```powershell
C:\SOC\Sysmon\Sysmon64.exe `
  -accepteula `
  -i C:\SOC\Sysmon\sysmonconfig.xml
```
---

# 7. Verifying Sysmon Events Locally

Sysmon events were reviewed in Windows Event Viewer:

```text
Applications and Services Logs
└── Microsoft
    └── Windows
        └── Sysmon
            └── Operational
```
# 8. Configuring Wazuh to Collect Sysmon

The Wazuh agent configuration was located at:

```text
C:\Program Files (x86)\ossec-agent\ossec.conf
```

Because the file is located under `Program Files (x86)`, it had to be edited from an elevated application.

PowerShell was opened as Administrator:

```powershell
notepad "C:\Program Files (x86)\ossec-agent\ossec.conf"
```

The following block was added before the final `</ossec_config>` tag:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

The Wazuh service was restarted:

```powershell
Restart-Service WazuhSvc
```

## Screenshot
> Wazuh Threat Hunting dashboard displaying Sysmon alerts from the Windows endpoint.


> Sysmon operational log showing endpoint telemetry generated by the Windows VM.

# 9. Configuring File Integrity Monitoring

A directory was created on Windows:

```powershell
New-Item `
  -ItemType Directory `
  -Path "C:\SOC\integrity-check" `
  -Force
```

The Wazuh agent configuration was reopened as Administrator:

```powershell
notepad "C:\Program Files (x86)\ossec-agent\ossec.conf"
```

Inside the existing `<syscheck>` section, the following entry was added:

```xml
<directories realtime="yes"
             check_all="yes"
             report_changes="yes">C:\SOC\integrity-check</directories>
```
# 10. Testing File Integrity Monitoring

A file was created:

```powershell
"First version" |
Set-Content "C:\SOC\integrity-check\test.txt"
```

The file was then modified:

```powershell
"Second version" |
Add-Content "C:\SOC\integrity-check\test.txt"
```
And a file was deleted, called "My business model"

The resulting events demonstrate that Wazuh can detect:

- File creation
- File modification
- File deletion
- Attribute changes
- Content changes for supported text files

## Screenshot
> Wazuh alert showing a file change inside the monitored integrity-check directory.

# 11. Skills Demonstrated

This project demonstrates experience with:

- VMware Workstation
- Linux server administration
- Windows administration
- SSH
- PowerShell
- Security monitoring
- Wazuh deployment
- Endpoint agent deployment
- Sysmon configuration
- Windows Event Logs
- XML configuration
- File integrity monitoring
- Threat hunting
- Alert analysis
- Troubleshooting

# 12. Future Improvements

Planned extensions include:

1. Detect repeated Windows authentication failures.
2. Detect suspicious PowerShell commands.
3. Detect encoded PowerShell execution.
4. Detect new local administrator accounts.
5. Detect changes to sensitive Windows directories.
6. Map detections to MITRE ATT&CK.
7. Test Wazuh Active Response.
8. Create custom Wazuh dashboard visualizations.
9. Integrate TheHive, Shuffle, or MISP.

---



## 📌 Keywords

`siem` · `cybersecurity` · `elastic-stack` · `suricata` · `zeek` · `sysmon` · `Windows-event-log` · `threat-detection` · `mitre-attack` · `soc` · `log-analysis` · `blue-team` · `filebeat` · `winlogbeat` · `kibana` · `elasticsearch` · `kali-linux` · `network-security` · `endpoint-security` · `pfsense` · `mythic` · `c2-framework` · `log-monitoring` · `event-correlation` · `homelab` · `vmware` · `pyramid-of-pain` 
