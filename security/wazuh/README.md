# Wazuh Security Homelab

## Overview

This project extends the homelab infrastructure into a centralized security monitoring and detection environment using Wazuh.

The lab was designed to demonstrate the complete security monitoring lifecycle:

1. Plan the security monitoring architecture
2. Deploy the Wazuh server
3. Harden the monitoring server
4. Enroll Linux and Windows endpoints
5. Collect authentication and system telemetry
6. Detect controlled security events
7. Investigate generated alerts
8. Integrate endpoint security telemetry
9. Implement and validate automated response
10. Document the resulting security architecture

The environment uses Wazuh Manager, Wazuh Indexer, Wazuh Dashboard, a Raspberry Pi Linux endpoint, and a Windows 11 endpoint.

---

## Lab Architecture

```text
                         Homelab LAN
                      192.168.200.0/24
                              |
             +----------------+----------------+
             |                                 |
             v                                 v
      +--------------+                  +---------------+
      |   Pi-hole    |                  | Windows 11    |
      | .178         |                  | .182          |
      | Debian 13    |                  | Bussy-Destoryer|
      | Wazuh Agent  |                  | Wazuh Agent   |
      +------+-------+                  +-------+-------+
             |                                  |
             +---------------+------------------+
                             |
                             v
                    +-------------------+
                    |   Wazuh Server    |
                    |   .180            |
                    |   Ubuntu 24.04    |
                    |   Wazuh 4.14.7    |
                    +---------+---------+
                              |
                 +------------+------------+
                 |                         |
                 v                         v
          Wazuh Manager              Wazuh Indexer
                 |                         |
                 +------------+------------+
                              |
                              v
                       Wazuh Dashboard
```

---

## Infrastructure

| Component | Address | Role |
|---|---|---|
| Aquila PRO M30 | 192.168.200.1 | Gateway / DHCP |
| Pi-hole | 192.168.200.178 | Linux endpoint / DNS |
| Wazuh Server | 192.168.200.180 | Manager / Indexer / Dashboard |
| Windows 11 | 192.168.200.182 | Windows endpoint |

### Wazuh Server

- OS: Ubuntu 24.04.5 LTS
- Wazuh: 4.14.7
- Hostname: `wazuh`
- IP: `192.168.200.180`

### Linux Agent

- Hostname: `pihole`
- IP: `192.168.200.178`
- OS: Debian 13
- Architecture: ARM64

### Windows Agent

- Hostname: `Bussy-Destoryer`
- IP: `192.168.200.182`
- OS: Windows 11 Pro
- Wazuh Agent: 4.14.7
- Agent ID: `002`

---

## Wazuh Server Hardening

The Wazuh server was hardened after deployment.

### SSH

- Root login disabled
- Password authentication disabled
- Public-key authentication enabled
- SSH restricted to the homelab LAN

### UFW

The Wazuh server uses UFW with:

- Default incoming: deny
- Default routed: deny
- Default outgoing: allow

Required Wazuh services are restricted to:

```text
192.168.200.0/24
```

including:

- TCP 22: SSH
- TCP 443: Wazuh Dashboard
- TCP 1514: Agent communication
- TCP 1515: Agent enrollment
- TCP 55000: Wazuh API

---

## Endpoint Monitoring

### Pi-hole

The Linux agent provides:

- SSH monitoring
- System monitoring
- File integrity monitoring
- Rootcheck
- System inventory
- Wazuh Active Response

### Windows

The Windows agent provides:

- Application event monitoring
- Security event monitoring
- System event monitoring
- Windows Defender Operational telemetry
- Windows Firewall Advanced Security telemetry
- File Integrity Monitoring
- Security Configuration Assessment
- System inventory
- Rootcheck

---

## Windows Authentication Detection

A controlled invalid-password test was performed against the Windows endpoint.

The test generated:

```text
Windows Event ID: 4625
Wazuh Rule: 60122
```

The event was successfully collected by the Wazuh agent and generated an alert on the manager.

The event was investigated and determined to be an intentional lab test rather than a real incident.

---

## Windows Defender Integration

The Windows Defender Operational event channel was added to the Wazuh agent:

```text
Microsoft-Windows-Windows Defender/Operational
```

The channel was verified as enabled and actively generating events.

Wazuh's native Windows Defender rules include events such as:

| Event ID | Wazuh Rule | Description | Level |
|---|---:|---|---:|
| 1150 | 62128 | Antimalware platform healthy | 3 |
| 1151 | 62129 | Endpoint Protection client health report | 2 |
| 2000 | 62130 | Antimalware definitions updated successfully | 3 |

The agent log confirmed the event channel was loaded successfully after restart.

---

## Windows Firewall Integration

Windows Firewall logging was initially enabled for all three Windows Firewall profiles:

- Domain
- Private
- Public

The native Windows Firewall Advanced Security event channel was then added to Wazuh:

```text
Microsoft-Windows-Windows Firewall With Advanced Security/Firewall
```

The Windows event log was verified as enabled and contained existing firewall telemetry.

Examples observed during validation included:

### Event 2082

Firewall configuration changes, including enabling dropped-packet logging.

### Event 2097

Windows Firewall rules being added.

### Event 2052

Windows Firewall rules being deleted.

The Wazuh agent confirmed successful collection of the native event channel:

```text
(1951): Analyzing event log:
'Microsoft-Windows-Windows Firewall With Advanced Security/Firewall'
```

This native event-channel approach was selected instead of relying on the legacy `pfirewall.log` text file.

---

## Detection and Investigation Workflow

The lab follows:

```text
Collect
   |
   v
Detect
   |
   v
Investigate
   |
   v
Respond
   |
   v
Validate
```

Examples completed so far include:

- Windows authentication failure detection
- Linux SSH invalid-user detection
- Windows Defender telemetry collection
- Windows Firewall telemetry collection
- Automated SSH containment

---

## SSH Detection and Automated Response

The Linux SSH detection workflow was initially tested using Wazuh's built-in rule 5710:

```text
Rule 5710
Level 5
sshd: Attempt to login using a non-existent user
```

The rule was first validated using `wazuh-logtest`, then verified with a controlled SSH authentication attempt.

The initial experiment used Wazuh's stock `firewall-drop` Active Response.

### Important operational finding

The stock `firewall-drop` response inserted a DROP rule that affected the Pi-hole's FORWARD traffic. Because the Pi-hole also hosts Docker services, this interfered with Docker networking and temporarily disrupted access to services.

This became an important security engineering lesson: automated containment must be designed around the actual network architecture rather than assuming that a generic firewall response is safe for every host.

---

## Custom SSH Containment

The stock response was replaced with a custom `ssh_contain` Active Response.

The custom response was designed to:

- Respond to SSH authentication alerts
- Process IPv4 addresses only
- Restrict containment to the homelab LAN
- Protect the management workstation
- Modify the Pi-hole INPUT chain rather than Docker-related FORWARD traffic
- Avoid duplicate rules
- Apply a temporary five-minute containment period
- Remove the exact containment rule when the timeout expires

The response was tested with controlled input and then validated against the real Wazuh Active Response workflow.

The management workstation was explicitly protected from containment during testing.

---

## Security Lessons Learned

### 1. Automated response can have unintended network effects

The initial `firewall-drop` test demonstrated that a technically successful firewall response can still cause an operational outage if it interacts with Docker networking or other forwarding paths.

### 2. Detection should be validated before response

Wazuh rules were tested with `wazuh-logtest` before live controlled events were generated.

### 3. Native Windows telemetry is valuable

Windows Defender and Windows Firewall expose structured event channels that Wazuh can consume directly.

### 4. Controlled testing matters

Security events were generated intentionally and investigated afterward. This allowed the detection pipeline to be validated without treating test activity as an actual incident.

### 5. Containment must account for management access

Testing automated firewall response against a remotely managed host can lock out the administrator. The custom response therefore includes safeguards for the management workstation.

---

## Related Documentation

Detailed SSH detection and Active Response documentation is maintained in:

```text
security/wazuh-ssh-detection/
```

This project will continue to be expanded as additional security monitoring and detection capabilities are added to the homelab.
