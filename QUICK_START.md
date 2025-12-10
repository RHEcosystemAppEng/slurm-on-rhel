# Quick Start Guide - Slurm on RHEL 9

Complete setup guide from installation to testing. Follow this if you already have RHEL 9 installed in a VM.

## Prerequisites

- RHEL 9 installed in VM
- Internet connectivity
- Root or sudo access

## Installation (5 Steps)

### Step 1: Install EPEL Repository

```bash
sudo dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
```

### Step 2: Install and Enable snapd

```bash
# Install snapd
sudo dnf install -y snapd

# Enable snapd service
sudo systemctl enable --now snapd.socket
sudo ln -s /var/lib/snapd/snap /snap

# Wait a few seconds for snapd to initialize
sleep 5

# Verify snapd is ready
snap version
```

**Important**: 
- If you get "too early for operation" error, wait 10-30 seconds and try again
- **Log out and log back in** (or restart the VM) for snap paths to work properly
- After login, verify: `snap version` should work without errors

### Step 3: Install Munge (Required for Authentication)

**Note**: If you encounter SSL certificate errors (OCSP response expired), see [Troubleshooting](#troubleshooting) section below.

```bash
# Install Munge
sudo dnf install -y munge munge-libs

# Note: munge-devel is optional (only needed for development)
# If available, you can install it: sudo dnf install -y munge-devel

# Generate Munge key
sudo /usr/sbin/create-munge-key

# Set permissions
sudo chown munge:munge /etc/munge/munge.key
sudo chmod 600 /etc/munge/munge.key

# Enable and start Munge
sudo systemctl enable --now munge

# Test Munge
munge -n | unmunge
```

### Step 4: Install Slurm

```bash
# Install Slurm client tools (via snap)
# Note: If you get "too early for operation" error, wait 10-30 seconds and try again
sudo snap install slurm --classic

# Verify client tools are installed
scontrol --version

# Install Slurm server daemons (from EPEL)
sudo dnf install -y slurm-slurmctld slurm-slurmd

# Verify daemons are installed
slurmctld --version
slurmd --version
```

### Step 5: Verify Installation

```bash
# Check client tools
scontrol --version

# Check daemons
slurmctld --version
slurmd --version
```

## Configuration

### Create Slurm Configuration

```bash
# Get CPU count and memory
CPU_COUNT=$(nproc)
MEMORY_MB=$(free -m | awk '/^Mem:/{print $2}')

# Display values for verification
echo "CPU count: ${CPU_COUNT}"
echo "Memory: ${MEMORY_MB}MB"

# Create configuration directory
sudo mkdir -p /etc/slurm

# Create slurm.conf
sudo tee /etc/slurm/slurm.conf > /dev/null << EOF
ClusterName=slurm-cluster
ControlMachine=localhost
SlurmUser=slurm
SlurmctldPort=6817
SlurmdPort=6818
AuthType=auth/munge
StateSaveLocation=/var/spool/slurmctld
SlurmdSpoolDir=/var/spool/slurmd
SwitchType=switch/none
MpiDefault=none
SlurmctldPidFile=/var/run/slurmctld.pid
SlurmdPidFile=/var/run/slurmd.pid
ProctrackType=proctrack/linuxproc
ReturnToService=1
SlurmctldTimeout=300
SlurmdTimeout=300
InactiveLimit=0
MinJobAge=300
KillWait=30
Waittime=0
SchedulerType=sched/backfill
SelectType=select/linear
SlurmctldLogFile=/var/log/slurm/slurmctld.log
SlurmdLogFile=/var/log/slurm/slurmd.log
NodeName=localhost NodeAddr=127.0.0.1 CPUs=${CPU_COUNT} RealMemory=${MEMORY_MB} State=UNKNOWN
PartitionName=compute Nodes=localhost Default=YES MaxTime=INFINITE State=UP
EOF

# Verify configuration was created correctly
echo "Verifying slurm.conf..."
grep RealMemory /etc/slurm/slurm.conf
echo "Configuration created successfully!"

# Create required directories
sudo mkdir -p /var/spool/slurmctld /var/spool/slurmd /var/log/slurm

# Create Slurm user (if not exists)
sudo groupadd -r slurm 2>/dev/null || true
sudo useradd -r -g slurm -d /var/lib/slurm -s /bin/bash slurm 2>/dev/null || true

# Set ownership
sudo chown slurm:slurm /var/spool/slurmctld
sudo chown slurm:slurm /var/spool/slurmd
sudo chown slurm:slurm /var/log/slurm
```

### Start Slurm Services

After installing `slurm-slurmctld` and `slurm-slurmd` packages, the systemd service files should be automatically created. Simply start the services:

```bash
# Start services
sudo systemctl enable --now slurmctld
sudo systemctl enable --now slurmd

# Check status
sudo systemctl status slurmctld
sudo systemctl status slurmd

# Verify Slurm recognizes the node and memory
sinfo -o "%N %m"
scontrol show node localhost | grep -i memory
```

**Service Management:**

```bash
# Stop services
sudo systemctl stop slurmctld slurmd

# Start services
sudo systemctl start slurmctld slurmd

# Restart services
sudo systemctl restart slurmctld slurmd

# Disable auto-start
sudo systemctl disable slurmctld slurmd
```

If service files don't exist, they will be created automatically when you install the packages.

## Testing

### Test 1: Simple Job

```bash
srun hostname
```

### Test 2: Batch Job

```bash
cat > test.sh << 'EOF'
#!/bin/bash
#SBATCH --job-name=test
#SBATCH --output=test.out
#SBATCH --time=00:01:00

echo "Hello from Slurm!"
hostname
date
EOF

sbatch test.sh
squeue
cat test.out
```

## Troubleshooting

### Snap "Too Early for Operation" Error

If you get `error: too early for operation, device not yet seeded`:

```bash
# Fix 1: Wait and check snapd status
sudo systemctl status snapd.socket
sudo systemctl status snapd

# Fix 2: Restart snapd services
sudo systemctl restart snapd.socket
sudo systemctl restart snapd
sleep 10

# Fix 3: Verify snapd is ready
snap version

# Fix 4: If still failing, log out and log back in (or restart VM)
# Then try: sudo snap install slurm --classic
```

**Why this happens**: snapd needs time to initialize after installation. The "seeding" process can take 10-30 seconds. If you try too quickly, it fails. Waiting or restarting snapd usually fixes it.

### SSL Certificate Errors (OCSP Response Expired)

If you get SSL certificate errors when installing packages:

```bash
# Fix 1: Update system time (most common cause)
sudo chronyd -q
# Or manually sync time:
sudo timedatectl set-ntp true

# Fix 2: Update certificate store
sudo update-ca-trust

# Fix 3: Temporarily disable SSL verification (not recommended, use only if above don't work)
sudo dnf install -y --setopt=sslverify=false munge munge-libs munge-devel

# Fix 4: Update subscription and certificates
sudo subscription-manager refresh
sudo subscription-manager repos --enable=rhel-9-for-aarch64-appstream-rpms
sudo subscription-manager repos --enable=rhel-9-for-aarch64-baseos-rpms
```

### Slurm Service Issues

```bash
# Check services
sudo systemctl status slurmctld
sudo systemctl status slurmd

# Check logs (most important!)
sudo tail -20 /var/log/slurm/slurmctld.log
sudo tail -20 /var/log/slurm/slurmd.log

# Check systemd logs
sudo journalctl -u slurmctld -n 20
sudo journalctl -u slurmd -n 20

# Check configuration
sudo scontrol show config

# Verify configuration file syntax
sudo slurmctld -C

# Check node status and memory
sinfo
sinfo -o "%N %m"
scontrol show node localhost | grep -i memory
```

### Memory Configuration Issues

If memory shows as 1MB or N/A in `sinfo`:

```bash
# Check actual system memory
free -m

# Check what's in slurm.conf
grep RealMemory /etc/slurm/slurm.conf

# Fix: Update RealMemory to actual system memory
MEMORY_MB=$(free -m | awk '/^Mem:/{print $2}')
echo "Updating RealMemory to ${MEMORY_MB}MB"

# Update RealMemory value
sudo sed -i "s/RealMemory=[0-9]*/RealMemory=${MEMORY_MB}/" /etc/slurm/slurm.conf

# Or if RealMemory doesn't exist, add it after CPUs
CPU_COUNT=$(nproc)
sudo sed -i "s/^NodeName=localhost.*/NodeName=localhost NodeAddr=127.0.0.1 CPUs=${CPU_COUNT} RealMemory=${MEMORY_MB} State=UNKNOWN/" /etc/slurm/slurm.conf

# Verify the change
grep RealMemory /etc/slurm/slurm.conf

# Restart services
sudo systemctl restart slurmctld slurmd

# Verify it's now correct
sinfo -o "%N %m"
scontrol show node localhost | grep -i memory
```

## Next Steps

For detailed examples, advanced configuration, and comprehensive troubleshooting, see [docs/DETAILED_GUIDE.md](docs/DETAILED_GUIDE.md)

## Reference

- Snap installation: [https://snapcraft.io/install/slurm/rhel](https://snapcraft.io/install/slurm/rhel)
- Slurm documentation: [https://slurm.schedmd.com/](https://slurm.schedmd.com/)
