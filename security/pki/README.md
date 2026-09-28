# Homelab PKI / Internal TLS

## Overview

This project documents the internal certificate infrastructure used by the homelab.

A self-signed **Homelab Root CA** was created to establish a private trust hierarchy for internal services using the `home.arpa` namespace. The Root CA signs service certificates, and trusted clients can install the Root CA certificate so those service certificates are recognized without public certificate authorities.

## Certificate Architecture

```text
                    Homelab Root CA
                    Self-signed CA
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
monitoring.home.arpa  pihole.home.arpa  jellyfin.home.arpa
      leaf cert           leaf cert          leaf cert
          |                |                |
          +----------------+----------------+
                           |
                     Nginx Proxy Manager
                      Custom SSL certs
```

The Root CA acts as the trust anchor. Individual service certificates are issued from that CA rather than using the Root CA certificate directly for normal service endpoints.

## Homelab Root CA

The Root CA was created specifically for the private homelab environment.

The CA private key was stored separately with restrictive permissions:

```text
homelab-ca.key
mode 600
```

The Root CA certificate was used as the trust anchor for the internal service certificates.

The Root CA was validated as self-signed with a validity period from August 15, 2026 through August 12, 2036.

## Internal Certificates

| Hostname | Role | Issuer |
|---|---|---|
| `monitoring.home.arpa` | Monitoring service | Homelab Root CA |
| `pihole.home.arpa` | Pi-hole web interface | Homelab Root CA |
| `jellyfin.home.arpa` | Jellyfin service | Homelab Root CA |

The `pihole.home.arpa` certificate was verified with a matching DNS SAN.

The Nginx Proxy Manager certificate for `monitoring.home.arpa` was also verified with the expected hostname.

## Nginx Proxy Manager Integration

Nginx Proxy Manager stores the custom SSL certificates under its custom SSL directory.

```text
/data/custom_ssl/npm-1/
/data/custom_ssl/npm-2/
/data/custom_ssl/npm-3/
/data/custom_ssl/pihole/
```

The certificate assignments correspond to the internal services documented above. The `fullchain.pem` files contain the service certificate and the Homelab Root CA chain where applicable, while `privkey.pem` contains the corresponding private key.

## Client Trust

Internal certificates are only trusted by clients that trust the Homelab Root CA.

During Jellyfin remote-access troubleshooting, the Homelab Root CA certificate was installed on the phone. This resolved the certificate trust problem that prevented the Jellyfin mobile application from connecting over the internal HTTPS endpoint.

```text
Homelab Root CA
       |
       v
jellyfin.home.arpa certificate
       |
       v
Nginx Proxy Manager / HTTPS
       |
       v
Trusted mobile client
```

## Validation

- Inspecting certificate issuers.
- Verifying expected DNS SANs.
- Verifying the Root CA was self-signed.
- Inspecting certificate validity periods.
- Verifying Nginx Proxy Manager custom SSL certificate files.
- Confirming the Jellyfin client could establish the HTTPS connection after the Root CA was trusted.

## Security Considerations

The Root CA private key is the most sensitive component of this trust hierarchy. It should remain offline or otherwise tightly protected when not required for certificate issuance.

Private keys for individual service certificates should also remain outside source control.

The Root CA certificate itself can be distributed to trusted clients because it contains public certificate information, but it should only be installed on devices intended to trust the homelab's internal certificate authority.

No private keys or certificate secrets are committed to this repository.

## Project Status

**Status:** Implemented and validated

The internal PKI provides a private trust hierarchy for homelab HTTPS services and integrates with the existing reverse-proxy and remote-access architecture.