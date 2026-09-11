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
                    │  IP: 192.168.200.245        │
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
- **Wazuh server/test source IP:** `192.168.200.245`
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

# 10. Configuring Automated Firewall Response

On the Wazuh server, the active-response configuration was verified:

```bash
sudo grep -n -A12 -B5 "<active-response>" /var/ossec/etc/ossec.conf
```

The relevant configuration was:

```xml
<active-response>
  <disabled>no</disabled>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>5710</rules_id>
  <timeout>300</timeout>
</active-response>
```

The command definition was also verified:

```xml
<command>
  <name>firewall-drop</name>
  <executable>firewall-drop</executable>
  <timeout_allowed>yes</timeout_allowed>
</command>
```

This means:

- Rule `5710` triggers the response
- `firewall-drop` is executed
- The response occurs locally on the agent where the alert originated
- The block lasts `300` seconds

`300` seconds equals **5 minutes**.

---

# 11. Live Controlled Attack

A real controlled SSH login attempt was generated from the Windows workstation:

```powershell
ssh nonexistentuser@192.168.200.178
```

The Pi-hole responded:

```text
nonexistentuser@192.168.200.178's password:
Permission denied, please try again.
```

The attempt originated from:

```text
192.168.200.245
```

This was an intentional test against the homelab.

---

# 12. Wazuh Detected the Attack

On the Wazuh server, the alert log showed the actual Pi-hole SSH event:

```text
Failed password for invalid user nonexistentuser from 192.168.200.245
```

The alert data included:

```text
parameters.alert.agent.ip: 192.168.200.178
parameters.alert.data.srcip: 192.168.200.245
parameters.alert.data.srcuser: nonexistentuser
parameters.alert.predecoder.program_name: sshd-session
```

Most importantly, the alert showed:

```text
parameters.program: active-response/bin/firewall-drop
```

This confirmed that the active-response mechanism was invoked.

---

# 13. Automated Firewall Block

Immediately after the controlled attack, SSH access from the Wazuh server to Pi-hole was lost.

The Pi-hole firewall was later inspected after reconnecting:

```bash
sudo iptables -L INPUT -n -v --line-numbers
```

During the active response, the INPUT chain contained:

```text
1   ...   DROP   all   --   *   *   192.168.200.245   0.0.0.0/0
```

This demonstrated that Wazuh had automatically inserted a temporary firewall rule blocking the attacking IP.

The rule specifically blocked:

```text
192.168.200.245
```

from reaching the Pi-hole.

---

# 14. Automatic Expiration

The configured response timeout was:

```xml
<timeout>300</timeout>
```

After approximately five minutes, SSH access returned.

We then checked:

```bash
sudo iptables -L INPUT -n -v --line-numbers
```

The result was:

```text
Chain INPUT (policy ACCEPT ...)
num   pkts bytes target     prot opt in     out     source     destination
```

There was **no DROP rule** remaining.

This confirmed that the active response:

1. Added the firewall block.
2. Prevented access from the test source.
3. Automatically removed the block after the configured timeout.
4. Restored normal connectivity.

---

# 15. Final Security Workflow

The completed workflow is:

```
┌─────────────────────────┐
│ Controlled SSH Attack   │
│ 192.168.200.245         │
└────────────┬────────────┘
             │
             │ SSH login attempt
             ▼
┌─────────────────────────┐
│ Pi-hole                 │
│ 192.168.200.178         │
│ sshd-session            │
└────────────┬────────────┘
             │
             │ Failed login
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
│ Active Response         │
│ firewall-drop           │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ Pi-hole iptables        │
│ DROP 192.168.200.245    │
└────────────┬────────────┘
             │
             │ 300 seconds
             ▼
┌─────────────────────────┐
│ Automatic rule removal  │
│ Access restored         │
└─────────────────────────┘
```

---

# 16. Troubleshooting Lessons

## `iptables` itself was not broken

The Pi-hole had a valid ARM64 `iptables-nft` installation.

The system reported:

```text
iptables v1.8.11 (nf_tables)
```

and root could successfully manipulate the INPUT chain.

## `wazuh-execd` runs as root

The Wazuh execution daemon was verified as:

```text
USER: root
EUSER: root
```

This is important because active responses need sufficient privileges to modify firewall state.

## Active response must be enabled on the agent

The Pi-hole initially had:

```xml
<disabled>yes</disabled>
```

Changing it to:

```xml
<disabled>no</disabled>
```

and restarting the agent was necessary.

## Manual execution is not identical to a real active response

Directly feeding JSON to `firewall-drop` produced confusing `check_keys`, `continue`, and stdin behavior.

The successful live test demonstrated that the correct Wazuh execution path was functioning.

## Be careful when testing firewall automation remotely

Because the test source was the Wazuh server itself, a successful firewall-drop response immediately blocked our SSH connection to the Pi-hole.

The five-minute timeout allowed access to recover automatically.

This is a useful operational lesson:

> Never test automated firewall containment remotely without knowing how you will regain access.

---

# 17. Current Configuration

### Wazuh Manager

```xml
<command>
  <name>firewall-drop</name>
  <executable>firewall-drop</executable>
  <timeout_allowed>yes</timeout_allowed>
</command>

<active-response>
  <disabled>no</disabled>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>5710</rules_id>
  <timeout>300</timeout>
</active-response>
```

### Pi-hole Agent

```xml
<active-response>
    <disabled>no</disabled>
    <ca_store>etc/wpk_root.pem</ca_store>
    <ca_verification>yes</ca_verification>
</active-response>
```

### Firewall State After Test

```text
INPUT policy: ACCEPT
Temporary Wazuh DROP rule: removed
```

---

# 18. Security Lab Milestone

## Completed: Wazuh SSH Detection + Automated Firewall Response

### Detection

- [x] Wazuh agent installed on Pi-hole
- [x] Pi-hole connected to Wazuh Manager
- [x] SSH logs collected
- [x] SSH event decoded
- [x] Rule 5710 identified
- [x] Rule 5710 tested with `wazuh-logtest`
- [x] Real SSH attack generated
- [x] Alert generated by Wazuh

### Response

- [x] `firewall-drop` command verified
- [x] Active response enabled on Pi-hole
- [x] Rule 5710 connected to active response
- [x] `iptables` response executed
- [x] Attacking IP automatically blocked
- [x] SSH access successfully interrupted
- [x] 300-second timeout verified
- [x] Firewall rule automatically removed
- [x] Connectivity restored

---

# 19. Evidence

Useful evidence captured during the lab includes:

### Rule detection

```text
Rule: 5710
Level: 5
Description: sshd: Attempt to login using a non-existent user
Source IP: 192.168.200.245
```

### Live attack

```text
Failed password for invalid user nonexistentuser
from 192.168.200.245
```

### Active response

```text
parameters.program: active-response/bin/firewall-drop
```

### Firewall block

```text
DROP all -- 192.168.200.245 0.0.0.0/0
```

### Cleanup

```text
INPUT chain returned to its normal state
No temporary DROP rule remained
```

---

> **Lab status: Detection and automated SSH containment successfully demonstrated.**
