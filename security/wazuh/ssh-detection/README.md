This document was moved from `security/wazuh-ssh-detection/README.md` so all Wazuh documentation now lives under `security/wazuh/`.

See the complete SSH detection and automated containment documentation here:

# Wazuh SSH Detection & Automated Containment

The SSH security workflow demonstrates Wazuh Rule 5710 detection, investigation of the built-in `firewall-drop` response, the Docker networking side effect discovered during testing, and the final custom `ssh_contain` Active Response.

The final response validates the source address, protects the management workstation, restricts eligible containment to the homelab LAN, and targets TCP/22 in the Pi-hole INPUT chain rather than Docker forwarding traffic.

## Final workflow

```text
SSH authentication attempt
        |
        v
Pi-hole sshd
        |
        v
Wazuh Agent
        |
        v
Wazuh Rule 5710
        |
        v
ssh_contain
        |
        +--> Trusted management source -> No block
        |
        +--> Invalid scope/IP -> Reject
        |
        +--> Eligible LAN IPv4 -> SSH-only INPUT containment
```

For the complete historical investigation, implementation details, configuration, validation evidence, and troubleshooting lessons, see the archived source documentation in the repository history.
