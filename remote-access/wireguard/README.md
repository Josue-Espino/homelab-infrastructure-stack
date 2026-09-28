# WireGuard Remote Access

## Overview

This project documents the WireGuard remote-access infrastructure built for the homelab.

WireGuard was deployed on the Raspberry Pi to provide encrypted remote access to the homelab network. Windows and mobile devices were configured as peers, allowing trusted clients to reach internal homelab services without exposing those services directly to the public internet.

## Architecture

```text
Internet -> Dynamic DNS -> Router Port Forward -> Raspberry Pi WireGuard -> Homelab LAN
                                                      |-> Windows peer
                                                      |-> Mobile peer
```

The Raspberry Pi acts as the WireGuard endpoint. Remote clients connect to the Pi and use the VPN tunnel to reach services on the homelab LAN.

## Peer Configuration

The lab included at least two client peers:

- Windows PC
- Mobile phone

Peer configuration was generated for individual clients rather than sharing a single client identity. The mobile client required disabling another active VPN during initial testing because the competing VPN prevented the WireGuard tunnel from operating correctly.

## Dynamic DNS and Port Forwarding

Dynamic DNS was configured for remote access, and the router forwards the WireGuard UDP service to the Raspberry Pi. The remote-access path is:

```text
Remote Client -> Internet -> Dynamic DNS -> Router Port Forward -> Raspberry Pi -> WireGuard wg0 -> 192.168.200.0/24
```

## Health Monitoring

A WireGuard health-check timer and supporting Uptime Kuma health-check script were created to verify that the VPN service remained operational.

## Validation

The deployment was validated through Windows peer connectivity, mobile peer connectivity, remote access to homelab services, dynamic DNS resolution, router port-forwarding validation, WireGuard interface/status checks, and Uptime Kuma health monitoring.

## Operational Lessons

- Each client should have its own WireGuard peer identity.
- Remote-access troubleshooting should consider competing VPN clients.
- Dynamic DNS provides a stable name when the public IP can change.
- VPN access can reduce the need to expose individual internal services directly.
- Monitoring the VPN endpoint helps detect remote-access failures.

## Security Considerations

WireGuard access should be treated as privileged access to the homelab. Private keys must remain on their respective clients and must never be committed to GitHub. Actual private keys, peer secrets, and router credentials are intentionally excluded.

## Project Status

**Status:** Implemented and validated
