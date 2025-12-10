# Slurm on RHEL - Detailed Guide

Complete guide with detailed installation, configuration, examples, testing, and troubleshooting for Slurm on RHEL 9.

## Table of Contents

1. [Installation Details](#installation-details)
2. [Configuration Details](#configuration-details)
3. [Advanced Configuration](#advanced-configuration)
4. [Adding Compute Nodes](#adding-compute-nodes-to-existing-cluster)
5. [Detailed Examples](#detailed-examples)
6. [Testing and Validation](#testing-and-validation)
7. [Troubleshooting](#troubleshooting)

## Installation Details

### Prerequisites

- RHEL 9 (ARM64 or x86_64)
- Root or sudo access
- Internet connectivity
- Minimum 2GB RAM, 2 CPU cores

### Step-by-Step Installation

#### 1. Install EPEL Repository

```bash
# For RHEL 9
sudo dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm

# Verify EPEL is enabled
sudo dnf repolist | grep epel
```

#### 2. Install and Enable snapd

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
- **Log out and log back in** (or restart) for snap paths to work properly
- After login, verify: `snap version` should work without errors

#### 3. Install Munge (Authentication)

Munge is required for secure communication between Slurm components.

```bash
# Install Munge packages
sudo dnf install -y munge munge-libs

# Note: munge-devel is optional (only needed for development)
# If available in your repositories, you can install it:
# sudo dnf install -y munge-devel

# Generate Munge key
sudo /usr/sbin/create-munge-key

# Set proper permissions
sudo chown munge:munge /etc/munge/munge.key
sudo chmod 600 /etc/munge/munge.key

# Enable and start Munge
sudo systemctl enable --now munge

# Test Munge authentication
munge -n | unmunge
```

**Expected output**: Should show authentication success message.

#### 4. Install Slurm

```bash
# Install Slurm client tools (via snap)
sudo snap install slurm --classic

# Install Slurm server daemons (from EPEL)
sudo dnf install -y slurm-slurmctld slurm-slurmd

# Verify installation
scontrol --version
slurmctld --version
slurmd --version
```

### Alternative: Build from Source

If EPEL packages don't work, build from source. See [Slurm documentation](https://slurm.schedmd.com/quickstart_admin.html) for detailed instructions.

## Configuration Details

### Basic Configuration

#### 1. Create Required Directories

```bash
# Create directories
sudo mkdir -p /var/spool/slurmctld
sudo mkdir -p /var/spool/slurmd
sudo mkdir -p /var/log/slurm
sudo mkdir -p /etc/slurm

# Create Slurm user
sudo groupadd -r slurm
sudo useradd -r -g slurm -d /var/lib/slurm -s /bin/bash slurm

# Set ownership
sudo chown slurm:slurm /var/spool/slurmctld
sudo chown slurm:slurm /var/spool/slurmd
sudo chown slurm:slurm /var/log/slurm
```

#### 2. Create slurm.conf

For a single-node setup (controller and compute on same machine):

```bash
# Get system information
CPU_COUNT=$(nproc)
MEMORY_MB=$(free -m | awk '/^Mem:/{print $2}')
HOSTNAME=$(hostname -s)

# Display values for verification
echo "CPU count: ${CPU_COUNT}"
echo "Memory: ${MEMORY_MB}MB"

# Create configuration
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

# Verify configuration
echo "Verifying slurm.conf..."
grep RealMemory /etc/slurm/slurm.conf
echo "CPU count: ${CPU_COUNT}, Memory: ${MEMORY_MB}MB"
```

#### 3. Start Services

```bash
# Start services
sudo systemctl enable --now slurmctld
sudo systemctl enable --now slurmd

# Verify services are running
sudo systemctl status slurmctld
sudo systemctl status slurmd

# Verify Slurm recognizes node and memory correctly
sinfo -o "%N %m"
scontrol show node localhost | grep -i memory
```

**Important**: After starting services, verify that `sinfo` shows the correct memory value (not 1MB or N/A). If memory shows incorrectly, see [Troubleshooting](#troubleshooting) section.

#### 4. Service Management

```bash
# Start/Stop/Restart
sudo systemctl start|stop|restart slurmctld slurmd

# Enable/Disable auto-start
sudo systemctl enable|disable slurmctld slurmd

# Check status
sudo systemctl status slurmctld slurmd

# Before stopping, cancel jobs: scancel -u $USER
```

### Configuration Parameters Explained

Key parameters: `ClusterName`, `ControlMachine`, `SlurmUser`, `AuthType=auth/munge`, `NodeName` (with `CPUs` and `RealMemory`), `PartitionName`.

**Important**: Nodes are CPU-only by default. For GPU support, add `Gres=gpu:X` to NodeName. See [Adding GPU Nodes](#adding-gpu-nodes).

## Advanced Configuration

### Multi-Node Setup

For a cluster with separate controller and compute nodes:

```bash
# On controller, create slurm.conf with all nodes
# NodeName=controller NodeAddr=192.168.1.10 CPUs=8 RealMemory=16384 State=UNKNOWN
# NodeName=compute1 NodeAddr=192.168.1.11 CPUs=4 RealMemory=8192 State=UNKNOWN
# NodeName=compute2 NodeAddr=192.168.1.12 CPUs=4 RealMemory=8192 State=UNKNOWN
# PartitionName=compute Nodes=controller,compute1,compute2 Default=YES MaxTime=INFINITE State=UP

# Copy slurm.conf and munge.key to compute nodes
sudo scp /etc/slurm/slurm.conf root@compute1:/etc/slurm/
sudo scp /etc/munge/munge.key root@compute1:/etc/munge/

# On compute nodes: Set munge key permissions and start services
sudo chown munge:munge /etc/munge/munge.key
sudo chmod 600 /etc/munge/munge.key
sudo systemctl restart munge
sudo systemctl enable --now slurmd
```

### Adding Compute Nodes to Existing Cluster

**Quick steps:**
1. On new node: Install EPEL, snapd, Munge, Slurm (same as initial setup)
2. On controller: Add node to slurm.conf: `NodeName=compute3 NodeAddr=192.168.1.13 CPUs=4 RealMemory=<MB> State=UNKNOWN`
3. Copy slurm.conf and munge.key to new node
4. On new node: Set munge key permissions, start munge and slurmd
5. On controller: Restart slurmctld, verify with `sinfo`

### Adding GPU Nodes

**Steps:**
1. Install Slurm on GPU node (same as compute node)
2. In slurm.conf, add GPU node with `Gres=gpu:X`: `NodeName=gpu-node1 NodeAddr=192.168.1.20 CPUs=8 RealMemory=<MB> Gres=gpu:2 State=UNKNOWN`
3. Create `/etc/slurm/gres.conf` on all nodes: `Name=gpu File=/dev/nvidia[0-1]`
4. Copy slurm.conf, gres.conf, and munge.key to GPU node
5. Start services and verify: `sinfo -o "%N %G"`

### Partition Configuration

Create multiple partitions for different job types:

```bash
# Add to slurm.conf
PartitionName=short Nodes=ALL Default=NO MaxTime=01:00:00 State=UP
PartitionName=long Nodes=ALL Default=NO MaxTime=7-00:00:00 State=UP
PartitionName=compute Nodes=ALL Default=YES MaxTime=INFINITE State=UP
```

## Detailed Examples

### Example 1: Simple Interactive Job

```bash
# Run a simple command
srun hostname

# Run with specific resources
srun -N 1 -n 2 --time=5:00 hostname

# Run with memory limit
srun --mem=1G hostname

# Run on specific partition
srun -p compute hostname
```

### Example 2: Basic Batch Job

```bash
# Create job script
cat > hello.sh << 'EOF'
#!/bin/bash
#SBATCH --job-name=hello
#SBATCH --output=hello.out
#SBATCH --error=hello.err
#SBATCH --time=00:01:00
#SBATCH --nodes=1
#SBATCH --ntasks=1

echo "Hello from Slurm!"
echo "Job ID: $SLURM_JOB_ID"
echo "Node: $SLURM_NODELIST"
echo "CPU count: $SLURM_CPUS_ON_NODE"
date
EOF

# Submit job
sbatch hello.sh

# Check status
squeue

# View output
cat hello.out
```

### Example 3: CPU-Intensive Job

```bash
cat > cpu_intensive.sh << 'EOF'
#!/bin/bash
#SBATCH --job-name=cpu-test
#SBATCH --output=cpu_test.out
#SBATCH --time=00:05:00
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=2

echo "Starting CPU-intensive task..."
echo "Using $SLURM_CPUS_ON_NODE CPUs"
echo "Start time: $(date)"

# CPU-intensive work using a faster method
echo "Running calculations..."
# Use a mathematical calculation loop (faster than echo)
result=0
for i in {1..1000000}; do
    result=$((result + i * 2))
done

echo "Result: $result"
echo "Calculations completed!"
echo "End time: $(date)"
echo "Task completed!"
EOF

# Submit the job
sbatch cpu_intensive.sh

# Check job status
squeue

# Wait for job to complete, then check output
# Output file: cpu_test.out (specified in #SBATCH --output)
cat cpu_test.out

# Or check job output in real-time (if job is still running)
tail -f cpu_test.out
```

### Example 4: Memory-Intensive Job

**Note**: For memory jobs to work, your `slurm.conf` must include `RealMemory` in the NodeName definition. See [Configuration Details](#configuration-details) section.

#### Step 1: Check Available Memory

Before running memory jobs, check the available memory:

```bash
# Check actual system memory
free -h

# Check memory configured in Slurm
sinfo -o "%N %m"  # Shows node and memory in MB
scontrol show node localhost | grep -i memory

# Check if RealMemory is configured in slurm.conf
grep RealMemory /etc/slurm/slurm.conf
```

**Understanding the output:**
- `free -h` shows actual system memory (total, used, available)
- `sinfo -o "%N %m"` shows what Slurm knows about memory
- If Slurm shows 'N/A' or no memory, you need to add `RealMemory` to `slurm.conf`

#### Step 2: Configure Memory in Slurm (if needed or incorrect)

If memory shows incorrect value (like 1MB instead of actual memory):

```bash
# Get system memory and update slurm.conf
MEMORY_MB=$(free -m | awk '/^Mem:/{print $2}')
echo "Updating RealMemory to ${MEMORY_MB}MB"

# Update RealMemory value
sudo sed -i "s/RealMemory=[0-9]*/RealMemory=${MEMORY_MB}/" /etc/slurm/slurm.conf

# Or if RealMemory doesn't exist, add it after CPUs
sudo sed -i "s/CPUs=\([0-9]*\)/CPUs=\1 RealMemory=${MEMORY_MB}/" /etc/slurm/slurm.conf

# Restart services and verify
sudo systemctl restart slurmctld slurmd
sinfo -o "%N %m"
scontrol show node localhost | grep -i memory
```

#### Step 3: Create and Submit Memory Job

```bash
cat > memory_test.sh << 'EOF'
#!/bin/bash
#SBATCH --job-name=mem-test
#SBATCH --output=mem_test.out
#SBATCH --time=00:02:00
#SBATCH --mem=512M
#SBATCH --nodes=1
#SBATCH --ntasks=1

echo "=== Memory Job Information ==="
echo "Requested memory: $SLURM_MEM_PER_NODE"
echo "Job started at: $(date)"
echo ""

echo "=== Allocating memory... ==="
# Allocate 400MB of memory (less than requested 512M)
python3 << PYTHON
import array
import time
data = array.array('B', [0]) * (400 * 1024 * 1024)
print(f"Allocated {len(data) / (1024*1024):.2f} MB")
print("Holding memory for 5 seconds...")
time.sleep(5)
print("Releasing memory...")
PYTHON

echo ""
echo "Memory test completed at: $(date)"
EOF
```

#### Step 4: Verify Memory Usage

**Before submitting the job:**

```bash
# Check system memory before job
echo "=== Memory BEFORE job ==="
free -h
echo ""
echo "Memory available in Slurm:"
sinfo -o "%N %m"
```

**Submit the job:**

```bash
sbatch memory_test.sh
```

**While job is running (in another terminal):**

```bash
# Monitor memory in real-time
watch -n 1 'free -h'

# Or check once
free -h
```

**After job completes:**

```bash
# Check memory after job
echo "=== Memory AFTER job ==="
free -h
echo ""
echo "Job output:"
cat mem_test.out
```

**Complete example with timing:**

```bash
# Step 1: Check memory before
echo "=== BEFORE ==="
free -h | grep -E "Mem|Swap"
sinfo -o "%N %m"

# Step 2: Submit job
JOBID=$(sbatch memory_test.sh | awk '{print $4}')
echo "Job submitted: $JOBID"

# Step 3: Wait for job to start
sleep 3
echo ""
echo "=== DURING (job running) ==="
free -h | grep -E "Mem|Swap"

# Step 4: Wait for job to complete
squeue -j $JOBID
echo "Waiting for job to complete..."
sleep 10

# Step 5: Check memory after
echo ""
echo "=== AFTER ==="
free -h | grep -E "Mem|Swap"
cat mem_test.out
```

**Troubleshooting**: If you get "Memory specification can not be satisfied":
- Check `RealMemory` in slurm.conf: `grep RealMemory /etc/slurm/slurm.conf`
- Use smaller memory request: `--mem=256M` instead of `--mem=512M`
- Restart services after config changes: `sudo systemctl restart slurmctld slurmd`

### Example 5: Array Jobs

```bash
cat > array_job.sh << 'EOF'
#!/bin/bash
#SBATCH --job-name=array-test
#SBATCH --output=array_%A_%a.out
#SBATCH --array=1-10
#SBATCH --time=00:01:00

echo "Array job ID: $SLURM_ARRAY_JOB_ID"
echo "Array task ID: $SLURM_ARRAY_TASK_ID"
echo "Processing item $SLURM_ARRAY_TASK_ID"
sleep 5
echo "Completed item $SLURM_ARRAY_TASK_ID"
EOF

sbatch array_job.sh

# Check all array jobs
squeue -u $USER

# View specific array job output
cat array_12345_1.out  # JobID_ArrayTaskID
```

### Example 6: Dependent Jobs

```bash
# Create first job script
cat > job1.sh << 'EOF'
#!/bin/bash
#SBATCH --job-name=job1
#SBATCH --output=job1.out
#SBATCH --time=00:01:00

echo "Job 1 starting..."
sleep 30
echo "Job 1 completed!"
EOF

# Create second job script
cat > job2.sh << 'EOF'
#!/bin/bash
#SBATCH --job-name=job2
#SBATCH --output=job2.out
#SBATCH --time=00:01:00

echo "Job 2 starting (after job1 completed)..."
echo "Job 1 must have finished successfully!"
echo "Job 2 completed!"
EOF

# Submit first job
JOB1=$(sbatch --job-name=job1 --output=job1.out job1.sh | awk '{print $4}')
echo "Job 1 ID: $JOB1"

# Submit second job that depends on first
sbatch --job-name=job2 --output=job2.out --dependency=afterok:$JOB1 job2.sh

# Check dependencies
squeue -u $USER
```

**Explanation**: Job 2 will wait in "Dependency" state until Job 1 completes successfully. Once Job 1 finishes, Job 2 will automatically start.

### Example 7: GPU Job

**Prerequisites**: GPU nodes must be configured with `Gres=gpu:X` in slurm.conf and `/etc/slurm/gres.conf` must be set up.

```bash
cat > gpu_job.sh << 'EOF'
#!/bin/bash
#SBATCH --job-name=gpu-test
#SBATCH --output=gpu_test.out
#SBATCH --error=gpu_test.err
#SBATCH --time=00:10:00
#SBATCH --partition=gpu
#SBATCH --gres=gpu:1
#SBATCH --nodes=1
#SBATCH --ntasks=1

echo "GPU Job Information:"
echo "Job ID: $SLURM_JOB_ID"
echo "Node: $SLURM_NODELIST"
echo "GPUs allocated: $SLURM_GPUS_ON_NODE"
echo "GPU IDs: $CUDA_VISIBLE_DEVICES"

# Check GPU availability
echo ""
echo "GPU Information:"
nvidia-smi

# Example: Run a simple GPU computation
echo ""
echo "Running GPU computation..."
# If you have a GPU-enabled Python script:
# python3 my_gpu_script.py

# Or test GPU with a simple CUDA program
# ./my_cuda_program

echo ""
echo "GPU job completed!"
EOF

# Submit GPU job
sbatch gpu_job.sh

# Check job status
squeue -u $USER

# View output
cat gpu_test.out
```

**Note**: For GPU jobs to work, you need GPU nodes configured in slurm.conf with `Gres=gpu:X` and `/etc/slurm/gres.conf` file. See [Adding GPU Nodes](#adding-gpu-nodes) section.

## Testing and Validation

```bash
# Basic checks
scontrol --version
sinfo
scontrol show config

# Test jobs
srun hostname
sbatch hello.sh && sleep 5 && cat hello.out

# Check services
sudo systemctl status slurmctld slurmd
```

## Troubleshooting

### Service Issues

```bash
# Check services and logs
sudo systemctl status slurmctld slurmd
sudo tail -20 /var/log/slurm/slurmctld.log
sudo tail -20 /var/log/slurm/slurmd.log

# Validate configuration
sudo slurmctld -C

# Restart services
sudo systemctl restart slurmctld slurmd
```

### Job Issues

```bash
# Check why job is pending
squeue -j <JOBID> -o "%.18i %.9P %.8j %.8u %.2t %.10M %.6D %R"
scontrol show job <JOBID>

# Check job output
cat slurm-<JOBID>.out
cat slurm-<JOBID>.err
```

### Common Issues

```bash
# Munge authentication
munge -n | unmunge
sudo systemctl restart munge

# Node shows DOWN
sinfo -N -l
sudo systemctl restart slurmd slurmctld

# Configuration errors
sudo slurmctld -C
grep NodeName /etc/slurm/slurm.conf

# Memory shows incorrectly (1MB or N/A)
MEMORY_MB=$(free -m | awk '/^Mem:/{print $2}')
sudo sed -i "s/RealMemory=[0-9]*/RealMemory=${MEMORY_MB}/" /etc/slurm/slurm.conf
sudo systemctl restart slurmctld slurmd
sinfo -o "%N %m"
```

### Common Error Messages and Solutions

| Error Message | Solution |
|---------------|----------|
| `error: Bad SelectTypeParameter` | Remove `SelectTypeParameters` line for `select/linear` |
| `error: cannot find auth plugin` | Use `auth/munge` instead of `auth/none` |
| `error: Could not open state file` | Normal on first startup, files will be created |
| `Node configuration differs from hardware` | Update `CPUs=` in NodeName to match actual CPU count |
| `fatal: failed to initialize authentication` | Install and configure Munge properly |
| `error: Unit file does not exist` | Install `slurm-slurmctld` and `slurm-slurmd` packages |
| `Jobs stuck in PENDING` | Check partition configuration and resource availability |
| `error: too early for operation` | Wait 10-30 seconds after installing snapd, or restart snapd services |
| `Memory specification can not be satisfied` | Add `RealMemory=<MB>` to NodeName in slurm.conf, restart services |
| `SSL certificate errors (OCSP expired)` | Sync system time: `sudo chronyd -q`, update certificates: `sudo update-ca-trust` |

### Diagnostic Commands

```bash
scontrol --version
scontrol show config
sinfo
squeue
scontrol show job <JOBID>
scontrol show node localhost
```

## Additional Resources

- [Slurm Official Documentation](https://slurm.schedmd.com/)
- [Slurm Quick Start Guide](https://slurm.schedmd.com/quickstart_admin.html)
- [Slurm Configuration Tool](https://slurm.schedmd.com/configurator.html)
- [Slurm FAQ](https://slurm.schedmd.com/faq.html)

