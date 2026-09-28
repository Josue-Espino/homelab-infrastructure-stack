# Wazuh Security Homelab: SSH Detection & Automated Firewall Response

## Overview

This project extends the homelab into a security monitoring and automated response environment using Wazuh.

The first security workflow implemented was:

> Detect an invalid SSH login attempt on the Pi-hole → generate a Wazuh alert → automatically block the attacking IP with `iptables` → remove the block after a timeout.

This was tested with a controlled SSH attack from the Wazuh server against the Pi-hole.

---

## Lab Architecture

```
                    ┌─────────────────────────────┐
                    │       Wazuh Server          │
                    │                             │
                    │  IP: 192.168.200.180        │
                    │  Wazuh Manager              │
                    └──────────────┬──────────────┘
                                   │
                         Wazuh agent connection
                                   │
                                   ▼
                    ┌─────────────────────────────┐
                    │          Pi-hole             │
                    │                             │
                    │  IP: 192.168.200.178        │
                    │  Raspberry Pi                │
                    │  Debian 13                   │
                    │  Wazuh Agent                 │
                    │  iptables                    │
                    └─────────────────────────────┘
```

### Components

- **Wazuh Manager:** `wazuh`
- **Wazuh Agent:** `pihole`
- **Pi-hole IP:** `192.168.200.178`
- **Wazuh server IP:** `192.168.200.180`
- **Historical Windows test source IP:** `192.168.200.245`
- **Current Windows test source IP:** `192.168.200.182`
- **Wazuh version:** `4.14.7`
- **Pi-hole architecture:** ARM64 / AArch64
- **Firewall:** `iptables` using the `nf_tables` backend

---

# 1. Initial Active Response Investigation

The initial goal was to make Wazuh's `firewall-drop` active response work on the Pi-hole.

We first verified the firewall binaries:

```bash
sudo ls -l /usr/sbin/iptables-nft
sudo ls -l /usr/sbin/xtables-nft-multi
sudo ls -l /usr/sbin/iptables-legacy
```

The system showed:

```text
/usr/sbin/iptables-nft -> xtables-nft-multi
/usr/sbin/xtables-nft-multi
/usr/sbin/iptables-legacy -> xtables-legacy-multi
```

We also verified the architecture:

```bash
sudo file /usr/sbin/iptables-nft
sudo file /usr/sbin/xtables-nft-multi
```

The nft implementation was confirmed to be an:

```text
ELF 64-bit LSB pie executable, ARM aarch64
```

This established that the Pi-hole had a valid ARM64 `iptables-nft` implementation.

---

# 2. Verifying Wazuh Active Response Permissions

We checked the Wazuh execution daemon:

```bash
sudo ps -o pid,user,euser,group,egroup,comm,args -C wazuh-execd
```

The process was running as root:

```text
USER     EUSER
root     root
```

Its effective capabilities were also present:

```text
CapPrm: 000001ffffffffff
CapEff: 000001ffffffffff
CapBnd: 000001ffffffffff
```

We checked the Wazuh directory permissions:

```bash
sudo ls -ld /var/ossec
sudo ls -ld /var/ossec/active-response
sudo ls -ld /var/ossec/active-response/bin
```

The directories were owned by `root:wazuh` with mode `750`.

The active response executable was also verified:

```bash
sudo ls -l /var/ossec/active-response/bin/firewall-drop
```

Result:

```text
-rwxr-x--- 1 root wazuh ... firewall-drop
```

---

# 3. Testing `firewall-drop` Manually

We initially tested whether the `wazuh` user could directly execute `iptables`.

```bash
sudo -u wazuh /usr/sbin/iptables -L INPUT -n
```

This returned:

```text
iptables v1.8.11 (nf_tables): Could not fetch rule set generation id:
Permission denied (you must be root)
```

The same happened when attempting to insert a rule as the `wazuh` user.

However, root could use `iptables` successfully:

```bash
sudo iptables -L INPUT -n -v --line-numbers
```

At this point there were no firewall rules in the INPUT chain.

---

# 4. Investigating `firewall-drop`

We executed the Wazuh `firewall-drop` active response manually using a JSON request.

The response initially returned exit status `255`.

A `strace` investigation was then performed:

```bash
sudo strace -f -s 512 -o /tmp/firewall-add.strace \
/var/ossec/active-response/bin/firewall-drop
```

The trace showed that the executable successfully:

- Started
- Loaded its Wazuh libraries
- Changed directory to `/var/ossec`
- Opened `active-responses.log`
- Read the supplied JSON
- Parsed the request
- Returned a `check_keys` response

The important observation was that the executable itself was functioning and processing the active-response protocol.

The logs contained:

```text
"command":"add"
```

followed by:

```text
"command":"check_keys"
```

and eventually:

```text
"command":"continue"
```

The earlier manual invocation was not an appropriate simulation of the full Wazuh active-response execution flow, which became important later.

---

# 5. Discovering the Active Response Configuration Issue

We inspected the Pi-hole Wazuh configuration:

```bash
sudo grep -n -A20 -B5 "active-response" /var/ossec/etc/ossec.conf
```

The Pi-hole contained:

```xml
<active-response>
    <disabled>yes</disabled>
    <ca_store>etc/wpk_root.pem</ca_store>
    <ca_verification>yes</ca_verification>
</active-response>
```

This meant active response was disabled on the Pi-hole agent.

A backup of the configuration was confirmed:

```bash
sudo ls -lh /var/ossec/etc/ossec.conf.backup
```

The backup existed before making the change.

---

# 6. Enabling Active Response on Pi-hole

The setting was changed from:

```xml
<disabled>yes</disabled>
```

to:

```xml
<disabled>no</disabled>
```

using:

```bash
sudo sed -i 's/<disabled>yes<\\/disabled>/<disabled>no<\\/disabled>/' \
/var/ossec/etc/ossec.conf
```

We verified the result:

```text
<active-response>
    <disabled>no</disabled>
    <ca_store>etc/wpk_root.pem</ca_store>
    <ca_verification>yes</ca_verification>
</active-response>
```

The Wazuh agent was restarted:

```bash
sudo systemctl restart wazuh-agent
```

and verified as running:

```bash
sudo systemctl status wazuh-agent --no-pager
```

The agent started successfully and launched:

- `wazuh-execd`
- `wazuh-agentd`
- `wazuh-syscheckd`
- `wazuh-logcollector`
- `wazuh-modulesd`

---

# 7. Verifying Wazuh Agent Registration

On the Wazuh server:

```bash
sudo /var/ossec/bin/agent_control -l
```

showed:

```text
ID: 000, Name: wazuh (server), IP: 127.0.0.1, Active/Local
ID: 001, Name: pihole, IP: any, Active
```

The manager log also showed the Pi-hole registration:

```text
wazuh-authd: INFO: Received request for a new agent (pihole)
wazuh-authd: INFO: Agent key generated for 'pihole'
```

This confirmed that the Pi-hole agent was registered and active.

---

# 8. Confirming Rule 5710

The built-in Wazuh rule was located with:

```bash
sudo grep -R -n -A15 -B5 '<rule id="5710"' \
/var/ossec/ruleset/rules /var/ossec/etc 2>/dev/null
```

Rule 5710 is defined in:

```text
/var/ossec/ruleset/rules/0095-sshd_rules.xml
```

The rule is:

```xml
<rule id="5710" level="5">
    <if_sid>5700</if_sid>
    <match>illegal user|invalid user</match>
    <description>sshd: Attempt to login using a non-existent user</description>
</rule>
```

The rule also maps to:

- `T1110.001` - Password Guessing
- `T1021.004` - SSH

---

# 9. Testing the Rule with `wazuh-logtest`

On the Wazuh server:

```bash
sudo /var/ossec/bin/wazuh-logtest
```

We supplied this test event:

```text
Sep 11 17:40:00 pihole sshd[12345]: Invalid user wazuhtest from 192.168.200.245 port 54321
```

Wazuh decoded:

```text
name: 'sshd'
srcip: '192.168.200.245'
srcport: '54321'
srcuser: 'wazuhtest'
```

Phase 3 matched:

```text
id: '5710'
level: '5'
description: 'sshd: Attempt to login using a non-existent user'
```

The result was:

```text
**Alert to be generated.
```

This proved that the detection rule worked before performing a live test.

---

# 10. Initial Automated Firewall Response

The first response design used Wazuh's built-in `firewall-drop` command.

The initial configuration connected Rule 5710 to the response with a five-minute timeout.

A controlled SSH authentication event was generated from the Windows workstation. The historical test source was `192.168.200.245`.

The event was detected successfully and Wazuh invoked:

~~~text
active-response/bin/firewall-drop
~~~

This proved that the detection-to-response pipeline was working.

---

# 11. Important Finding: Stock `firewall-drop` Affected Docker Forwarding

The initial test exposed an important operational problem.

The Pi-hole also hosts Docker services. The stock `firewall-drop` response inserted a DROP rule affecting the Pi-hole's FORWARD chain.

The observed rule was:

~~~text
-A FORWARD -s 192.168.200.245/32 -j DROP
~~~

This interfered with forwarded Docker traffic and temporarily disrupted access to services running behind the Pi-hole.

The rule had to be removed to restore normal connectivity.

The clean firewall state after recovery showed the Docker forwarding chains intact and an empty INPUT chain.

### Engineering conclusion

The problem was not Wazuh detection. The problem was that a generic IP-level firewall response was not appropriate for a host that also participates in Docker forwarding.

> **Operational lesson:** successful automated containment is not automatically safe containment.

This finding led to a redesign rather than continuing to use the stock response.

---

# 12. Custom SSH Containment Design

The stock `firewall-drop` response was replaced with a custom Active Response named:

~~~text
ssh_contain
~~~

The response was designed specifically around the homelab architecture.

### Design requirements

The custom response:

- Extracts the source IP from the Wazuh alert
- Rejects IPv6 addresses
- Restricts containment to `192.168.200.0/24`
- Protects the management workstation
- Targets the Pi-hole INPUT chain
- Restricts containment to TCP port 22
- Does not modify the Docker FORWARD path
- Avoids duplicate rules
- Tags rules with `WAZUH-SSH-CONTAIN`
- Supports Wazuh add/delete Active Response operations
- Uses a five-minute timeout

The management workstation is explicitly protected because testing automated firewall response against the administrator's own source address can otherwise cause an immediate lockout.

---

# 13. Custom Response Implementation

The source script was developed under:

~~~text
/root/wazuh-ar-lab/ssh_contain.py
~~~

and deployed to:

~~~text
/var/ossec/active-response/bin/ssh_contain
~~~

The deployed script uses restricted ownership and permissions:

~~~text
root:wazuh
mode 750
~~~

The source and deployed script SHA-256 hash was verified as:

~~~text
56e3f4acf0a1891d0499bbc2e8136d707db7457432ca04ca789852a216fb3712
~~~

Operational logging is written to:

~~~text
/var/ossec/logs/ssh-contain.log
~~~

Before live integration, the response was tested against:

- Trusted management workstation
- Untrusted host inside the homelab LAN
- Host outside the homelab LAN
- Public IPv4 address
- IPv6 address

The dry-run tests confirmed that only eligible LAN IPv4 addresses could be considered for containment and that the trusted management workstation was excluded.

---

# 14. Wazuh Manager Configuration

The custom command was registered on the Wazuh manager:

~~~xml
<command>
  <name>ssh_contain</name>
  <executable>ssh_contain</executable>
  <timeout_allowed>yes</timeout_allowed>
</command>
~~~

Rule 5710 was connected to the custom response:

~~~xml
<active-response>
  <disabled>no</disabled>
  <command>ssh_contain</command>
  <location>local</location>
  <rules_id>5710</rules_id>
  <timeout>300</timeout>
</active-response>
~~~

A backup of the manager configuration was created before the custom response was deployed:

~~~text
/var/ossec/etc/ossec.conf.before-custom-ssh-response
~~~

The Wazuh manager configuration was validated after the change.

---

# 15. Real Active Response Validation

A controlled SSH authentication event was generated against Pi-hole using the current Windows workstation:

~~~text
192.168.200.182
~~~

The real Wazuh Active Response log showed:

~~~text
Received command=add, srcip=192.168.200.182
~~~

The custom response recognized the management workstation as trusted and correctly refused to install a blocking rule:

~~~text
SAFETY: 192.168.200.182 is trusted; no block installed
~~~

The Pi-hole INPUT chain remained unchanged and the Docker FORWARD chain was not modified.

This validated the most important safety requirement: the automated response could process the real Wazuh event without locking out the management workstation.

---

# 16. Final Containment Behavior

For an eligible, untrusted IPv4 source on the homelab LAN, the custom response is designed to add a temporary rule to the Pi-hole INPUT chain that targets SSH only.

The rule is tagged:

~~~text
WAZUH-SSH-CONTAIN
~~~

This intentionally avoids blocking unrelated forwarded traffic such as:

- Docker networking
- Jellyfin
- Uptime Kuma
- WireGuard forwarding

The Wazuh timeout is:

~~~text
300 seconds
~~~

When the timeout expires, Wazuh sends the corresponding delete operation so the exact containment rule can be removed.

---

# 17. Final Security Workflow

~~~text
┌─────────────────────────┐
│ SSH authentication      │
│ attempt                 │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ Pi-hole sshd            │
│ 192.168.200.178         │
└────────────┬────────────┘
             │
             │ Invalid user
             ▼
┌─────────────────────────┐
│ Wazuh Agent             │
│ Log collection          │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ Wazuh Manager           │
│ Rule 5710               │
│ Level 5                 │
└────────────┬────────────┘
             │
             │ Alert
             ▼
┌─────────────────────────┐
│ Custom Active Response  │
│ ssh_contain             │
└────────────┬────────────┘
             │
             ├─────────────── Trusted source
             │                → No block
             │
             ├─────────────── Invalid scope/IP
             │                → Reject
             │
             ▼
┌─────────────────────────┐
│ Pi-hole INPUT chain     │
│ TCP/22 only             │
│ WAZUH-SSH-CONTAIN       │
└────────────┬────────────┘
             │
             │ 300 seconds
             ▼
┌─────────────────────────┐
│ Wazuh delete operation  │
│ Temporary rule removed  │
└─────────────────────────┘
~~~

---

# 18. Troubleshooting Lessons

## `iptables` itself was not broken

The Pi-hole had a valid ARM64 `iptables-nft` implementation:

~~~text
iptables v1.8.11 (nf_tables)
~~~

Root could successfully inspect and manipulate firewall state.

## `wazuh-execd` runs with root privileges

The Wazuh execution daemon was verified as running as root, allowing Active Response scripts to perform privileged operations.

## Active Response must be enabled on the agent

The Pi-hole initially had Active Response disabled. Changing:

~~~xml
<disabled>yes</disabled>
~~~

to:

~~~xml
<disabled>no</disabled>
~~~

and restarting the agent was necessary.

## Manual execution is not identical to a real Active Response

Directly executing `firewall-drop` with test JSON produced confusing `check_keys` and `continue` behavior.

The actual Wazuh Active Response execution path was the authoritative test.

## Generic firewall responses need architectural awareness

The largest lesson from the first implementation was that firewall automation must account for how the target host is used.

The Pi-hole is not simply an SSH server. It also participates in Docker networking and other homelab services.

A response that blocks an entire source IP through the FORWARD chain can therefore have consequences beyond SSH.

---

# 19. Final Configuration

### Wazuh Manager

~~~xml
<command>
  <name>ssh_contain</name>
  <executable>ssh_contain</executable>
  <timeout_allowed>yes</timeout_allowed>
</command>

<active-response>
  <disabled>no</disabled>
  <command>ssh_contain</command>
  <location>local</location>
  <rules_id>5710</rules_id>
  <timeout>300</timeout>
</active-response>
~~~

### Pi-hole Agent

~~~xml
<active-response>
    <disabled>no</disabled>
    <ca_store>etc/wpk_root.pem</ca_store>
    <ca_verification>yes</ca_verification>
</active-response>
~~~

### Response Scope

~~~text
Eligible source:
192.168.200.0/24 IPv4

Protected management source:
192.168.200.182

Target:
Pi-hole INPUT

Protocol:
TCP

Port:
22

Timeout:
300 seconds

Rule tag:
WAZUH-SSH-CONTAIN
~~~

---

# 20. Evidence

Useful evidence captured during the lab includes:

### Detection rule

~~~text
Rule: 5710
Level: 5
Description: sshd: Attempt to login using a non-existent user
~~~

### Initial stock response

~~~text
-A FORWARD -s 192.168.200.245/32 -j DROP
~~~

This demonstrated why the default response was not appropriate for the Pi-hole's Docker environment.

### Custom response

~~~text
ssh_contain
~~~

### Safety validation

~~~text
Received command=add, srcip=192.168.200.182
SAFETY: 192.168.200.182 is trusted; no block installed
~~~

### Final design

~~~text
Wazuh Rule 5710
        |
        v
ssh_contain
        |
        v
Validate source
        |
        +---- Trusted management IP -> No block
        |
        +---- Invalid scope/IP -> Reject
        |
        +---- Eligible LAN IPv4 -> SSH-only INPUT containment
~~~

---

# 21. Final Takeaway

The SSH detection lab evolved from a basic detection-and-block experiment into a safer automated response design.

The initial implementation proved that Wazuh could detect the SSH event and automatically modify the Pi-hole firewall. The resulting Docker networking disruption then provided a practical reason to redesign the response.

The final implementation separates **detection** from **containment policy**:

- Wazuh detects the SSH event.
- Rule 5710 identifies the invalid-user condition.
- `ssh_contain` validates the source.
- Trusted management addresses are protected.
- Only eligible LAN IPv4 sources are considered.
- Containment targets SSH instead of general forwarded traffic.
- The response is temporary and tied to Wazuh's timeout lifecycle.

This became the first completed automated-response workflow in the Wazuh security lab.

> **Lab status: SSH detection and safety-focused automated containment successfully demonstrated.**
