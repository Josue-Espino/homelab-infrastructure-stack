# Wazuh Linux Agent

## Overview

This document covers the deployment and validation of the Wazuh agent on the Pi-hole host, the first Linux endpoint added to the Wazuh security lab.

The Pi-hole is an important monitoring target because it provides DNS services and also hosts Docker-based homelab services. Monitoring it therefore provides visibility into both Linux host activity and a system that participates in other infrastructure services.

---

## Endpoint

| Component | Value |
|---|---|
| Hostname | `pihole` |
| IP | `192.168.200.178` |
| OS | Debian 13 |
| Architecture | ARM64 / AArch64 |
| Role | Pi-hole DNS / Linux endpoint |
| Wazuh Agent ID | `001` |
| Wazuh Manager | `192.168.200.180` |

The Pi-hole is a physical Raspberry Pi rather than a Proxmox container.

---

# 1. Agent Deployment

The Pi-hole was enrolled as the first remote Wazuh endpoint.

The Wazuh Manager generated the agent registration information and the Pi-hole was configured to communicate with:

```text
Wazuh Manager
192.168.200.180
```

After configuration, the Wazuh agent service was started and verified.

---

# 2. Agent Registration

The Wazuh Manager confirmed the Pi-hole as Agent ID `001`.

Manager-side validation:

```text
ID: 000, Name: wazuh (server), IP: 127.0.0.1, Active/Local
ID: 001, Name: pihole, IP: any, Active
```

This established the first remote endpoint-to-manager connection in the security lab.

---

# 3. Linux Host Monitoring

The Linux agent provides centralized visibility into the Pi-hole host.

The monitoring scope includes:

- SSH activity
- File integrity monitoring
- Rootcheck
- Security Configuration Assessment
- System inventory
- Process and service information
- Network information
- Active Response

The Pi-hole therefore acts as both a production homelab service host and a monitored security endpoint.

---

# 4. SSH Monitoring

SSH monitoring became the first detection workflow built on the Linux agent.

The Pi-hole's SSH authentication activity is collected by Wazuh and analyzed by the manager.

The primary detection used during the lab was Wazuh Rule 5710:

```text
Rule: 5710
Level: 5
Description:
sshd: Attempt to login using a non-existent user
```

The rule detects SSH authentication attempts involving invalid or non-existent users.

Before performing a live test, the detection logic was validated with `wazuh-logtest`.

---

# 5. SSH Detection Workflow

The resulting workflow was:

```text
SSH authentication attempt
          |
          v
Pi-hole sshd
          |
          v
Wazuh Agent 001
          |
          v
Wazuh Manager
          |
          v
Rule 5710
          |
          v
Alert
```

This established the first complete detection pipeline in the security lab.

The detailed SSH detection and automated containment implementation is documented separately:

```text
security/wazuh/ssh-detection/
```

---

# 6. Active Response

Active Response was initially disabled on the Pi-hole.

The original configuration contained:

```xml
<active-response>
    <disabled>yes</disabled>
    <ca_store>etc/wpk_root.pem</ca_store>
    <ca_verification>yes</ca_verification>
</active-response>
```

It was changed to:

```xml
<active-response>
    <disabled>no</disabled>
    <ca_store>etc/wpk_root.pem</ca_store>
    <ca_verification>yes</ca_verification>
</active-response>
```

After restarting the agent, the Active Response execution framework became available to the endpoint.

This was an important prerequisite for the SSH containment project.

---

# 7. File Integrity Monitoring

The Wazuh Syscheck module is enabled on the Pi-hole.

File Integrity Monitoring provides visibility into changes to monitored files and directories.

The agent's Syscheck activity was validated during the lab as part of the endpoint's normal Wazuh operation.

The Pi-hole therefore provides both event-based monitoring and host-state monitoring.

---

# 8. Rootcheck

Rootcheck is enabled on the Linux agent.

Rootcheck provides an additional host-security assessment layer that can identify configured checks associated with suspicious system conditions.

It operates alongside Syscheck and event collection rather than replacing them.

---

# 9. Security Configuration Assessment

Security Configuration Assessment is enabled on the Pi-hole agent.

The configuration includes:

```xml
<sca>
  <enabled>yes</enabled>
  <scan_on_start>yes</scan_on_start>
  <interval>12h</interval>
</sca>
```

This allows the endpoint to periodically evaluate its security configuration against the applicable Wazuh assessment content.

---

# 10. System Inventory

The Wazuh System Collector provides inventory information from the Pi-hole.

The enabled collection categories include information such as:

- Hardware
- Operating system
- Network configuration
- Installed packages
- Listening ports
- Processes
- Users
- Groups
- Services

The inventory adds context to security events by providing information about the monitored endpoint itself.

---

# 11. Pi-hole and Docker Considerations

The Pi-hole is not an isolated Linux server.

It also participates in the homelab's service infrastructure, including Docker networking and services.

This became especially important during Active Response testing.

The initial use of Wazuh's generic `firewall-drop` response inserted a source-IP DROP rule affecting the Pi-hole FORWARD chain.

Because Docker traffic uses forwarding paths, the response temporarily disrupted access to services.

This observation directly influenced the final Active Response design.

The custom `ssh_contain` response was therefore designed to target SSH on the Pi-hole INPUT chain rather than broadly blocking the source IP through forwarding traffic.

Detailed implementation:

```text
security/wazuh/ssh-detection/
```

---

# 12. Firewall Architecture

The Pi-hole uses:

```text
iptables
iptables-nft
nf_tables backend
```

The firewall was inspected during the Active Response investigation.

The normal Docker forwarding structure included:

```text
DOCKER-USER
DOCKER-FORWARD
wg0 forwarding rules
```

The custom SSH containment design deliberately avoids modifying the Docker forwarding path.

---

# 13. Active Response Safety

The custom SSH containment response was designed with the Pi-hole's infrastructure role in mind.

The response:

- Validates the source IP
- Rejects IPv6
- Restricts eligible sources to the homelab LAN
- Protects the management workstation
- Targets TCP/22
- Uses the Pi-hole INPUT chain
- Avoids Docker FORWARD traffic
- Prevents duplicate containment rules
- Uses Wazuh's add/delete lifecycle
- Applies a five-minute timeout

The final validation confirmed that the management workstation was recognized as trusted and was not blocked.

---

# 14. Validation

The Pi-hole agent was validated at multiple levels.

### Registration

```text
Agent ID 001: Active
```

### Host

```text
Debian 13
ARM64 / AArch64
192.168.200.178
```

### Detection

```text
SSH invalid-user event
        |
        v
Wazuh Rule 5710
        |
        v
Alert
```

### Active Response

```text
Rule 5710
        |
        v
ssh_contain
        |
        v
Source validation
        |
        v
SSH-only containment
```

### Safety

```text
Trusted management source
        |
        v
No firewall block
```

---

# 15. Troubleshooting Lessons

## The Pi-hole is both a security endpoint and infrastructure host

This was the most important architectural consideration.

A firewall response that is safe for a simple SSH server may not be safe for a host running Docker, VPN forwarding, DNS, and other services.

## Active Response must be enabled on the endpoint

The initial Pi-hole configuration had Active Response disabled.

This had to be changed before automated response could be validated.

## Detection should be tested before containment

Rule 5710 was validated with `wazuh-logtest` before the live SSH test.

This separated detection validation from response validation.

## Remote firewall testing can cause lockout

The initial response test demonstrated that automated firewall changes can interrupt the administrator's connection.

The final response therefore includes a trusted management source safeguard.

---

# 16. Completed Milestone

## Wazuh Linux Endpoint Monitoring

### Deployment

- [x] Pi-hole enrolled as Wazuh Agent 001
- [x] Agent connected to Wazuh Manager
- [x] Linux endpoint monitoring established

### Monitoring

- [x] SSH monitoring
- [x] File Integrity Monitoring
- [x] Rootcheck
- [x] Security Configuration Assessment
- [x] System inventory
- [x] Process/service visibility
- [x] Network visibility

### Detection

- [x] SSH invalid-user event validated
- [x] Rule 5710 validated
- [x] Real controlled SSH event generated
- [x] Wazuh alert confirmed

### Response

- [x] Active Response enabled
- [x] Built-in firewall response investigated
- [x] Docker forwarding side effect identified
- [x] Custom SSH containment designed
- [x] Management source protection validated

---

# 17. Final Architecture

```text
                         Pi-hole
                    192.168.200.178
                         Debian 13
                         ARM64
                            |
             +--------------+--------------+
             |              |              |
             v              v              v
          SSH logs       Syscheck       Inventory
             |              |              |
             +--------------+--------------+
                            |
                            v
                     Wazuh Agent 001
                            |
                            | TCP 1514
                            v
                    Wazuh Manager
                    192.168.200.180
                            |
             +--------------+--------------+
             |              |              |
             v              v              v
         Detection       SCA/Rootcheck   Alerts
             |
             v
      Custom Active Response
          ssh_contain
             |
             v
      Pi-hole INPUT TCP/22
```

---

## Final Takeaway

The Pi-hole was the first remote Linux endpoint in the Wazuh security lab and became the foundation for the lab's first detection and automated-response workflow.

The endpoint demonstrates an important security engineering principle: monitoring and response must be designed around the role of the system being protected.

Because the Pi-hole also provides infrastructure services and Docker networking, the final SSH containment design avoids broad source-IP blocking through the forwarding path and instead limits containment to the SSH service itself.

> **Lab status: Linux endpoint monitoring, SSH detection, and safety-focused automated containment successfully demonstrated.**
