# Emergency: Mount JuiceFS from Backblaze B2

**Purpose:** Quick access to backup data from B2 cloud storage on any Linux system

**Use Case:** Total homelab loss - need to access backups to start recovery

**Time:** 10-15 minutes to get read access to all backup data

---

## Prerequisites

- Fresh Linux system (Proxmox, Ubuntu, Debian, etc.)
- Internet connectivity
- Backblaze B2 credentials (Application Key ID + Application Key)
- Root access

---

## Quick Mount (5 Steps)

### Step 1: Install Redis

```bash
apt update
apt install -y redis-server

# Start Redis
systemctl enable redis-server
systemctl start redis-server

# Verify
redis-cli ping
# Expected: PONG
```

### Step 2: Install JuiceFS

```bash
# Download JuiceFS
curl -sSL https://d.juicefs.com/install | sh -s ce

# Verify
juicefs version
```

### Step 3: Configure B2 Credentials

You need your Backblaze B2 credentials:
- **Application Key ID** (looks like: `0031a2b3c4d5e6f7890000001`)
- **Application Key** (secret, looks like: `K001abcdef1234567890123456789012`)
- **Bucket Name** (example: `homelab-backup-b2`)
- **Endpoint** (example: `https://s3.us-west-000.backblazeb2.com`)

```bash
# Set environment variables
export AWS_ACCESS_KEY_ID="YOUR_B2_KEY_ID"
export AWS_SECRET_ACCESS_KEY="YOUR_B2_APPLICATION_KEY"

# Or install AWS CLI and configure
apt install -y awscli
aws configure
# Enter B2 credentials when prompted
# Region: us-west-000 (or your B2 region)
```

### Step 4: Download Latest Metadata Backup

```bash
# List available metadata backups
aws s3 ls s3://YOUR-BUCKET-NAME/backup/juicefs/metadata/ \
  --endpoint-url=https://s3.us-west-000.backblazeb2.com | tail -10

# Download latest metadata dump
LATEST=$(aws s3 ls s3://YOUR-BUCKET-NAME/backup/juicefs/metadata/ \
  --endpoint-url=https://s3.us-west-000.backblazeb2.com | \
  tail -1 | awk '{print $4}')

aws s3 cp s3://YOUR-BUCKET-NAME/backup/juicefs/metadata/$LATEST \
  /tmp/juicefs-metadata.dump.gz \
  --endpoint-url=https://s3.us-west-000.backblazeb2.com

# Extract metadata
gunzip /tmp/juicefs-metadata.dump.gz
```

### Step 5: Mount JuiceFS (Read-Only, No Cache)

```bash
# Format JuiceFS (connects to existing B2 data)
juicefs format \
  --storage s3 \
  --bucket https://YOUR-BUCKET-NAME.s3.us-west-000.backblazeb2.com \
  --access-key $AWS_ACCESS_KEY_ID \
  --secret-key $AWS_SECRET_ACCESS_KEY \
  redis://localhost/2 \
  myjfs-recovery

# Load metadata
juicefs load redis://localhost/2 /tmp/juicefs-metadata.dump

# Create mount point
mkdir -p /mnt/juicefs-recovery

# Mount (read-only, no cache)
juicefs mount redis://localhost/2 /mnt/juicefs-recovery \
  --read-only \
  --no-bgjob \
  -d

# Verify mount
df -h | grep juicefs
ls -la /mnt/juicefs-recovery/
```

**Success!** You now have read access to all backup data at `/mnt/juicefs-recovery/`

---

## Access Critical Files

```bash
# Navigate to backup root
cd /mnt/juicefs-recovery/backup/

# List contents
ls -la
# Expected: cephfs/, postgresql/, juicefs-metadata/, docs/

# Access terraform state (CRITICAL for recovery)
ls -lh /mnt/juicefs-recovery/backup/cephfs/config/terraform-proxmox-talos/terraform.tfstate
ls -lh /mnt/juicefs-recovery/backup/cephfs/config/terraform-proxmox-talos/terraform.tfvars

# Access documentation
ls /mnt/juicefs-recovery/backup/cephfs/docs/

# Access PostgreSQL backups
ls -lh /mnt/juicefs-recovery/backup/postgresql/
```

---

## Copy Files for Recovery

```bash
# Create working directory
mkdir -p /root/recovery

# Copy terraform files
cp /mnt/juicefs-recovery/backup/cephfs/config/terraform-proxmox-talos/terraform.tfstate /root/recovery/
cp /mnt/juicefs-recovery/backup/cephfs/config/terraform-proxmox-talos/terraform.tfvars /root/recovery/

# Copy all terraform configuration
cp -r /mnt/juicefs-recovery/backup/cephfs/config/terraform-proxmox-talos/* /root/recovery/

# Verify
ls -lh /root/recovery/terraform.tfstate
# Expected: ~497KB
```

---

## Troubleshooting

### Issue: "Unable to connect to Redis"

```bash
systemctl status redis-server
systemctl restart redis-server
redis-cli ping
```

### Issue: "Access denied" to B2

```bash
# Verify credentials
aws s3 ls s3://YOUR-BUCKET-NAME/ --endpoint-url=https://s3.us-west-000.backblazeb2.com

# Check environment variables
echo $AWS_ACCESS_KEY_ID
echo $AWS_SECRET_ACCESS_KEY
```

### Issue: "Metadata not found"

```bash
# List all available metadata backups
aws s3 ls s3://YOUR-BUCKET-NAME/backup/juicefs/metadata/ \
  --endpoint-url=https://s3.us-west-000.backblazeb2.com

# Download a specific backup by filename
aws s3 cp s3://YOUR-BUCKET-NAME/backup/juicefs/metadata/homelab-backup-YYYYMMDD_HHMMSS.dump.gz \
  /tmp/juicefs-metadata.dump.gz \
  --endpoint-url=https://s3.us-west-000.backblazeb2.com
```

### Issue: Mount fails

```bash
# Check logs
juicefs status redis://localhost/2

# Run filesystem check
juicefs fsck redis://localhost/2

# Try mount with debug
juicefs mount redis://localhost/2 /mnt/juicefs-recovery --read-only --debug
```

---

## Unmount When Done

```bash
# Unmount
umount /mnt/juicefs-recovery

# Or force unmount
fusermount -u /mnt/juicefs-recovery

# Clean up Redis
redis-cli -n 2 FLUSHDB
```

---

## Important Notes

- **Read-only mount** - Cannot modify backup data (safety)
- **No cache** - All data fetched from B2 on demand (slower but simpler)
- **Network required** - Must have internet to access B2
- **Temporary mount** - For recovery purposes, not production use

---

## What's in the Backup

```
/mnt/juicefs-recovery/backup/
├── cephfs/                          ← Complete CephFS backup
│   ├── config/
│   │   ├── terraform-proxmox-talos/ ← CRITICAL: Terraform state & vars
│   │   ├── appdaemon/               ← AppDaemon automation code
│   │   ├── homeassistant/           ← Home Assistant configuration
│   │   └── ...
│   └── docs/                        ← Complete documentation
├── postgresql/                      ← PostgreSQL database dumps (30 days)
│   └── homeassistant-*.dump
└── juicefs-metadata/                ← JuiceFS metadata backups (hourly)
```

---

## Next Steps

Once you have access to backup data:

1. **Read disaster recovery docs:**
   ```bash
   # View the complete loss recovery guide
   cat /mnt/juicefs-recovery/backup/cephfs/docs/06-operations/disaster-recovery/complete-loss-recovery.md

   # Or download from GitHub
   curl -O https://raw.githubusercontent.com/jkempson/disaster-recovery/main/COMPLETE-LOSS-RECOVERY.md
   ```

2. **Extract terraform files** (see above)

3. **Follow complete recovery procedure** in COMPLETE-LOSS-RECOVERY.md

---

**Last Updated:** 2025-11-23
**Purpose:** Emergency read access to B2 backups
**Complexity:** Minimal - just enough to read backup data
**Prerequisites:** Linux system + B2 credentials + Internet
