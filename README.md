# Disaster Recovery Documentation

This repository contains disaster recovery procedures and guides for complete homelab recovery.

## Quick Start

**🚨 If your homelab is completely destroyed:**

1. **First:** [EMERGENCY-MOUNT-B2.md](EMERGENCY-MOUNT-B2.md) - Get read access to backups (10-15 minutes)
2. **Then:** [COMPLETE-LOSS-RECOVERY.md](COMPLETE-LOSS-RECOVERY.md) - Full infrastructure rebuild (4-8 hours)

## Contents

### Complete Loss Recovery

- **[EMERGENCY-MOUNT-B2.md](EMERGENCY-MOUNT-B2.md)** - Quick mount JuiceFS from Backblaze B2 for emergency backup access
  - **Time:** 10-15 minutes
  - **Use Case:** Total homelab loss - need immediate access to backups
  - **Outcome:** Read-only access to all backup data

- **[COMPLETE-LOSS-RECOVERY.md](COMPLETE-LOSS-RECOVERY.md)** - Complete infrastructure rebuild from scratch
  - **Time:** 4-8 hours active work, 24-48 hours total
  - **Scenario:** Fire, flood, theft - all hardware destroyed
  - **Steps:** Hardware setup → JuiceFS restore → Terraform rebuild → Data restore → Verification

### Partial Recovery

- **[JUICEFS-DISASTER-RECOVERY.md](JUICEFS-DISASTER-RECOVERY.md)** - Detailed JuiceFS recovery procedures
  - **Scenario 1:** Complete system failure (Redis + local system lost)
  - **Scenario 2:** Redis failure only (system intact)
  - **Scenario 3:** Partial data recovery from older backup

## Usage

### For Total Loss

```bash
# On a fresh Proxmox/Linux system
wget https://raw.githubusercontent.com/jkempson/disaster-recovery/main/EMERGENCY-MOUNT-B2.md
# Follow the emergency mount guide first
# Then proceed with complete loss recovery
```

### For Reference

```bash
# Clone this repository for offline access
git clone https://github.com/jkempson/disaster-recovery.git
cd disaster-recovery
```

### Print for Emergency

Print [EMERGENCY-MOUNT-B2.md](EMERGENCY-MOUNT-B2.md) and [COMPLETE-LOSS-RECOVERY.md](COMPLETE-LOSS-RECOVERY.md) and store in a safe location (fireproof safe, off-site location, etc.)

## Recovery Time Objectives

| Scenario | Recovery Time | Data Loss | Guide |
|----------|--------------|-----------|-------|
| Total infrastructure loss | 4-8 hours (active) | Max 24 hours | [COMPLETE-LOSS-RECOVERY.md](COMPLETE-LOSS-RECOVERY.md) |
| JuiceFS mount access only | 10-15 minutes | None | [EMERGENCY-MOUNT-B2.md](EMERGENCY-MOUNT-B2.md) |
| Redis metadata loss | 30-60 minutes | Max 1-6 hours | [JUICEFS-DISASTER-RECOVERY.md](JUICEFS-DISASTER-RECOVERY.md) |

## Prerequisites for Recovery

**Required Information** (store in password manager):
- Backblaze B2 application key ID and secret
- B2 bucket name and region
- Network configuration (IP scheme, gateway, DNS)
- Proxmox credentials

**Required Access:**
- Fresh Linux/Proxmox system
- Internet connectivity
- Password manager with B2 credentials

## Security Note

All credentials and sensitive information have been redacted from these guides. Replace placeholders (e.g., `<YOUR_B2_KEY_ID>`) with your actual values from your secure password manager.

**Never commit actual secrets to this repository.**

---

**Last Updated:** 2025-11-23
**Repository:** https://github.com/jkempson/disaster-recovery
