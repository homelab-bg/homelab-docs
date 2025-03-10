---
layout: default
title: Maintenance
nav_order: 5
has_children: true
permalink: /docs/maintenance
---

# Maintenance Procedures
{: .no_toc }

This section covers routine and emergency maintenance procedures for the homelab.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Maintenance Schedule

The following table outlines the routine maintenance schedule for the homelab:

| Task | Frequency | Duration | Impact | 
|:-----|:----------|:---------|:-------|
| OS Updates | Monthly | 1-2 hours | Potential service disruption |
| Application Updates | Weekly | 30-60 minutes | Minor service disruption |
| Database Backups | Daily | 15-30 minutes | None |
| Full System Backup | Weekly | 2-3 hours | None |
| Hardware Inspection | Quarterly | 1 hour | None |
| UPS Testing | Bi-annually | 30 minutes | None |
| Network Audit | Quarterly | 1-2 hours | None |

## Update Procedures

### Operating System Updates

{: .warning }
Always create a snapshot or backup before performing OS updates.

#### Proxmox Updates

```bash
# Update package list
apt update

# Perform system upgrade
apt dist-upgrade

# Check if reboot is needed
if [ -f /var/run/reboot-required ]; then
  echo "Reboot required"
  # Plan for a maintenance window
fi
```

#### TrueNAS Updates

TrueNAS updates are performed through the web UI:

1. Navigate to System > Update
2. Click "Check for Updates"
3. If updates are available, click "Download Updates"
4. Once downloaded, apply the updates during a maintenance window

### Application Updates

Most applications are updated via Kubernetes:

```bash
# Update Helm repository
helm repo update

# Upgrade a specific application
helm upgrade [release-name] [chart] --namespace [namespace] --values values.yaml

# Example for updating Traefik
helm upgrade traefik traefik/traefik --namespace traefik --values traefik-values.yaml
```

## Backup Procedures

The homelab uses a 3-2-1 backup strategy:

- 3 copies of data
- 2 different media types
- 1 off-site copy

### Backup Methods

| Type | Tool | Schedule | Retention | Location |
|:-----|:-----|:---------|:----------|:---------|
| VM Backups | Proxmox Backup | Daily | 7 daily, 4 weekly | Backup server |
| Config Backups | Ansible + Git | On change | Unlimited | GitHub + Local |
| Data Backups | Restic | Daily | 7 daily, 4 weekly, 12 monthly | NAS + B2 |
| Docker Volumes | Velero | Daily | 7 daily | NAS |

### VM Backup Script

```bash
#!/bin/bash
# VM backup script

DATE=$(date +%Y-%m-%d)
BACKUP_DIR="/mnt/backup/$DATE"

# Create backup directory
mkdir -p $BACKUP_DIR

# Backup VMs
echo "Backing up VMs..."
for VM_ID in $(qm list | grep -v VMID | awk '{print $1}')
do
  echo "Backing up VM $VM_ID"
  vzdump $VM_ID --compress --mode snapshot --storage local-zfs
done

echo "Backup completed successfully!"
```

## Troubleshooting

### Common Issues

| Issue | First Response | Next Steps |
|:------|:---------------|:-----------|
| Service unreachable | Check pod status | Examine logs, restart service |
| High CPU/Memory | Identify consuming process | Scale resources or optimize |
| Disk space low | Clear logs and temporary files | Evaluate storage expansion |
| Network connectivity | Check physical connections | Verify switch ports, VLAN settings |

### Diagnostic Commands

#### Kubernetes

```bash
# Check pod status
kubectl get pods -n [namespace]

# View pod logs
kubectl logs -n [namespace] [pod-name]

# Describe a pod for detailed information
kubectl describe pod -n [namespace] [pod-name]

# Check node status
kubectl get nodes
```

#### System

```bash
# Check system resource usage
htop

# Check disk usage
df -h

# Check network connectivity
ping -c 4 192.168.1.1

# View logs
journalctl -u [service] -f
```

## Recovery Procedures

{: .important }
Document all recovery actions taken in the incident log.

### Service Recovery

1. Identify the failing service
2. Check logs for errors
3. Attempt service restart
4. If restart fails, restore from backup
5. Verify service functionality
6. Document the incident

### Full System Recovery

In case of catastrophic failure:

1. Boot from recovery media
2. Restore system from backup
3. Verify hardware functionality
4. Restore VMs and containers
5. Verify network connectivity
6. Test critical services
7. Document the recovery process