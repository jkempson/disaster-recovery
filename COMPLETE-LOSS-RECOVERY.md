# Complete Infrastructure Loss Recovery

**Scenario:** Fire, flood, theft, or other event causing total destruction of all local hardware

**Last Updated:** 2025-11-23
**Recovery Time Estimate:** 4-8 hours active work, 24-48 hours total (includes hardware delivery)
**Data Loss Estimate:** Maximum 24 hours (from last backup)

---

## Important Notes

⚠️ **All commands in this guide are run from the Proxmox host** (the new physical server you're setting up)

**Working Directory:** `/root/recovery` on the Proxmox host
**Recovery Approach:** Proxmox host → Install tools → Restore data → Deploy VMs → Restore cluster

This guide assumes:
- You are SSHed into a fresh Proxmox installation
- You have network connectivity
- You can access Backblaze B2
- All commands run as `root` on the Proxmox host

---

## Prerequisites

Before starting recovery, ensure you have:

### Required Access

- [ ] **Backblaze B2 account access**
  - Account email and password
  - 2FA device or backup codes

- [ ] **This documentation**
  - GitHub: https://github.com/jkempson/disaster-recovery
  - Or: Paper printout of this file
  - Or: Copy stored off-site

- [ ] **Network access**
  - Access to your home network (or ability to recreate it)
  - Router access
  - Internet connectivity

### Required Information

- [ ] **Proxmox credentials**
  - Root password
  - IP addressing scheme (192.168.0.0/24)

- [ ] **B2 credentials**
  - Application key ID
  - Application key
  - Bucket name and region

- [ ] **Network configuration**
  - Router IP
  - DNS servers
  - Gateway configuration

---

## Recovery Overview

```
Day 0: Disaster occurs
  ↓
Day 1-2: Order replacement hardware
  ↓
Day 3: Hardware arrives
  ↓
Phase 1: Install Proxmox (1-2 hours)
  ↓
Phase 2: Restore JuiceFS access (1-2 hours)
  ↓
Phase 3: Extract Terraform files (30 minutes)
  ↓
Phase 4: Rebuild infrastructure (2-3 hours)
  ↓
Phase 5: Restore data (1-2 hours)
  ↓
Phase 6: Verification (1 hour)
  ↓
Complete: Homelab operational (24-48 hours total)
```

---

## Phase 1: Hardware Setup (1-2 hours)

### Step 1.1: Acquire Hardware

**Minimum Requirements:**
- CPU: Intel i5/i7 or AMD Ryzen equivalent (4+ cores)
- RAM: 32GB minimum
- Storage: 500GB NVMe SSD minimum
- Network: Gigabit Ethernet

**Recommended:**
- HP EliteDesk 800 G3 or similar business-class mini PC

### Step 1.2: Install Proxmox VE

1. **Download Proxmox VE ISO:**
   - URL: https://www.proxmox.com/en/downloads/proxmox-virtual-environment
   - Version 8.x recommended

2. **Create bootable USB:**
   ```bash
   # Linux
   dd if=proxmox-ve_*.iso of=/dev/sdX bs=1M status=progress

   # macOS
   dd if=proxmox-ve_*.iso of=/dev/diskX bs=1m

   # Windows: Use Rufus or similar
   ```

3. **Boot from USB and install:**
   - Follow Proxmox installer
   - Network configuration:
     - Hostname: `node1.yourdomain.org`
     - IP: `192.168.0.4` (or your preferred IP)
     - Gateway: `192.168.0.1`
     - DNS: `192.168.0.1` or `1.1.1.1`

4. **Verify network connectivity:**
   ```bash
   ping 8.8.8.8
   ping google.com
   ```

5. **Access Proxmox web interface:**
   - URL: https://192.168.0.4:8006
   - Login: `root`
   - Password: [Your root password]

### Step 1.3: Update Proxmox

```bash
# SSH to Proxmox node
ssh root@192.168.0.4

# Update system
apt update
apt upgrade -y
```

### Step 1.4: Prepare Proxmox Host for Recovery

**Install Required Tools:**

```bash
# Install essential packages
apt install -y git curl wget gnupg software-properties-common \
  build-essential jq unzip ca-certificates

# Install Terraform
wget -O- https://apt.releases.hashicorp.com/gpg | \
  gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  tee /etc/apt/sources.list.d/hashicorp.list
apt update
apt install -y terraform

# Verify Terraform
terraform version

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl

# Verify kubectl
kubectl version --client

# Install Talos CLI
curl -sL https://talos.dev/install | sh

# Verify talosctl
talosctl version --client
```

**Create Directory Structure:**

```bash
# Create working directories
mkdir -p /root/recovery
mkdir -p /mnt/juicefs
mkdir -p /mnt/pve/cephfs
mkdir -p /var/log/recovery

# Set working directory
cd /root/recovery
```

**Set Up Environment:**

```bash
# Create recovery environment file
cat > /root/recovery/.envrc << 'EOF'
# Recovery environment variables
export RECOVERY_DATE=$(date +%Y%m%d)
export RECOVERY_LOG="/var/log/recovery/recovery-${RECOVERY_DATE}.log"
export KUBECONFIG="/root/recovery/kubeconfig.yaml"
export TALOSCONFIG="/root/recovery/talosconfig.yaml"

# Log all commands
exec > >(tee -a "$RECOVERY_LOG") 2>&1

echo "Recovery environment loaded - logging to $RECOVERY_LOG"
EOF

# Source environment
source /root/recovery/.envrc
```

**Checkpoint:** ✅ Proxmox host prepared with all tools installed

---

## Phase 2: Restore JuiceFS Access (1-2 hours)

**See:** [EMERGENCY-MOUNT-B2.md](EMERGENCY-MOUNT-B2.md) for detailed steps

**Quick Summary:**

1. Install Redis and JuiceFS
2. Configure B2 credentials
3. Download latest metadata backup from B2
4. Mount JuiceFS read-only at `/mnt/juicefs-recovery`

After completing these steps, you should have:
- JuiceFS mounted at `/mnt/juicefs-recovery/`
- Access to all backup data
- Terraform state files visible at `/mnt/juicefs-recovery/backup/cephfs/config/terraform-proxmox-talos/`

**Checkpoint:** ✅ JuiceFS mounted with backup data accessible

---

## Phase 3: Extract Terraform Files (30 minutes)

### Step 3.1: Copy Terraform State and Variables

```bash
# Ensure you're in the recovery directory
cd /root/recovery

# Copy critical Terraform files from JuiceFS backup
cp /mnt/juicefs-recovery/backup/cephfs/config/terraform-proxmox-talos/terraform.tfstate .
cp /mnt/juicefs-recovery/backup/cephfs/config/terraform-proxmox-talos/terraform.tfvars .

# Verify file sizes
ls -lh terraform.tfstate terraform.tfvars
# Expected: tfstate ~497KB, tfvars ~1.6KB
```

### Step 3.2: Copy Complete Terraform Directory Structure

```bash
# Copy entire Terraform directory structure
cp -r /mnt/juicefs-recovery/backup/cephfs/config/terraform-proxmox-talos/* .

# Verify structure
ls -la
# Expected: main.tf, variables.tf, modules/, talos/, terraform.tfstate, terraform.tfvars
```

### Step 3.3: Verify Critical Secrets

```bash
# Check that terraform.tfstate contains secrets
grep -q "ceph_admin_key" terraform.tfstate && echo "✓ Ceph key found"
grep -q "homeassistant_token" terraform.tfstate && echo "✓ HA token found"
grep -q "cloudflare_token" terraform.tfstate && echo "✓ Cloudflare token found"
```

**Checkpoint:** ✅ Terraform files extracted and verified

---

## Phase 4: Rebuild Infrastructure (2-3 hours)

### Step 4.1: Initialize Terraform

```bash
cd /root/recovery

# Initialize Terraform
terraform init

# Expected output:
# - Initializing provider plugins...
# - Terraform has been successfully initialized!
```

### Step 4.2: Review Terraform Plan

```bash
# Review what Terraform will create
terraform plan

# Expected output:
# - 6 Proxmox VMs (3 control plane, 3 workers)
# - Talos machine configurations
# - Kubernetes cluster bootstrap
# - Application deployments
# - Secrets creation
# - ~100-200 resources to create
```

**Review carefully:**
- VM names and IPs should match your original setup
- Resource counts should be reasonable
- No unexpected deletions

### Step 4.3: Apply Terraform Configuration

```bash
# Deploy infrastructure
terraform apply

# Type 'yes' when prompted
```

**This will take 30-60 minutes and will:**
1. Create Talos VMs on Proxmox
2. Bootstrap Talos Kubernetes cluster
3. Install CNI (Flannel)
4. Deploy MetalLB load balancer
5. Deploy metrics-server
6. Create all Kubernetes secrets
7. Deploy all applications

**Watch for errors:**
- Ceph-related errors: Expected if Ceph not ready (can fix later)
- Network errors: Check Proxmox networking
- Resource errors: Check Proxmox has enough CPU/RAM

**Checkpoint:** ✅ Terraform apply completed

---

## Phase 5: Restore Data (1-2 hours)

### Step 5.1: Set Up CephFS Access (if using Ceph)

```bash
# Create CephFS mount point on Proxmox
mkdir -p /mnt/pve/cephfs

# Mount CephFS (replace [CEPH_ADMIN_KEY] with actual key from terraform.tfstate)
mount -t ceph node1:6789:/ /mnt/pve/cephfs -o name=admin,secret=[CEPH_ADMIN_KEY]

# Verify
df -h | grep cephfs
```

### Step 5.2: Restore CephFS Data from Backup

```bash
# Install rclone
apt install -y rclone

# Sync data from JuiceFS backup to CephFS
rclone sync /mnt/juicefs-recovery/backup/cephfs/ /mnt/pve/cephfs/ \
  --transfers=8 \
  --checkers=16 \
  --progress \
  --checksum

# This will take 10-30 minutes depending on data size
```

### Step 5.3: Restore PostgreSQL Database

```bash
# Get PostgreSQL pod name
export KUBECONFIG=/root/recovery/kubeconfig.yaml
kubectl get pods -l app=postgresql

# Find latest PostgreSQL backup
ls -lh /mnt/juicefs-recovery/backup/postgresql/

# Copy backup into PostgreSQL pod
BACKUP_FILE=$(ls -t /mnt/juicefs-recovery/backup/postgresql/homeassistant-*.dump | head -1)
kubectl cp $BACKUP_FILE postgresql-0:/tmp/restore.dump

# Restore database
kubectl exec postgresql-0 -- dropdb -U hass homeassistant
kubectl exec postgresql-0 -- createdb -U hass homeassistant
kubectl exec postgresql-0 -- pg_restore -U hass -d homeassistant /tmp/restore.dump

# Cleanup
kubectl exec postgresql-0 -- rm /tmp/restore.dump
```

### Step 5.4: Restart Applications

```bash
# Restart Home Assistant to pick up restored database
kubectl rollout restart deployment/homeassistant

# Restart AppDaemon to reload automations
kubectl rollout restart deployment/appdaemon

# Wait for pods to be ready
kubectl get pods -w
```

**Checkpoint:** ✅ All data restored, applications running

---

## Phase 6: Verification (1 hour)

### Step 6.1: Cluster Health

```bash
# Check all nodes are ready
kubectl get nodes
# Expected: All nodes in Ready state

# Check all pods are running
kubectl get pods -A
# Expected: All pods Running or Completed

# Check services have IPs
kubectl get svc -A
# Expected: LoadBalancer services have EXTERNAL-IP assigned
```

### Step 6.2: Application Access

Test each application's web interface:

```bash
# Test Home Assistant
curl -f https://ha.yourdomain.org
# Expected: HTTP 200

# Test Gitea (adjust IP to your configuration)
curl -f http://192.168.1.115:3000
# Expected: HTTP 200

# Test other applications
curl -f http://192.168.1.122:3000  # Grafana
curl -f http://192.168.1.119       # Nginx
```

### Step 6.3: Functional Testing

**Home Assistant:**
- [ ] Web interface loads
- [ ] Automations present
- [ ] Devices responding
- [ ] History data visible (from restored database)

**AppDaemon:**
- [ ] Logs show "Apps initialized"
- [ ] Test automation (turn on a light)
- [ ] No error messages in logs

**Database:**
- [ ] PostgreSQL accessible
- [ ] Home Assistant connected to database
- [ ] Historical data present

**Checkpoint:** ✅ All verification checks passed

---

## Recovery Complete

### Final Steps

1. **Update DNS/Cloudflare** (if needed)
   - Point domains to new IP addresses
   - Update Cloudflare tunnel configuration

2. **Re-enable automation schedules:**
   ```bash
   # Backup cronjob should already be running
   kubectl get cronjobs
   ```

3. **Monitor for issues:**
   ```bash
   # Watch logs for first few hours
   kubectl logs -f deployment/homeassistant
   kubectl logs -f deployment/appdaemon
   ```

4. **Document recovery:**
   - Total time taken: _____ hours
   - Issues encountered: _____
   - Data loss: _____ hours
   - Update this runbook with any improvements

### Success Criteria

- ✅ All Kubernetes nodes healthy
- ✅ All applications running
- ✅ Home Assistant web interface accessible
- ✅ Automations functioning
- ✅ Historical data present
- ✅ No critical errors in logs

---

## Troubleshooting

### Issue: JuiceFS mount fails

**Error:** `unable to connect to Redis`

**Solution:**
```bash
systemctl status redis-server
systemctl restart redis-server
redis-cli ping
```

### Issue: Terraform apply fails with Ceph errors

**Error:** `failed to connect to Ceph monitors`

**Solution:**
- Deploy without Ceph initially (use local storage)
- Set up Ceph cluster separately
- Update Terraform to use Ceph once ready

### Issue: PostgreSQL restore fails

**Error:** `role "hass" does not exist`

**Solution:**
```bash
kubectl exec postgresql-0 -- psql -U postgres -c "CREATE USER hass WITH PASSWORD 'hasspass123';"
kubectl exec postgresql-0 -- psql -U postgres -c "CREATE DATABASE homeassistant OWNER hass;"
# Then retry restore
```

### Issue: Applications not starting

**Error:** `pods stuck in Pending`

**Check:**
```bash
kubectl describe pod <pod-name>
# Look for: insufficient resources, image pull errors, PVC mounting issues
```

**Common causes:**
- CephFS not mounted (mount manually)
- Insufficient resources (reduce replicas temporarily)
- Container images not pulling (check network)

---

## Prevention for Next Time

**After completing recovery:**

1. **Print this document** and store in safe location
2. **Test recovery procedure** quarterly
3. **Document lessons learned** from this recovery
4. **Update emergency contacts** if any changed
5. **Verify backups** are running correctly
6. **Consider additional backup location**

---

## Related Documentation

- [EMERGENCY-MOUNT-B2.md](EMERGENCY-MOUNT-B2.md) - Quick JuiceFS mount guide
- [JUICEFS-DISASTER-RECOVERY.md](JUICEFS-DISASTER-RECOVERY.md) - Detailed JuiceFS recovery procedures
- GitHub Repository: https://github.com/jkempson/disaster-recovery

---

**Document Version:** 1.0
**Last Tested:** NEVER ⚠️
**Next Test Due:** [Schedule quarterly testing]
