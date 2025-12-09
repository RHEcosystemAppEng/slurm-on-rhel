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
```

**Important**: Log out and log back in (or restart) for snap paths to work.

### Step 3: Install Munge (Required for Authentication)

```bash
# Install Munge
sudo dnf install -y munge munge-libs munge-devel

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
sudo snap install slurm --classic

# Install Slurm server daemons (from EPEL)
sudo dnf install -y slurm-slurmctld slurm-slurmd
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
# Get CPU count
CPU_COUNT=$(nproc)

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

NodeName=localhost NodeAddr=127.0.0.1 CPUs=${CPU_COUNT} State=UNKNOWN
PartitionName=compute Nodes=localhost Default=YES MaxTime=INFINITE State=UP
EOF

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

# Check node status
sinfo
```

## Next Steps

For detailed examples, advanced configuration, and comprehensive troubleshooting, see [docs/DETAILED_GUIDE.md](docs/DETAILED_GUIDE.md)

## Reference

- Snap installation: [https://snapcraft.io/install/slurm/rhel](https://snapcraft.io/install/slurm/rhel)
- Slurm documentation: [https://slurm.schedmd.com/](https://slurm.schedmd.com/)
