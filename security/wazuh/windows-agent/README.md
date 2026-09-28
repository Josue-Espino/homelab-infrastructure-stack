# Wazuh Windows Agent

## Overview

This document covers the deployment and validation of the Wazuh agent on the Windows 11 workstation used as the primary Windows endpoint in the security homelab.

The goal was not simply to install the agent, but to establish a useful Windows telemetry pipeline that could:

- Register the endpoint with Wazuh Manager
- Collect Windows Application, Security, and System events
- Detect controlled authentication failures
- Collect Windows Defender telemetry
- Collect Windows Firewall Advanced Security telemetry
- Perform file integrity monitoring and system inventory
- Validate that events reach the Wazuh Manager

---

## Endpoint

| Component | Value |
|---|---|
| Hostname | `Bussy-Destoryer` |
| OS | Windows 11 Pro |
| LAN IP | `192.168.200.182` |
| Wazuh Agent | 4.14.7 |
| Wazuh Manager | `192.168.200.180` |
| Agent ID | `002` |
| Install path | `C:\Program Files (x86)\ossec-agent` |

The physical LAN address was used for Wazuh communication. Other interfaces on the workstation, including VMware and Tailscale interfaces, were not used as the endpoint's Wazuh management address.

---

# 1. Wazuh Agent Installation

The Wazuh 4.14.7 MSI installer was downloaded to:

```text
C:\WazuhInstall\wazuh-agent-4.14.7-1.msi
```

The agent was installed with the Wazuh Manager address and endpoint name supplied during installation:

```text
WAZUH_MANAGER="192.168.200.180"
WAZUH_AGENT_NAME="Bussy-Destoryer"
```

After installation, the Windows service was started:

```powershell
Get-Service WazuhSvc
Start-Service WazuhSvc
```

The service was verified as running.

---

# 2. Agent Configuration

The Windows agent configuration points to the Wazuh Manager over TCP:

```xml
<server>
  <address>192.168.200.180</address>
  <port>1514</port>
  <protocol>tcp</protocol>
</server>

<agent_name>Bussy-Destoryer</agent_name>
```

The agent uses the Windows configuration profile:

```text
windows/windows10
```

The agent configuration was backed up before major telemetry changes.

---

# 3. Manager Connectivity

Connectivity from Windows to the Wazuh server was validated before relying on the agent connection.

The Wazuh communication ports tested were:

```text
TCP 1514  Agent communication
TCP 1515  Agent enrollment
```

Windows connectivity testing confirmed that both ports were reachable from the workstation.

The Wazuh agent subsequently established its connection to:

```text
192.168.200.180:1514/tcp
```

---

# 4. Agent Registration

The Wazuh Manager registered the Windows endpoint as Agent ID `002`.

Manager-side validation:

```text
ID: 000 wazuh (server) Active/Local
ID: 001 pihole Active
ID: 002 Bussy-Destoryer Active
```

The agent details confirmed:

- Windows 11 Pro
- Wazuh 4.14.7
- Agent ID 002
- Active status
- Successful Syscheck execution

This established the basic Windows monitoring pipeline.

---

# 5. Windows Event Monitoring

The initial Windows configuration collected three standard Windows event channels:

### Application

```text
Application
```

### Security

```text
Security
```

### System

```text
System
```

These channels provide the foundation for Windows application activity, authentication/security auditing, and operating-system events.

The Security channel was configured with filtering intended to reduce unnecessary event noise while retaining relevant security activity.

---

# 6. Windows Authentication Detection

A controlled authentication-failure test was used to verify the Security event pipeline.

The test intentionally attempted to authenticate with an invalid local account/password.

Windows generated:

```text
Event ID: 4625
Provider: Microsoft-Windows-Security-Auditing
```

The event was successfully collected by Wazuh.

Wazuh generated:

```text
Rule: 60122
Level: 5
Description: Logon Failure - Unknown user or bad password
```

The resulting event included:

```text
EventID: 4625
Target user: WazuhLabTest
Target domain: Bussy-Destoryer
Logon type: 2
Source address: ::1
```

The test was intentional and was investigated as a lab event rather than treated as a real security incident.

### Detection flow

```text
Controlled bad authentication
          |
          v
Windows Security Event 4625
          |
          v
Wazuh Windows Agent
          |
          v
Wazuh Manager
          |
          v
Rule 60122
          |
          v
Authentication failure alert
```

This validated that the Windows endpoint could produce a real security event and that Wazuh could decode and alert on it.

---

# 7. Windows Defender Telemetry

The Windows Defender Operational event channel was verified before integrating it with Wazuh.

The channel:

```text
Microsoft-Windows-Windows Defender/Operational
```

was confirmed to be enabled and actively generating events.

At validation time, the channel contained thousands of records and recent Defender events were being generated normally.

The channel was added to the Wazuh agent configuration using the Windows Event Channel collector:

```xml
<localfile>
  <location>Microsoft-Windows-Windows Defender/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

The agent configuration was backed up before this change.

After restarting the Wazuh service, the service remained running and reconnected successfully to the manager.

Wazuh's ruleset contains native handling for the Windows Defender Operational channel, confirming that the telemetry source is recognized by the platform.

> The purpose of this integration is to make Defender telemetry available to the centralized Wazuh monitoring pipeline. This document does not treat routine Defender informational events as security incidents.

---

# 8. Windows Firewall Telemetry

Windows Firewall logging was enabled for the Domain, Private, and Public firewall profiles.

The blocked-packet logging configuration was verified with:

```powershell
Get-NetFirewallProfile |
  Select-Object Name, Enabled, LogAllowed, LogBlocked, LogFileName
```

The resulting configuration enabled blocked-packet logging while leaving allowed-packet logging disabled.

The native Windows Firewall Advanced Security event channel was then selected as the Wazuh telemetry source:

```text
Microsoft-Windows-Windows Firewall With Advanced Security/Firewall
```

The channel was verified as enabled and already contained firewall telemetry.

---

# 9. Native Windows Firewall Event Channel

The Wazuh agent was configured to collect the native Windows Firewall Advanced Security event channel:

```xml
<localfile>
  <location>Microsoft-Windows-Windows Firewall With Advanced Security/Firewall</location>
  <log_format>eventchannel</log_format>
</localfile>
```

During validation, the event channel contained examples of normal firewall activity, including:

- Firewall configuration changes
- Firewall rule additions
- Firewall rule deletions

For example, Event 2082 was observed for a firewall configuration change, while Events 2097 and 2052 were observed for rule additions and deletions.

These examples demonstrate that the Windows endpoint is producing structured firewall telemetry that can be consumed centrally.

---

# 10. Why Native Event Channels Were Used

The initial firewall integration attempted to have Wazuh monitor:

```text
C:\Windows\System32\LogFiles\Firewall\pfirewall.log
```

The agent successfully opened the file, but the native Windows Firewall Advanced Security event channel provided a more structured Windows-native telemetry source.

The final configuration therefore uses:

```text
Microsoft-Windows-Windows Firewall With Advanced Security/Firewall
```

rather than relying on Wazuh to tail the legacy text log.

The `pfirewall.log` file remains a Windows firewall logging artifact, but it is not the final Wazuh collection source documented here.

---

# 11. File Integrity Monitoring

The Windows agent's Syscheck module is enabled.

The configuration monitors Windows system and security-relevant locations, including areas associated with:

- Windows system files
- Startup configuration
- Registry data
- PowerShell
- Windows services

The agent's Syscheck process was validated from the manager.

The manager reported successful Syscheck start and completion for Agent 002.

---

# 12. Security Configuration Assessment

Security Configuration Assessment (SCA) is enabled on the Windows agent.

The configuration includes:

```xml
<sca>
  <enabled>yes</enabled>
  <scan_on_start>yes</scan_on_start>
  <interval>12h</interval>
</sca>
```

This allows Wazuh to periodically evaluate the endpoint against its configured security assessment content.

---

# 13. System Inventory

The Windows agent's System Collector is enabled with a one-hour collection interval.

The collected inventory categories include:

- Hardware
- Operating system
- Network configuration
- Installed packages
- Listening ports
- Processes
- Users
- Groups
- Services
- Browser extensions

This gives the Wazuh manager an inventory view of the monitored Windows endpoint in addition to event-based security telemetry.

---

# 14. Active Response

Active Response is enabled in the Windows agent configuration.

The Windows endpoint was primarily used as a monitored endpoint during this phase of the lab.

The automated firewall containment workflow documented separately for the Pi-hole was intentionally not reused as a Windows response without designing a Windows-specific response policy.

Detailed Linux SSH containment documentation:

```text
security/wazuh-ssh-detection/
```

---

# 15. Validation

The Windows agent was validated at multiple layers.

### Service

```text
WazuhSvc: Running
```

### Manager registration

```text
Agent 002: Active
```

### Network connectivity

```text
Windows -> 192.168.200.180:1514 TCP
Windows -> 192.168.200.180:1515 TCP
```

### Security event

```text
Windows Event 4625
        |
        v
Wazuh Rule 60122
        |
        v
Alert generated
```

### Defender telemetry

```text
Windows Defender Operational
        |
        v
Wazuh Event Channel Collector
        |
        v
Wazuh Manager
```

### Firewall telemetry

```text
Windows Firewall Advanced Security
        |
        v
Wazuh Event Channel Collector
        |
        v
Wazuh Manager
```

---

# 16. Troubleshooting and Investigation Notes

## Windows event collection was working

Real Windows Security events reached Wazuh and produced alerts.

The Event 4625 test provided a controlled end-to-end validation without requiring an actual malicious event.

## Not every Windows event needs to become an alert

The Windows endpoint generates a large amount of normal operating-system, Defender, and firewall telemetry.

The purpose of centralized collection is to make this data available for detection and investigation. Alert tuning should be based on observed behavior rather than treating every informational event as suspicious.

## A custom test event did not appear as an alert

A custom Windows Application event using Event ID 4242 was created during validation.

Windows successfully recorded the event, but it did not appear as a Wazuh alert.

The Wazuh configuration had:

```text
logall=no
logall_json=no
```

and no raw event archive was being used for this test.

The event therefore was not used as evidence that the Windows event pipeline was broken. The real Security Event 4625 test provided the successful end-to-end detection evidence.

## Dashboard performance

During investigation, the Wazuh Dashboard browser session temporarily froze all open browser tabs.

The Wazuh services themselves remained operational.

Subsequent validation therefore relied more heavily on CLI and endpoint-side checks instead of repeatedly running heavy dashboard searches.

---

# 17. Configuration Backups

Backups were created before major Windows agent telemetry changes.

Examples include:

```text
C:\Program Files (x86)\ossec-agent\ossec.conf.backup-before-defender
C:\Program Files (x86)\ossec-agent\ossec.conf.backup-before-firewall-log
C:\Program Files (x86)\ossec-agent\ossec.conf.backup-before-native-firewall
```

These backups provided rollback points while the telemetry sources were being developed and validated.

---

# 18. Completed Milestone

## Wazuh Windows Endpoint Monitoring

### Deployment

- [x] Wazuh Windows agent installed
- [x] Agent configured for Wazuh Manager
- [x] Windows service started
- [x] Agent registered as ID 002
- [x] Manager connectivity validated

### Event Monitoring

- [x] Application event channel collected
- [x] Security event channel collected
- [x] System event channel collected
- [x] Controlled authentication failure generated
- [x] Windows Event 4625 verified
- [x] Wazuh Rule 60122 verified

### Endpoint Security Telemetry

- [x] Windows Defender Operational channel verified
- [x] Defender channel added to Wazuh
- [x] Windows Firewall logging configured
- [x] Native Windows Firewall event channel verified
- [x] Native firewall channel added to Wazuh

### Endpoint Monitoring

- [x] File Integrity Monitoring enabled
- [x] Security Configuration Assessment enabled
- [x] System inventory enabled
- [x] Active Response framework enabled

---

# 19. Final Architecture

```text
                         Windows 11 Pro
                       Bussy-Destoryer
                       192.168.200.182
                              |
              +---------------+---------------+
              |               |               |
              v               v               v
        Security Logs     Defender Logs   Firewall Logs
              |               |               |
              +---------------+---------------+
                              |
                              v
                       Wazuh Agent 002
                              |
                              | TCP 1514
                              v
                     Wazuh Manager
                     192.168.200.180
                              |
              +---------------+---------------+
              |               |               |
              v               v               v
          Detection       Inventory         SCA
          & Alerts        & Syscheck       Results
```

---

## Final Takeaway

The Windows endpoint became the second major monitored endpoint in the security lab.

The important milestone was not simply installing the Wazuh agent. The endpoint was validated as a real telemetry source by generating a controlled Windows authentication failure, observing Event 4625, and confirming Wazuh Rule 60122.

The monitoring scope was then expanded to include Windows Defender and native Windows Firewall Advanced Security event channels, while maintaining file integrity monitoring, security assessment, and system inventory.

This creates a foundation for the next stage of the security lab: using multiple endpoint data sources together for detection, investigation, and correlation.

> **Lab status: Windows endpoint deployment and centralized security telemetry successfully demonstrated.**
