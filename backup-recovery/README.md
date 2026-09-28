# Automated Backup & Recovery

## Overview

This project documents the automated backup workflow used to protect key homelab application data.

The Raspberry Pi writes scheduled backups to an SMB share mounted at `/mnt/homelab-backups`. The backup process focuses on application databases and configuration state that would be difficult to reconstruct manually.

## Architecture

```text
Raspberry Pi
     |
     | automated-backup.sh
     | scheduled by cron
     v
/mnt/homelab-backups
     |
     v
SMB backup share

Backed-up application data:
  +-- Pi-hole gravity.db
  +-- Pi-hole pihole-FTL.db
  +-- Uptime Kuma kuma.db
  +-- Nginx Proxy Manager database.sqlite
```

## Backup Destination

The backup destination is an SMB share mounted locally on the Raspberry Pi:

```text
/mnt/homelab-backups
```

The SMB share provides storage separate from the application's active database locations. This separation allows application data to be copied away from the live service environment.

## Backup Script

The automated process uses:

```text
/usr/local/bin/automated-backup.sh
```

The script copies the selected application database files to the backup destination.

Current backup targets include:

| Application | Data |
|---|---|
| Pi-hole | `gravity.db` |
| Pi-hole | `pihole-FTL.db` |
| Uptime Kuma | `kuma.db` |
| Nginx Proxy Manager | `database.sqlite` |

## Scheduling

The backup script is executed by cron on a daily schedule.

The design keeps the recurring operation on the infrastructure host rather than requiring a manual backup step.

## Validation

The backup workflow was validated by checking:

- The SMB backup mount was available.
- The backup script could access the destination.
- Expected application database files were present in the backup destination.
- The scheduled backup workflow completed successfully.
- Backup files could be inspected independently of the live application database paths.

## Recovery Model

The backup design provides recoverable copies of key application databases, but a complete disaster-recovery test is a separate activity.

A full recovery validation should eventually include:

1. Provisioning or selecting a clean recovery environment.
2. Restoring each backed-up database.
3. Verifying file ownership and permissions.
4. Starting the affected service with the restored data.
5. Validating application functionality.
6. Recording the recovery procedure and expected recovery time.

Until that exercise is performed, the project should be described as **automated backup with recovery capability**, rather than a fully validated disaster-recovery solution.

## Security Considerations

Backup storage should be treated as sensitive infrastructure because application databases may contain configuration, monitoring history, account information, or other operational data.

The SMB backup destination should not be exposed more broadly than necessary. Authentication and write permissions should be limited to the systems and services that require them.

Backup copies should also be protected from accidental deletion or unauthorized modification where practical.

## Operational Lessons

- Automating backups removes dependence on a manual reminder.
- Backing up application state is more useful than backing up only container definitions.
- A separate storage destination reduces dependence on the live application host.
- Backup success should be verified rather than assumed.
- A backup is not fully validated until restoration has been tested.

## Project Status

**Backup automation:** Implemented and validated

**Full restore test:** Not yet documented as completed

The backup system is part of the broader homelab infrastructure and complements the service, monitoring, remote-access, and security layers.