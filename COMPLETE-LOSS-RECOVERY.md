# Complete Infrastructure Loss Recovery

**Scenario:** Fire, flood, theft, or other event causing total destruction of all local hardware

**Last Updated:** 2025-11-23
**Recovery Time Estimate:** 4-8 hours active work, 24-48 hours total (includes hardware delivery)
**Data Loss Estimate:** Maximum 24 hours (from last backup)

📝 **NOTE:** This guide exists in TWO locations:
- **Local:** `/mnt/pve/cephfs/docs/06-operations/disaster-recovery/complete-loss-recovery.md`
- **GitHub:** https://github.com/jkempson/disaster-recovery/blob/main/COMPLETE-LOSS-RECOVERY.md

**If you update one, update the other to keep them in sync!**

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

**Your Original Setup (for reference):**
- **3 identical nodes** (same device model)
- **CPU:** Intel Celeron N5105, 4 cores, 2.0GHz
- **RAM:** node1: 32GB, node2/node3: 15GB
- **Storage:**
  - System disk (for Proxmox OS)
  - 466GB SSD (for Ceph OSD)
- **Network:**
  - **3× Gigabit Ethernet NICs** (enp2s0, enp4s0, enp5s0)
  - **4th NIC available** (enp3s0) but unused

**Minimum Requirements for 3-Node Cluster:**
- **CPU:** 4+ cores (Intel Celeron N5105 or better)
- **RAM:**
  - node1: 32GB recommended (hosts more VMs)
  - node2/node3: 15GB minimum each
- **Storage (per node):**
  - 1× System disk: 100GB+ (for Proxmox OS)
  - 1× Ceph OSD disk: 466GB+ SSD (for distributed storage)
- **Network:**
  - **3× Gigabit Ethernet NICs minimum** (MTU 9000 support required)
  - **NIC 1 (vmbr0):** Proxmox management + Ceph cluster traffic (MTU 9000)
  - **NIC 2 (vmbr2):** Internal isolated network (MTU 9000)
  - **NIC 3 (vmbr1):** VM-only network (no MTU requirement)
  - All NICs must support MTU 9000 for Ceph performance optimization

**Hardware Options:**
- **Exact match:** Same device models as original setup
- **Alternative:** Business-class mini PC with similar specs
- **Budget:** Used enterprise small form factor PCs

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
   - Follow Proxmox installer prompts
   - **During installation, configure only the PRIMARY network interface (will create vmbr0)**
   - Additional network bridges will be configured in Step 1.3.1

   **Primary Network Configuration (vmbr0):**
   - **Interface:** First NIC (will become vmbr0)
   - **IP Addresses:**
     - node1: `192.168.0.4/8` (not /24!)
     - node2: `192.168.0.9/8`
     - node3: `192.168.0.6/8`
   - **Gateway:** `192.168.0.1`
   - **DNS:** `192.168.0.1` (your router) or `1.1.1.1` (Cloudflare)
   - **Hostnames:**
     - node1: `node1.yourdomain.org`
     - node2: `node2.yourdomain.org`
     - node3: `node3.yourdomain.org`

4. **Verify network connectivity:**
   ```bash
   ping 8.8.8.8
   ping google.com

   # Verify nodes can reach each other (after all 3 are installed)
   ping 192.168.0.9  # from node1 to node2
   ping 192.168.0.6  # from node1 to node3
   ```

5. **Access Proxmox web interface:**
   - **node1:** https://192.168.0.4:8006
   - **node2:** https://192.168.0.9:8006
   - **node3:** https://192.168.0.6:8006
   - **Login:** `root`
   - **Password:** [Your root password]

### Step 1.3: Update Proxmox

```bash
# SSH to Proxmox node
ssh root@192.168.0.4

# Update system
apt update
apt upgrade -y
```

### Step 1.3.1: Configure Additional Network Bridges

**⚠️ IMPORTANT:** Your setup uses **3 physical NICs per node** for network separation:

**Network Architecture:**
- **vmbr0 (enp2s0):** Main network - Proxmox management + Ceph cluster traffic
- **vmbr2 (enp4s0):** Internal isolated network
- **vmbr1 (enp5s0):** VM-only network (no host IP)

**Configure on ALL nodes (node1, node2, node3):**

```bash
# Edit network configuration
nano /etc/network/interfaces
```

**Add the following configuration:**

```
# Primary network (already configured during install)
auto vmbr0
iface vmbr0 inet static
	address 192.168.0.4/8  # Change per node: .4, .9, .6
	gateway 192.168.0.1
	bridge-ports enp2s0
	bridge-stp off
	bridge-fd 0
	mtu 9000

# Internal isolated network
auto vmbr2
iface vmbr2 inet static
	address 10.10.10.1/24  # Change per node: .1, .2, .3
	bridge-ports enp4s0
	bridge-stp off
	bridge-fd 0
	mtu 9000

# VM-only network (no host IP)
auto vmbr1
iface vmbr1 inet manual
	bridge-ports enp5s0
	bridge-stp off
	bridge-fd 0

# Set MTU on all physical interfaces
iface enp2s0 inet manual
	mtu 9000

iface enp3s0 inet manual
	mtu 9000

iface enp4s0 inet manual
	mtu 9000

iface enp5s0 inet manual
```

**Node-specific IP addresses:**
- **node1:** vmbr0: 192.168.0.4/8, vmbr2: 10.10.10.1/24
- **node2:** vmbr0: 192.168.0.9/8, vmbr2: 10.10.10.2/24
- **node3:** vmbr0: 192.168.0.6/8, vmbr2: 10.10.10.3/24

**Apply network configuration:**

```bash
# Restart networking
systemctl restart networking

# Or reboot if networking restart doesn't work cleanly
reboot

# After reboot, verify all bridges
ip addr show
ip link show | grep mtu

# Expected: vmbr0 and vmbr2 should show mtu 9000
```

**Verify on all nodes:**

```bash
# Check all bridges are up
ip addr show vmbr0
ip addr show vmbr1
ip addr show vmbr2

# Check MTU is 9000 on vmbr0 and vmbr2
ip link show vmbr0 | grep mtu
ip link show vmbr2 | grep mtu
```

### Step 1.3.2: Install Claude Code (Optional but Recommended)

**💡 WHY:** Claude Code can read and execute the rest of this recovery guide, making the process much easier!

```bash
# Install Claude Code CLI
curl -fsSL https://raw.githubusercontent.com/anthropics/claude-code/main/install.sh | sh

# Or manual installation:
# Download from: https://github.com/anthropics/claude-code/releases

# Verify installation
claude --version

# Set your API key (get from: https://console.anthropic.com/)
export ANTHROPIC_API_KEY="your-api-key-here"

# Add to shell profile for persistence
echo 'export ANTHROPIC_API_KEY="your-api-key-here"' >> ~/.bashrc
source ~/.bashrc

# Test Claude Code
claude "hello, can you help me with disaster recovery?"
```

**Using Claude Code for Recovery:**

```bash
# Clone this disaster recovery repo
cd /root/recovery
git clone https://github.com/jkempson/disaster-recovery.git

# Let Claude Code read and help execute the recovery steps
claude "Read the file disaster-recovery/COMPLETE-LOSS-RECOVERY.md and help me execute Phase 4: Ceph cluster setup"

# Claude Code can now:
# - Read all the recovery documentation
# - Execute commands step-by-step
# - Handle errors and troubleshooting
# - Verify each step before proceeding
```

**Benefits of using Claude Code:**
- ✅ Reads and understands the entire recovery documentation
- ✅ Executes commands safely with verification
- ✅ Handles errors and provides troubleshooting
- ✅ Tracks progress through the recovery steps
- ✅ Can adapt steps based on your specific situation

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

### Step 4.2: Create Ceph Cluster (Manual - Required First)

⚠️ **IMPORTANT:** You must create a NEW Ceph cluster before Terraform can deploy Talos nodes that use it.

This section covers setting up a 3-node Ceph cluster based on your original configuration:
- **3 Proxmox nodes:** node1 (192.168.0.4), node2 (192.168.0.9), node3 (192.168.0.6)
- **3 monitors**, **3 managers**, **3 MDS**, **3 OSDs** (one per node)
- **CephFS filesystem** for shared storage

#### Prerequisites for Ceph Setup

**You need 3 Proxmox nodes installed:**
- All nodes must be in the same network (192.168.0.0/24)
- All nodes must be able to ping each other
- Each node needs at least one dedicated disk/partition for Ceph OSD

**If you only have ONE physical server:**
- Option A: Deploy Ceph single-node (simplified, less redundant)
- Option B: Skip Ceph initially, use local storage, add later
- Option C: Order 2 more mini PCs for proper 3-node cluster

#### Option 1: Three-Node Ceph Cluster (Original Setup)

**Step 4.2.1: Install Proxmox on All 3 Nodes**

```bash
# Repeat Phase 1 steps for each node:
# - node1: 192.168.0.4 (primary)
# - node2: 192.168.0.9
# - node3: 192.168.0.6

# After installing Proxmox on all 3 nodes, create Proxmox cluster from node1:
ssh root@192.168.0.4
pvecm create homelab-cluster

# Join node2 to cluster
ssh root@192.168.0.9
pvecm add 192.168.0.4

# Join node3 to cluster
ssh root@192.168.0.6
pvecm add 192.168.0.4

# Verify cluster status
pvecm status
pvecm nodes
```

**Step 4.2.2: Configure Network (Jumbo Frames)**

```bash
# On ALL nodes (node1, node2, node3):
# Edit /etc/network/interfaces to set MTU 9000

cat >> /etc/network/interfaces << 'EOF'

# Jumbo frames for Ceph
post-up /sbin/ip link set dev vmbr0 mtu 9000
EOF

# Apply MTU change
ip link set dev vmbr0 mtu 9000

# Verify
ip link show vmbr0 | grep mtu
# Expected: mtu 9000
```

**Step 4.2.3: Install Ceph Packages on All Nodes**

```bash
# On ALL nodes (node1, node2, node3):
apt update
apt install -y ceph ceph-mds
```

**Step 4.2.4: Initialize Ceph Cluster (from node1)**

```bash
# Run on node1 ONLY:
cd /root/recovery

# Generate unique cluster ID
CLUSTER_ID=$(uuidgen)
echo "Cluster ID: $CLUSTER_ID"

# Create ceph.conf
cat > /etc/ceph/ceph.conf << EOF
[global]
fsid = $CLUSTER_ID
mon initial members = node1, node2, node3
mon host = 192.168.0.4, 192.168.0.9, 192.168.0.6
public network = 192.168.0.0/24
auth cluster required = cephx
auth service required = cephx
auth client required = cephx
osd pool default size = 3
osd pool default min size = 2
osd pool default pg num = 32
osd pool default pgp num = 32
EOF

# Create monitor keyring
ceph-authtool --create-keyring /tmp/ceph.mon.keyring --gen-key -n mon. --cap mon 'allow *'

# Create admin keyring
ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring --gen-key -n client.admin --cap mon 'allow *' --cap osd 'allow *' --cap mds 'allow *' --cap mgr 'allow *'

# Create bootstrap-osd keyring
ceph-authtool --create-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring --gen-key -n client.bootstrap-osd --cap mon 'profile bootstrap-osd' --cap mgr 'allow r'

# Add client.admin and bootstrap-osd keys to mon keyring
ceph-authtool /tmp/ceph.mon.keyring --import-keyring /etc/ceph/ceph.client.admin.keyring
ceph-authtool /tmp/ceph.mon.keyring --import-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring

# Generate monitor map
monmaptool --create --add node1 192.168.0.4 --add node2 192.168.0.9 --add node3 192.168.0.6 --fsid $CLUSTER_ID /tmp/monmap

# Create monitor data directory
mkdir -p /var/lib/ceph/mon/ceph-node1
chown ceph:ceph /var/lib/ceph/mon/ceph-node1

# Populate monitor daemon
sudo -u ceph ceph-mon --mkfs -i node1 --monmap /tmp/monmap --keyring /tmp/ceph.mon.keyring

# Enable and start monitor
systemctl enable ceph-mon@node1
systemctl start ceph-mon@node1

# Check monitor status
ceph -s
```

**Step 4.2.5: Add Monitors on node2 and node3**

```bash
# Copy config and keyrings to node2 and node3
for node in node2 node3; do
  ssh root@$node "mkdir -p /etc/ceph /var/lib/ceph/mon /var/lib/ceph/bootstrap-osd"
  scp /etc/ceph/ceph.conf root@$node:/etc/ceph/
  scp /etc/ceph/ceph.client.admin.keyring root@$node:/etc/ceph/
  scp /tmp/ceph.mon.keyring root@$node:/tmp/
  scp /tmp/monmap root@$node:/tmp/
  scp /var/lib/ceph/bootstrap-osd/ceph.keyring root@$node:/var/lib/ceph/bootstrap-osd/
done

# On node2:
ssh root@192.168.0.9 << 'EOF'
mkdir -p /var/lib/ceph/mon/ceph-node2
chown ceph:ceph /var/lib/ceph/mon/ceph-node2
sudo -u ceph ceph-mon --mkfs -i node2 --monmap /tmp/monmap --keyring /tmp/ceph.mon.keyring
systemctl enable ceph-mon@node2
systemctl start ceph-mon@node2
EOF

# On node3:
ssh root@192.168.0.6 << 'EOF'
mkdir -p /var/lib/ceph/mon/ceph-node3
chown ceph:ceph /var/lib/ceph/mon/ceph-node3
sudo -u ceph ceph-mon --mkfs -i node3 --monmap /tmp/monmap --keyring /tmp/ceph.mon.keyring
systemctl enable ceph-mon@node3
systemctl start ceph-mon@node3
EOF

# Verify all monitors are up (run on node1)
ceph -s
# Expected: mon: 3 daemons, quorum node1,node2,node3
```

**Step 4.2.6: Deploy Managers**

```bash
# On node1, node2, node3:
for node in node1 node2 node3; do
  ssh root@$node << 'EOF'
ceph auth get-or-create mgr.$(hostname) mon 'allow profile mgr' osd 'allow *' mds 'allow *' > /var/lib/ceph/mgr/ceph-$(hostname)/keyring
systemctl enable ceph-mgr@$(hostname)
systemctl start ceph-mgr@$(hostname)
EOF
done

# Verify (run on node1)
ceph -s
# Expected: mgr: <active>, standbys: <standby1>, <standby2>
```

**Step 4.2.7: Create OSDs**

```bash
# On each node, identify the disk for OSD
# Find available disks:
ssh root@192.168.0.4 lsblk
ssh root@192.168.0.9 lsblk
ssh root@192.168.0.6 lsblk

# Assuming /dev/sdb on each node (ADJUST TO YOUR DISKS!)
# ⚠️ WARNING: This will ERASE the disk!

# Create OSD on node1
ssh root@192.168.0.4 "ceph-volume lvm create --data /dev/sdb"

# Create OSD on node2
ssh root@192.168.0.9 "ceph-volume lvm create --data /dev/sdb"

# Create OSD on node3
ssh root@192.168.0.6 "ceph-volume lvm create --data /dev/sdb"

# Verify OSDs (run on node1)
ceph osd tree
ceph osd status
# Expected: 3 osds: 3 up, 3 in
```

**Step 4.2.8: Create CephFS (Metadata Servers and Filesystem)**

```bash
# Create MDS directories on all nodes
for node in node1 node2 node3; do
  ssh root@$node "mkdir -p /var/lib/ceph/mds/ceph-$node"
  ssh root@$node "ceph auth get-or-create mds.$node mon 'profile mds' mgr 'profile mds' mds 'allow *' osd 'allow *' > /var/lib/ceph/mds/ceph-$node/keyring"
  ssh root@$node "chown -R ceph:ceph /var/lib/ceph/mds/ceph-$node"
  ssh root@$node "systemctl enable ceph-mds@$node"
  ssh root@$node "systemctl start ceph-mds@$node"
done

# Create CephFS pools
ceph osd pool create cephfs_data 32
ceph osd pool create cephfs_metadata 32

# Create CephFS filesystem
ceph fs new cephfs cephfs_metadata cephfs_data

# Verify
ceph fs status
ceph mds stat
# Expected: 1 up, 2 standby
```

**Step 4.2.9: Create Additional Pools**

```bash
# Create RBD pool for VM disks
ceph osd pool create ceph_pool_1 32

# Initialize RBD pool
rbd pool init ceph_pool_1

# Verify pools
ceph osd pool ls
ceph df
```

**Step 4.2.10: Mount CephFS on Proxmox Nodes**

```bash
# Extract Ceph admin key
CEPH_KEY=$(grep key /etc/ceph/ceph.client.admin.keyring | awk '{print $3}')
echo "Ceph admin key: $CEPH_KEY"

# Save key for Terraform
echo "CEPH_ADMIN_KEY=$CEPH_KEY" >> /root/recovery/.envrc

# Mount CephFS on all nodes
for node in node1 node2 node3; do
  ssh root@$node "mkdir -p /mnt/pve/cephfs"
  ssh root@$node "mount -t ceph 192.168.0.4,192.168.0.9,192.168.0.6:/ /mnt/pve/cephfs -o name=admin,secret=$CEPH_KEY"
done

# Verify mounts
df -h /mnt/pve/cephfs

# Add to /etc/fstab on all nodes for persistence
for node in node1 node2 node3; do
  ssh root@$node "echo '192.168.0.4,192.168.0.9,192.168.0.6:/     /mnt/pve/cephfs     ceph    name=admin,secret=$CEPH_KEY,noatime,_netdev    0       2' >> /etc/fstab"
done
```

**Step 4.2.11: Verify Ceph Cluster Health**

```bash
# Check cluster status
ceph -s
ceph health detail

# Expected output:
#   cluster:
#     id:     <uuid>
#     health: HEALTH_OK
#
#   services:
#     mon: 3 daemons, quorum node1,node2,node3
#     mgr: <active>(active), standbys: <standby1>, <standby2>
#     mds: cephfs:1 {0=<active>} 2 up:standby
#     osd: 3 osds: 3 up, 3 in
#
#   data:
#     pools:   4 pools, 128 pgs
#     objects: 0 objects, 0 B
#     usage:   <usage> / <total>
#     pgs:     128 active+clean

# Test CephFS write
echo "test" > /mnt/pve/cephfs/recovery-test.txt
cat /mnt/pve/cephfs/recovery-test.txt
rm /mnt/pve/cephfs/recovery-test.txt
```

**Checkpoint:** ✅ Three-node Ceph cluster operational with CephFS mounted

---

#### Option 2: Single-Node Ceph (Simplified, Less Redundant)

If you only have ONE physical server and can't wait for additional hardware:

```bash
# Install Ceph
apt install -y ceph ceph-mds

# Deploy with Proxmox GUI
# 1. In Proxmox web UI, go to Datacenter > Ceph
# 2. Click "Install Ceph"
# 3. Follow wizard:
#    - Select single-node configuration
#    - Choose disk for OSD
#    - Create CephFS

# Or use CLI:
pveceph install
pveceph init --network 192.168.0.0/24
pveceph mon create
pveceph mgr create
pveceph osd create /dev/sdb  # Adjust disk

# Create pools and CephFS
ceph osd pool create cephfs_data 32
ceph osd pool create cephfs_metadata 32
ceph fs new cephfs cephfs_metadata cephfs_data

# Mount CephFS
mkdir -p /mnt/pve/cephfs
CEPH_KEY=$(grep key /etc/ceph/ceph.client.admin.keyring | awk '{print $3}')
mount -t ceph 192.168.0.4:/ /mnt/pve/cephfs -o name=admin,secret=$CEPH_KEY

# Note: Single-node Ceph has NO redundancy
# Replication will show HEALTH_WARN because min_size=2 cannot be met
# Set size=1 for single-node:
ceph osd pool set cephfs_data size 1 --yes-i-really-mean-it
ceph osd pool set cephfs_metadata size 1 --yes-i-really-mean-it
```

**Checkpoint:** ✅ Single-node Ceph operational (no redundancy)

---

#### Option 3: Skip Ceph Initially

If Ceph setup is too complex for immediate recovery:

```bash
# Edit terraform.tfvars to use local storage instead of Ceph
# In /root/recovery/terraform.tfvars, find and modify:
# - Comment out ceph_* variables
# - Set storage backend to local

# Deploy cluster with local storage
# Add Ceph later once cluster is stable and you have more time
```

**Note:** You'll need to migrate data to Ceph later if you choose this option.

### Step 4.3: Review Terraform Plan

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

### Step 4.4: Apply Terraform Configuration

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
