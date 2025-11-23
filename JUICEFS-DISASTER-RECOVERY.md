# JuiceFS Disaster Recovery Guide

## Overview

This guide covers complete disaster recovery procedures for a JuiceFS filesystem using Backblaze B2 object storage and Redis metadata storage, with automatic hourly metadata backups.

## System Configuration

Replace these values with your specific configuration:

- **Filesystem Name**: `myjfs-b2` (your filesystem name)
- **Object Storage**: `s3://<YOUR_B2_BUCKET>/<FILESYSTEM_NAME>/`
- **Storage Provider**: Backblaze B2
- **B2 Region**: `eu-central-003` (or your region)
- **Metadata Engine**: Redis (localhost:6379, database 2)
- **Mount Point**: `/mnt/juicefs-host`
- **Cache Directory**: `/mnt/juicefs-cache` (400GB recommended)

## Critical Credentials

**Required credentials** (store securely in password manager):

- **B2 Key ID**: Found in Backblaze B2 console
- **B2 Application Key**: Generated when creating application key
- **B2 Endpoint**: `https://s3.<region>.backblazeb2.com`
- **Filesystem UUID**: Obtained during filesystem creation

These credentials are used as S3-compatible environment variables:
```bash
export AWS_ACCESS_KEY_ID=<YOUR_B2_KEY_ID>
export AWS_SECRET_ACCESS_KEY=<YOUR_B2_APPLICATION_KEY>
export AWS_DEFAULT_REGION=<YOUR_B2_REGION>
```

## Recovery Scenarios

### Scenario 1: Complete System Failure (Redis + Local System Lost)

**Prerequisites:**
- New Linux system with JuiceFS installed
- Backblaze B2 credentials from password manager
- Access to B2 bucket

---

#### Step 1: Install JuiceFS

```bash
# Download and install JuiceFS (check for latest version)
wget https://github.com/juicedata/juicefs/releases/download/v1.1.2/juicefs-1.1.2-linux-amd64.tar.gz
tar -xzf juicefs-*.tar.gz
sudo mv juicefs /usr/local/bin/
chmod +x /usr/local/bin/juicefs
```

---

#### Step 2: Install and Configure Redis

```bash
# Install Redis
sudo apt update && sudo apt install redis-server -y

# Configure Redis
sudo sed -i 's/^# maxmemory <bytes>/maxmemory 2gb/' /etc/redis/redis.conf
sudo sed -i 's/^# maxmemory-policy noeviction/maxmemory-policy allkeys-lru/' /etc/redis/redis.conf

# Enable persistence
sudo sed -i 's/^save 900 1/save 900 1/' /etc/redis/redis.conf
sudo sed -i 's/^save 300 10/save 300 10/' /etc/redis/redis.conf
sudo sed -i 's/^save 60 10000/save 60 10000/' /etc/redis/redis.conf

# Start Redis
sudo systemctl enable redis-server
sudo systemctl start redis-server

# Verify Redis is running
redis-cli ping
# Should return: PONG
```

---

#### Step 3: Configure Backblaze B2 Credentials

```bash
# Set environment variables for B2 (S3-compatible API)
export AWS_ACCESS_KEY_ID=<YOUR_B2_KEY_ID>
export AWS_SECRET_ACCESS_KEY=<YOUR_B2_APPLICATION_KEY>
export AWS_DEFAULT_REGION=<YOUR_B2_REGION>

# Add to shell profile for persistence
cat >> ~/.bashrc << 'EOF'
export AWS_ACCESS_KEY_ID=<YOUR_B2_KEY_ID>
export AWS_SECRET_ACCESS_KEY=<YOUR_B2_APPLICATION_KEY>
export AWS_DEFAULT_REGION=<YOUR_B2_REGION>
EOF

source ~/.bashrc
```

**Note**: Replace `<YOUR_B2_KEY_ID>`, `<YOUR_B2_APPLICATION_KEY>`, and `<YOUR_B2_REGION>` with your actual values from Bitwarden/password manager.

---

#### Step 4: Find Latest Metadata Backup

**Option A: Automatic Backups (Hourly, JuiceFS built-in)**
```bash
# List available automatic backups stored in B2
aws s3 ls s3://<YOUR_B2_BUCKET>/<FILESYSTEM_NAME>/meta/ \
  --endpoint-url https://s3.<YOUR_B2_REGION>.backblazeb2.com | tail -5
```

**Option B: Manual Backups (6-hourly, complete)**
```bash
# List manual backups in local storage (if accessible)
ls -lth /mnt/pve/cephfs/juicefs-db-backups/ | head -10

# Or from B2
aws s3 ls s3://<YOUR_B2_BUCKET>/backups/juicefs-metadata/ \
  --endpoint-url https://s3.<YOUR_B2_REGION>.backblazeb2.com | tail -5
```

---

#### Step 5: Restore Metadata

**Method A: From Automatic Dump (Fastest)**
```bash
# Download latest automatic metadata dump from B2
LATEST_DUMP=$(aws s3 ls s3://<YOUR_B2_BUCKET>/<FILESYSTEM_NAME>/meta/ \
  --endpoint-url https://s3.<YOUR_B2_REGION>.backblazeb2.com | tail -1 | awk '{print $4}')

aws s3 cp s3://<YOUR_B2_BUCKET>/<FILESYSTEM_NAME>/meta/$LATEST_DUMP ./ \
  --endpoint-url https://s3.<YOUR_B2_REGION>.backblazeb2.com

# Load metadata into Redis database (use database 2 or your configured database)
juicefs load redis://localhost/2 $LATEST_DUMP

# Verify filesystem metadata
juicefs fsck redis://localhost/2
```

**Method B: From Local Backup (If Available)**
```bash
# Extract latest backup
LATEST_BACKUP=$(ls -t /path/to/backups/juicefs-metadata-*.tar.gz | head -1)
tar -xzf $LATEST_BACKUP -C /tmp/

# Load metadata into Redis database 2
juicefs load redis://localhost/2 /tmp/juicefs-metadata-*.json

# Optionally restore Redis snapshot for faster recovery
sudo systemctl stop redis-server
sudo cp /tmp/redis-dump-*.rdb /var/lib/redis/dump.rdb
sudo chown redis:redis /var/lib/redis/dump.rdb
sudo systemctl start redis-server
```

**Method C: From B2 Complete Backup**
```bash
# Download latest complete backup
LATEST_BACKUP=$(aws s3 ls s3://<YOUR_B2_BUCKET>/backups/juicefs-metadata/ \
  --endpoint-url https://s3.<YOUR_B2_REGION>.backblazeb2.com | tail -1 | awk '{print $4}')

aws s3 cp s3://<YOUR_B2_BUCKET>/backups/juicefs-metadata/$LATEST_BACKUP ./ \
  --endpoint-url https://s3.<YOUR_B2_REGION>.backblazeb2.com

# Extract and load
tar -xzf $LATEST_BACKUP
juicefs load redis://localhost/2 juicefs-metadata-*.json
```

---

#### Step 6: Mount JuiceFS

```bash
# Create mount and cache directories
sudo mkdir -p /mnt/juicefs-host /mnt/juicefs-cache

# Ensure credentials are set
export AWS_ACCESS_KEY_ID=<YOUR_B2_KEY_ID>
export AWS_SECRET_ACCESS_KEY=<YOUR_B2_APPLICATION_KEY>
export AWS_DEFAULT_REGION=<YOUR_B2_REGION>

# Mount filesystem
juicefs mount redis://localhost/2 /mnt/juicefs-host \
  --cache-dir /mnt/juicefs-cache \
  --cache-size 400000 \
  --background \
  --writeback

# Verify mount
df -h /mnt/juicefs-host
mount | grep juicefs
```

---

#### Step 7: Verify Recovery

```bash
# Run filesystem check
juicefs fsck redis://localhost/2

# Check storage statistics
juicefs status redis://localhost/2

# Verify file count
find /mnt/juicefs-host -maxdepth 1 -type d

# Test critical directories
ls -la /mnt/juicefs-host/

# Test file access
cat /mnt/juicefs-host/<path-to-important-file> | head -5
```

---

#### Step 8: Restore File Permissions

After metadata restore, ownership may need correction. Adjust these commands for your specific services:

```bash
# Example: Fix application datastore permissions
sudo chown -R <service_user>:<service_group> /mnt/juicefs-host/<datastore_path>/

# Verify permissions
ls -la /mnt/juicefs-host/<datastore_path>/ | head -5
```

---

#### Step 9: Configure Systemd Service (Optional)

```bash
# Create systemd service for auto-mount on boot
sudo tee /etc/systemd/system/juicefs-mount.service > /dev/null << 'EOF'
[Unit]
Description=JuiceFS Mount
After=network-online.target
Wants=network-online.target

[Service]
Type=forking
Environment=AWS_ACCESS_KEY_ID=<YOUR_B2_KEY_ID>
Environment=AWS_SECRET_ACCESS_KEY=<YOUR_B2_APPLICATION_KEY>
Environment=AWS_DEFAULT_REGION=<YOUR_B2_REGION>
ExecStart=/usr/local/bin/juicefs mount redis://localhost/2 /mnt/juicefs-host --cache-dir /mnt/juicefs-cache --cache-size 400000 --background --writeback
ExecStop=/usr/local/bin/juicefs umount /mnt/juicefs-host
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# Enable and start service
sudo systemctl daemon-reload
sudo systemctl enable juicefs-mount.service
sudo systemctl start juicefs-mount.service
```

**Security Note**: Storing credentials in systemd service files is convenient but less secure. Consider using systemd credentials or environment files with restricted permissions.

---

### Scenario 2: Redis Failure Only (System Intact)

**Prerequisites:**
- JuiceFS and system still functional
- Only Redis metadata lost
- File access to local backups available

---

#### Step 1: Stop JuiceFS Mount and Services

```bash
# Stop services using JuiceFS (adjust for your services)
sudo systemctl stop <your-services-using-juicefs>

# Unmount JuiceFS
sudo umount /mnt/juicefs-host

# Or force unmount if needed
sudo pkill juicefs
sleep 2
sudo umount -l /mnt/juicefs-host
```

---

#### Step 2: Reset Redis

```bash
# Stop Redis
sudo systemctl stop redis-server

# Clear Redis data
sudo rm -f /var/lib/redis/dump.rdb /var/lib/redis/appendonly.aof

# Start Redis
sudo systemctl start redis-server

# Verify clean state
redis-cli -n 2 DBSIZE
# Should return (integer) 0
```

---

#### Step 3: Restore from Latest Local Backup

```bash
# Find latest local backup
LATEST_BACKUP=$(ls -t /path/to/backups/juicefs-metadata-*.tar.gz | head -1)
echo "Restoring from: $LATEST_BACKUP"

# Extract backup
tar -xzf $LATEST_BACKUP -C /tmp/

# Load metadata into database 2
juicefs load redis://localhost/2 /tmp/juicefs-metadata-*.json

# Verify
juicefs fsck redis://localhost/2
```

---

#### Step 4: Remount JuiceFS

```bash
# Ensure credentials are set
export AWS_ACCESS_KEY_ID=<YOUR_B2_KEY_ID>
export AWS_SECRET_ACCESS_KEY=<YOUR_B2_APPLICATION_KEY>
export AWS_DEFAULT_REGION=<YOUR_B2_REGION>

# Mount
juicefs mount redis://localhost/2 /mnt/juicefs-host \
  --cache-dir /mnt/juicefs-cache \
  --cache-size 400000 \
  --background \
  --writeback

# Verify
df -h /mnt/juicefs-host
ls -la /mnt/juicefs-host/
```

---

#### Step 5: Restore Services

```bash
# Restart services that use JuiceFS
sudo systemctl start <your-services-using-juicefs>

# If using NFS exports, refresh them
sudo exportfs -ra

# Verify NFS clients (if applicable)
showmount -a
```

---

### Scenario 3: Partial Data Recovery

**Use Case**: Need to recover specific files from an older backup

---

#### Step 1: Create Recovery Directory

```bash
sudo mkdir -p /mnt/juicefs-recovery
```

---

#### Step 2: Load Backup to Temporary Database

```bash
# Extract backup to recover from
tar -xzf /path/to/older-backup.tar.gz -C /tmp/

# Load to Redis database 3 (temporary)
juicefs load redis://localhost/3 /tmp/juicefs-metadata-*.json
```

---

#### Step 3: Mount Recovery Filesystem

```bash
# Mount with temporary cache
juicefs mount redis://localhost/3 /mnt/juicefs-recovery \
  --cache-dir /tmp/recovery-cache \
  --background
```

---

#### Step 4: Copy Required Files

```bash
# Copy specific directories or files
cp -a /mnt/juicefs-recovery/path/to/needed/data /mnt/juicefs-host/path/to/restore/

# Or use rsync for large datasets
rsync -avP /mnt/juicefs-recovery/data/ /mnt/juicefs-host/data/

# Fix ownership after copy if needed
sudo chown -R <user>:<group> /mnt/juicefs-host/data/
```

---

#### Step 5: Cleanup

```bash
# Unmount recovery filesystem
sudo umount /mnt/juicefs-recovery

# Clear temporary Redis database
redis-cli SELECT 3
redis-cli FLUSHDB

# Remove temporary cache
rm -rf /tmp/recovery-cache /tmp/juicefs-metadata-* /tmp/redis-dump-*
```

---

## Recovery Time Estimates

- **Metadata Restore (Local Storage)**: 2-5 minutes
- **Metadata Restore (B2 Download)**: 5-10 minutes
- **Mount Process**: 1-2 minutes
- **Initial File Access**: Immediate (cached) to 2-5 minutes (B2 download)
- **Full Performance Restoration**: 15-30 minutes (cache warming)
- **Complete System Recovery**: 30-60 minutes

## Data Loss Expectations

| Backup Type | Frequency | Max Data Loss |
|-------------|-----------|---------------|
| Automatic Dumps | Hourly | 1 hour of metadata changes |
| Manual Backups | Every 6 hours | 6 hours of metadata changes |
| File Content | N/A | None (stored in B2) |

**Note**: File content is never lost as it's stored in B2. Only metadata changes (new files, renames, deletes) since last backup may be lost.

## Post-Recovery Verification Checklist

Use this checklist after any recovery operation:

- [ ] **Filesystem Check**: Run `juicefs fsck redis://localhost/2` - no errors
- [ ] **File Count**: Verify expected inode count
- [ ] **Storage Size**: Matches expected usage
- [ ] **Critical Directories Exist**: All important directories accessible
- [ ] **Permissions Correct**: Service accounts have proper ownership
- [ ] **Services Accessible**: All dependent services can access JuiceFS
- [ ] **Test File Operations**:
  - [ ] Can read existing files
  - [ ] Can create new files
  - [ ] Can delete files
  - [ ] Can modify files
- [ ] **Performance Acceptable**:
  - [ ] File access < 5 seconds for cached files
  - [ ] No excessive B2 API errors in logs

## Backup Locations

### Automatic Backups (JuiceFS Built-in)

- **Frequency**: Hourly (JuiceFS automatic)
- **Location**: `s3://<YOUR_B2_BUCKET>/<FILESYSTEM_NAME>/meta/`
- **Format**: `dump-YYYY-MM-DD-HHMMSS.json.gz`
- **Size**: ~14MB compressed (varies with metadata size)
- **Content**: Metadata snapshot only
- **Retention**: Managed by JuiceFS (typically 7-14 days)

### Manual Backups (Script-based)

- **Frequency**: Every 6 hours (configurable via cron: `0 */6 * * *`)
- **Primary Location**: Local storage (e.g., Ceph, NFS, local disk)
- **Secondary Location**: `s3://<YOUR_B2_BUCKET>/backups/juicefs-metadata/` (B2)
- **Format**: `juicefs-metadata-YYYYMMDD-HHMMSS.tar.gz`
- **Size**: ~140MB compressed
- **Content**: Metadata JSON dump + Redis snapshot
- **Retention**:
  - Local: 30 days (configurable)
  - B2: 90 days (configurable)

### Accessing Backups

**List Local Backups:**
```bash
ls -lth /path/to/local/backups/
```

**List B2 Backups:**
```bash
aws s3 ls s3://<YOUR_B2_BUCKET>/backups/juicefs-metadata/ \
  --endpoint-url https://s3.<YOUR_B2_REGION>.backblazeb2.com
```

**Download Specific Backup:**
```bash
aws s3 cp s3://<YOUR_B2_BUCKET>/backups/juicefs-metadata/juicefs-metadata-20251123-120000.tar.gz ./ \
  --endpoint-url https://s3.<YOUR_B2_REGION>.backblazeb2.com
```

## Important Files and Locations

- **Systemd Service**: `/etc/systemd/system/juicefs-mount.service`
- **Backup Script**: `/usr/local/bin/backup-juicefs-metadata.sh`
- **Backup Log**: `/var/log/juicefs-backup.log`
- **Redis Config**: `/etc/redis/redis.conf`
- **NFS Exports** (if used): `/etc/exports`
- **Local Backups**: Configure based on your setup
- **Local Cache**: `/mnt/juicefs-cache/` (400GB recommended)

## Common Issues and Solutions

### Issue: "Transport endpoint is not connected"

**Cause**: JuiceFS mount is stale or crashed

**Solution**:
```bash
sudo umount -l /mnt/juicefs-host
juicefs mount redis://localhost/2 /mnt/juicefs-host --background --writeback
```

---

### Issue: "Permission denied" errors

**Cause**: Incorrect ownership after restore

**Solution**:
```bash
sudo chown -R <service_user>:<service_group> /mnt/juicefs-host/<path>/
sudo systemctl restart <affected-service>
```

---

### Issue: Slow file access after recovery

**Cause**: Empty local cache, files need download from B2

**Solution**:
- **Normal behavior** - cache will warm up over 15-30 minutes
- Files accessed will be cached locally
- Frequently used files will become fast
- Monitor with: `df -h /mnt/juicefs-cache`

---

### Issue: Redis "OOM command not allowed"

**Cause**: Redis out of memory

**Solution**:
```bash
# Check Redis memory
redis-cli INFO memory

# Increase maxmemory in /etc/redis/redis.conf
sudo sed -i 's/maxmemory 2gb/maxmemory 4gb/' /etc/redis/redis.conf
sudo systemctl restart redis-server
```

---

### Issue: "Filesystem is read-only"

**Cause**: JuiceFS mounted in read-only mode after errors

**Solution**:
```bash
# Remount in read-write mode
sudo umount /mnt/juicefs-host
juicefs mount redis://localhost/2 /mnt/juicefs-host --background --writeback
```

---

## Testing Recovery Procedure

It's recommended to test recovery procedures regularly. Use this quick test:

```bash
# 1. Mount to temporary location with latest backup
mkdir -p /tmp/juicefs-test
LATEST=$(ls -t /path/to/backups/*.tar.gz | head -1)
tar -xzf $LATEST -C /tmp/
juicefs load redis://localhost/5 /tmp/juicefs-metadata-*.json
juicefs mount redis://localhost/5 /tmp/juicefs-test --background

# 2. Verify critical files exist
ls -la /tmp/juicefs-test/

# 3. Test file access
cat /tmp/juicefs-test/<important-file> | head

# 4. Cleanup
sudo umount /tmp/juicefs-test
redis-cli SELECT 5 && redis-cli FLUSHDB
rm -rf /tmp/juicefs-metadata-* /tmp/redis-dump-*
```

---

## Backup Script Template

Here's a template backup script for reference:

```bash
#!/bin/bash
# JuiceFS Metadata Backup Script

set -euo pipefail

# B2 Credentials
export AWS_ACCESS_KEY_ID="<YOUR_B2_KEY_ID>"
export AWS_SECRET_ACCESS_KEY="<YOUR_B2_APPLICATION_KEY>"
export AWS_DEFAULT_REGION="<YOUR_B2_REGION>"

# Configuration
REDIS_DB="2"
BACKUP_NAME="juicefs-metadata"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
LOCAL_BACKUP_DIR="/path/to/local/backups"
B2_BUCKET="<YOUR_B2_BUCKET>"
B2_PREFIX="backups/juicefs-metadata"
B2_ENDPOINT="https://s3.<YOUR_B2_REGION>.backblazeb2.com"
TEMP_DIR="/tmp/juicefs-backup-${TIMESTAMP}"

# Create directories
mkdir -p "$LOCAL_BACKUP_DIR" "$TEMP_DIR"

# Export JuiceFS metadata
/usr/local/bin/juicefs dump redis://localhost/${REDIS_DB} "${TEMP_DIR}/${BACKUP_NAME}-${TIMESTAMP}.json"

# Backup Redis
redis-cli BGSAVE
sleep 5
cp /var/lib/redis/dump.rdb "${TEMP_DIR}/redis-dump-${TIMESTAMP}.rdb"

# Create tarball
cd "$TEMP_DIR"
tar -czf "${BACKUP_NAME}-${TIMESTAMP}.tar.gz" *.json *.rdb

# Copy to local storage
cp "${TEMP_DIR}/${BACKUP_NAME}-${TIMESTAMP}.tar.gz" "$LOCAL_BACKUP_DIR/"

# Upload to B2
aws s3 cp "${TEMP_DIR}/${BACKUP_NAME}-${TIMESTAMP}.tar.gz" \
  "s3://${B2_BUCKET}/${B2_PREFIX}/" \
  --endpoint-url "${B2_ENDPOINT}"

# Cleanup old backups (30 days local, 90 days B2)
find "$LOCAL_BACKUP_DIR" -name "${BACKUP_NAME}-*.tar.gz" -mtime +30 -delete

# Cleanup temp
rm -rf "$TEMP_DIR"

echo "Backup completed: ${BACKUP_NAME}-${TIMESTAMP}.tar.gz"
```

---

## Emergency Contact Information

**Store this information securely:**

- **B2 Account**: Your Backblaze account email
- **B2 Bucket Name**: `<YOUR_B2_BUCKET>`
- **B2 Region**: `<YOUR_B2_REGION>`
- **B2 Endpoint**: `https://s3.<YOUR_B2_REGION>.backblazeb2.com`
- **Credentials Location**: Password manager (Bitwarden, 1Password, etc.)
- **Redis Database**: localhost:6379/2 (or your configured database number)
- **Filesystem UUID**: Stored in Redis metadata

---

**Last Updated:** 2025-11-23
**For:** JuiceFS + Backblaze B2 Deployment
**License:** MIT
