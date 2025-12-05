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

#### 2. Install snapd

```bash
sudo dnf install -y snapd
sudo systemctl enable --now snapd.socket
sudo ln -s /var/lib/snapd/snap /snap
```

**Important**: Log out and log back in (or restart) for snap paths to work properly.

#### 3. Install Munge (Authentication)

Munge is required for secure communication between Slurm components.

```bash
# Install Munge packages
sudo dnf install -y munge munge-libs munge-devel

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

If EPEL packages don't work (common on ARM64), build from source:

```bash
# Install build dependencies
sudo dnf groupinstall -y "Development Tools"
sudo dnf install -y gcc gcc-c++ make \
    readline-devel pam-devel openssl-devel \
    python3 python3-devel mariadb-devel \
    hwloc-devel numactl-devel lua-devel \
    libcurl-devel rrdtool-devel zlib-devel \
    bzip2-devel

# Download and build Slurm
cd /tmp
wget https://download.schedmd.com/slurm/slurm-23.11.2.tar.bz2
tar xjf slurm-23.11.2.tar.bz2
cd slurm-23.11.2

# Configure
./configure --prefix=/usr --sysconfdir=/etc/slurm

# Build (takes 10-30 minutes)
make -j$(nproc)

# Install
sudo make install

# Install systemd service files
if [ -d "contribs/systemd" ]; then
    sudo cp contribs/systemd/slurmctld.service /etc/systemd/system/
    sudo cp contribs/systemd/slurmd.service /etc/systemd/system/
    sudo systemctl daemon-reload
fi
```

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
HOSTNAME=$(hostname -s)

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

NodeName=localhost NodeAddr=127.0.0.1 CPUs=${CPU_COUNT} State=UNKNOWN
PartitionName=compute Nodes=localhost Default=YES MaxTime=INFINITE State=UP
EOF
```

#### 3. Start Services

```bash
# Start services
sudo systemctl enable --now slurmctld
sudo systemctl enable --now slurmd

# Verify services are running
sudo systemctl status slurmctld
sudo systemctl status slurmd
```

#### 4. Service Management

**Start Services:**
```bash
sudo systemctl start slurmctld
sudo systemctl start slurmd
```

**Stop Services:**
```bash
# Stop both services
sudo systemctl stop slurmctld
sudo systemctl stop slurmd

# Or stop both at once
sudo systemctl stop slurmctld slurmd
```

**Restart Services:**
```bash
# Restart both services
sudo systemctl restart slurmctld
sudo systemctl restart slurmd

# Or restart both at once
sudo systemctl restart slurmctld slurmd
```

**Check Service Status:**
```bash
sudo systemctl status slurmctld
sudo systemctl status slurmd
```

**Enable/Disable Auto-Start:**
```bash
# Enable services to start on boot
sudo systemctl enable slurmctld
sudo systemctl enable slurmd

# Disable services from starting on boot
sudo systemctl disable slurmctld
sudo systemctl disable slurmd
```

**Note**: Before stopping services, you may want to cancel running jobs:
```bash
# Cancel all your jobs
scancel -u $USER

# Then stop services
sudo systemctl stop slurmctld slurmd
```

### Configuration Parameters Explained

| Parameter | Description | Example |
|-----------|-------------|---------|
| `ClusterName` | Name of your cluster | `slurm-cluster` |
| `ControlMachine` | Hostname of controller | `localhost` |
| `SlurmUser` | User that runs Slurm daemons | `slurm` |
| `SlurmctldPort` | Port for controller | `6817` |
| `SlurmdPort` | Port for compute daemon | `6818` |
| `AuthType` | Authentication method | `auth/munge` |
| `SchedulerType` | Job scheduler algorithm | `sched/backfill` |
| `SelectType` | Node selection method | `select/linear` |
| `NodeName` | Node definition | `localhost CPUs=4` |
| `PartitionName` | Job partition | `compute` |

**Important Notes:**
- **By default, nodes are CPU-only** - No GPU configuration is needed for CPU workloads
- **GPUs must be explicitly configured** - To enable GPU support, add `Gres=gpu:X` to the `NodeName` line in `slurm.conf` (where X is the number of GPUs)
- **CPU nodes**: `NodeName=localhost CPUs=4 State=UNKNOWN` (no `Gres` parameter)
- **GPU nodes**: `NodeName=gpu-node1 NodeAddr=192.168.1.20 CPUs=8 Gres=gpu:2 State=UNKNOWN` (includes `Gres=gpu:2`)
- See [Adding GPU Nodes](#adding-gpu-nodes) section for complete GPU setup instructions

## Advanced Configuration

### Multi-Node Setup

For a cluster with separate controller and compute nodes:

```bash
# On controller node, create slurm.conf
sudo tee /etc/slurm/slurm.conf > /dev/null << 'EOF'
ClusterName=slurm-cluster
ControlMachine=controller
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

NodeName=controller NodeAddr=192.168.1.10 CPUs=8 State=UNKNOWN
NodeName=compute1 NodeAddr=192.168.1.11 CPUs=4 State=UNKNOWN
NodeName=compute2 NodeAddr=192.168.1.12 CPUs=4 State=UNKNOWN

PartitionName=compute Nodes=controller,compute1,compute2 Default=YES MaxTime=INFINITE State=UP
EOF

# Copy configuration to compute nodes
sudo scp /etc/slurm/slurm.conf root@compute1:/etc/slurm/
sudo scp /etc/slurm/slurm.conf root@compute2:/etc/slurm/

# Copy Munge key to all nodes (same key everywhere!)
sudo scp /etc/munge/munge.key root@compute1:/etc/munge/
sudo scp /etc/munge/munge.key root@compute2:/etc/munge/

# On each compute node, set Munge key permissions
sudo chown munge:munge /etc/munge/munge.key
sudo chmod 600 /etc/munge/munge.key
sudo systemctl restart munge
```

### Adding Compute Nodes to Existing Cluster

To add more worker/compute nodes to your cluster:

#### Step 1: Prepare the New Compute Node

On the new compute node machine:

```bash
# Install EPEL
sudo dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm

# Install snapd
sudo dnf install -y snapd
sudo systemctl enable --now snapd.socket
sudo ln -s /var/lib/snapd/snap /snap
# Log out and back in

# Install Munge
sudo dnf install -y munge munge-libs munge-devel

# Install Slurm
sudo snap install slurm --classic
sudo dnf install -y slurm-slurmd

# Create Slurm user and directories
sudo groupadd -r slurm
sudo useradd -r -g slurm -d /var/lib/slurm -s /bin/bash slurm
sudo mkdir -p /var/spool/slurmd /var/log/slurm /etc/slurm
sudo chown slurm:slurm /var/spool/slurmd /var/log/slurm
```

#### Step 2: Copy Configuration from Controller

On the controller node:

```bash
# Get the new node's hostname and IP
# Example: compute3 with IP 192.168.1.13

# Edit slurm.conf on controller to add new node
sudo vi /etc/slurm/slurm.conf

# Add the new node definition:
# NodeName=compute3 NodeAddr=192.168.1.13 CPUs=4 State=UNKNOWN

# Update partition to include new node:
# PartitionName=compute Nodes=controller,compute1,compute2,compute3 Default=YES MaxTime=INFINITE State=UP
```

#### Step 3: Copy Files to New Node

On the controller node:

```bash
# Copy slurm.conf to new node
sudo scp /etc/slurm/slurm.conf root@compute3:/etc/slurm/

# Copy Munge key (CRITICAL: must be same on all nodes!)
sudo scp /etc/munge/munge.key root@compute3:/etc/munge/
```

#### Step 4: Configure New Node

On the new compute node:

```bash
# Set Munge key permissions
sudo chown munge:munge /etc/munge/munge.key
sudo chmod 600 /etc/munge/munge.key

# Start Munge
sudo systemctl enable --now munge

# Test Munge
munge -n | unmunge

# Start slurmd
sudo systemctl enable --now slurmd

# Check status
sudo systemctl status slurmd
```

#### Step 5: Update Controller and Verify

On the controller node:

```bash
# Reload Slurm configuration (if using scontrol reconfigure)
sudo scontrol reconfigure

# Or restart slurmctld
sudo systemctl restart slurmctld

# Verify new node appears
sinfo

# Check node details
scontrol show node compute3

# Test job on new node
srun -w compute3 hostname
```

### Adding GPU Nodes

To add GPU nodes to your cluster:

#### Step 1: Prepare GPU Node

On the GPU node:

```bash
# Install NVIDIA drivers and CUDA (if not already installed)
# Verify GPU is detected
nvidia-smi

# Install Slurm (same as regular compute node)
sudo snap install slurm --classic
sudo dnf install -y slurm-slurmd

# Create Slurm user and directories
sudo groupadd -r slurm
sudo useradd -r -g slurm -d /var/lib/slurm -s /bin/bash slurm
sudo mkdir -p /var/spool/slurmd /var/log/slurm /etc/slurm
sudo chown slurm:slurm /var/spool/slurmd /var/log/slurm
```

#### Step 2: Configure GPU Resources in slurm.conf

**Important**: GPUs must be explicitly configured by adding `Gres=gpu:X` to the `NodeName` line in `slurm.conf`. Without this parameter, nodes are CPU-only by default.

On the controller node, edit `/etc/slurm/slurm.conf`:

```bash
# Add GPU node definition
# Format: NodeName=<name> NodeAddr=<ip> CPUs=<count> Gres=gpu:<gpu_count> State=UNKNOWN
# The Gres=gpu:X parameter is REQUIRED to enable GPU support
# Example: 2 GPUs on the node
NodeName=gpu-node1 NodeAddr=192.168.1.20 CPUs=8 Gres=gpu:2 State=UNKNOWN

# Create GPU partition (optional but recommended)
PartitionName=gpu Nodes=gpu-node1 Default=NO MaxTime=INFINITE State=UP
```

**Key Points:**
- `Gres=gpu:2` means this node has 2 GPUs available
- `Gres=gpu:1` means this node has 1 GPU available
- Without `Gres=gpu:X`, the node is CPU-only (default behavior)
- The GPU count (`X`) must match the actual number of GPUs on the node

#### Step 3: Create gres.conf

Create `/etc/slurm/gres.conf` on **all nodes** (controller and GPU nodes):

```bash
# On controller and GPU nodes
sudo tee /etc/slurm/gres.conf > /dev/null << 'EOF'
# GPU Resource Configuration
# For 2 GPUs, use /dev/nvidia[0-1]
# For 4 GPUs, use /dev/nvidia[0-3]
Name=gpu File=/dev/nvidia[0-1]
EOF
```

#### Step 4: Copy Configuration and Start Services

```bash
# Copy files to GPU node
sudo scp /etc/slurm/slurm.conf root@gpu-node1:/etc/slurm/
sudo scp /etc/slurm/gres.conf root@gpu-node1:/etc/slurm/
sudo scp /etc/munge/munge.key root@gpu-node1:/etc/munge/

# On GPU node
sudo chown munge:munge /etc/munge/munge.key
sudo chmod 600 /etc/munge/munge.key
sudo systemctl enable --now munge
sudo systemctl enable --now slurmd
```

#### Step 5: Verify GPU Detection

```bash
# On controller, check GPU resources
sinfo -o "%N %G"  # Shows nodes and GPU resources
scontrol show node gpu-node1 | grep -i gpu

# Test GPU job
srun --gres=gpu:1 --partition=gpu nvidia-smi
```

#### Quick Reference: Adding a Node

```bash
# 1. On new node: Install Slurm and Munge
# 2. On controller: Add node to slurm.conf
# 3. Copy slurm.conf and munge.key to new node
# 4. On new node: Start slurmd
# 5. On controller: Verify with sinfo
```

### Partition Configuration

Create multiple partitions for different job types:

```bash
# Add to slurm.conf
PartitionName=short Nodes=ALL Default=NO MaxTime=01:00:00 State=UP
PartitionName=long Nodes=ALL Default=NO MaxTime=7-00:00:00 State=UP
PartitionName=compute Nodes=ALL Default=YES MaxTime=INFINITE State=UP
```

### Resource Limits

Configure resource limits:

```bash
# Add to slurm.conf
MaxJobCount=10000
MaxArraySize=1001
MaxNodeCount=1000
DefMemPerNode=4096
MaxMemPerNode=UNLIMITED
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

**Alternative: Use a real CPU-intensive tool**

For actual CPU-intensive work, use tools like `stress` or mathematical computations:

```bash
cat > cpu_stress.sh << 'EOF'
#!/bin/bash
#SBATCH --job-name=cpu-stress
#SBATCH --output=cpu_stress.out
#SBATCH --time=00:02:00
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=2

echo "Starting CPU stress test..."
echo "Using $SLURM_CPUS_ON_NODE CPUs"
echo "Start time: $(date)"

# Use stress command if available, or Python for calculations
if command -v stress &> /dev/null; then
    stress --cpu 2 --timeout 60s
else
    # Python-based CPU work
    python3 << PYTHON
import time
start = time.time()
result = sum(i * 2 for i in range(10000000))
end = time.time()
print(f"Calculation result: {result}")
print(f"Time taken: {end - start:.2f} seconds")
PYTHON
fi

echo "End time: $(date)"
echo "Task completed!"
EOF
```

### Example 4: Memory-Intensive Job

```bash
cat > memory_test.sh << 'EOF'
#!/bin/bash
#SBATCH --job-name=mem-test
#SBATCH --output=mem_test.out
#SBATCH --time=00:02:00
#SBATCH --mem=2G
#SBATCH --nodes=1
#SBATCH --ntasks=1

echo "Allocating memory..."
# Allocate 1.5GB of memory
python3 << PYTHON
import array
data = array.array('B', [0]) * (1500 * 1024 * 1024)
print(f"Allocated {len(data) / (1024*1024):.2f} MB")
input("Press Enter to continue...")
PYTHON

echo "Memory test completed"
EOF

sbatch memory_test.sh
```

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
# Submit first job
JOB1=$(sbatch --job-name=job1 --output=job1.out job1.sh | awk '{print $4}')
echo "Job 1 ID: $JOB1"

# Submit second job that depends on first
sbatch --job-name=job2 --output=job2.out --dependency=afterok:$JOB1 job2.sh

# Check dependencies
squeue -u $USER
```

### Example 7: Job with Environment Variables

```bash
cat > env_job.sh << 'EOF'
#!/bin/bash
#SBATCH --job-name=env-test
#SBATCH --output=env_test.out
#SBATCH --time=00:01:00

# Export custom environment variables
export MY_VAR="Hello from Slurm"
export MY_NUM=42

echo "Custom variable: $MY_VAR"
echo "Number: $MY_NUM"
echo "Slurm job ID: $SLURM_JOB_ID"
echo "Slurm node: $SLURM_NODELIST"
EOF

sbatch env_job.sh
```

### Example 8: Multiple Jobs in Sequence

```bash
# Submit multiple jobs
for i in {1..5}; do
    sbatch --job-name=job$i --output=job$i.out test.sh
done

# Check all jobs
squeue -u $USER

# Wait for completion
sleep 30

# Check all outputs
cat job*.out
```

### Example 9: Job with Time Limit

```bash
cat > timed_job.sh << 'EOF'
#!/bin/bash
#SBATCH --job-name=timed
#SBATCH --output=timed.out
#SBATCH --time=00:00:30  # 30 seconds

echo "Job started at: $(date)"
echo "Will run for 30 seconds"
sleep 30
echo "Job completed at: $(date)"
EOF

sbatch timed_job.sh
```

### Example 10: Monitoring Job Progress

```bash
# Submit job
JOBID=$(sbatch --job-name=monitor job.sh | awk '{print $4}')

# Monitor job in real-time
watch -n 1 "squeue -j $JOBID"

# Check job details
scontrol show job $JOBID

# Follow job output
tail -f job.out
```

### Example 11: GPU Job

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

**Requesting Multiple GPUs:**

```bash
cat > gpu_multi.sh << 'EOF'
#!/bin/bash
#SBATCH --job-name=gpu-multi
#SBATCH --output=gpu_multi.out
#SBATCH --partition=gpu
#SBATCH --gres=gpu:2        # Request 2 GPUs
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --time=00:10:00

echo "Allocated GPUs: $SLURM_GPUS_ON_NODE"
nvidia-smi
# Your multi-GPU workload here
EOF

sbatch gpu_multi.sh
```

**Interactive GPU Job:**

```bash
# Request GPU interactively
srun --gres=gpu:1 --partition=gpu --time=00:30:00 nvidia-smi

# Or get an interactive shell with GPU
srun --gres=gpu:1 --partition=gpu --time=01:00:00 --pty bash
# Then inside the shell:
nvidia-smi
python3 my_gpu_script.py
```

**Note**: For GPU jobs to work, you need:
1. GPU nodes configured in slurm.conf with `Gres=gpu:X`
2. `/etc/slurm/gres.conf` file on all nodes
3. NVIDIA drivers and CUDA installed on GPU nodes
4. GPU partition created in slurm.conf

## Testing and Validation

### Basic Validation

```bash
# Check Slurm version
scontrol --version

# Check configuration
scontrol show config

# Check node status
sinfo

# Check detailed node info
sinfo -N -l

# Check partition status
sinfo -l

# Check services
sudo systemctl status slurmctld
sudo systemctl status slurmd
```

### Comprehensive Test Suite

```bash
# Test 1: Simple command
srun hostname

# Test 2: Resource allocation
srun -N 1 -n 2 --time=1:00 hostname

# Test 3: Batch job
sbatch test.sh && sleep 5 && cat test.out

# Test 4: Multiple jobs
for i in {1..3}; do sbatch --job-name=test$i test.sh; done
squeue
sleep 10
cat test*.out

# Test 5: Job cancellation
JOBID=$(sbatch test.sh | awk '{print $4}')
scancel $JOBID
squeue -j $JOBID

# Test 6: Job status query
JOBID=$(sbatch test.sh | awk '{print $4}')
scontrol show job $JOBID

# Test 7: Node information
scontrol show node localhost

# Test 8: Partition information
scontrol show partition compute
```

### Performance Testing

```bash
# Test job throughput
time for i in {1..10}; do sbatch test.sh; done

# Test resource utilization
srun --time=1:00 --mem=1G stress --cpu 2 --timeout 60s

# Monitor resource usage
sstat -j <JOBID> --format=JobID,MaxRSS,MaxVMSize
```

## Troubleshooting

### Service Management

**Stop Services:**
```bash
# Stop both services
sudo systemctl stop slurmctld slurmd

# Cancel running jobs first (recommended)
scancel -u $USER
sudo systemctl stop slurmctld slurmd
```

**Start Services:**
```bash
sudo systemctl start slurmctld slurmd
```

**Restart Services:**
```bash
# Restart both services
sudo systemctl restart slurmctld slurmd
```

**Check Service Status:**
```bash
sudo systemctl status slurmctld
sudo systemctl status slurmd
```

**Enable/Disable Auto-Start:**
```bash
# Enable services to start on boot
sudo systemctl enable slurmctld slurmd

# Disable services from starting on boot
sudo systemctl disable slurmctld slurmd
```

### Service Issues

#### slurmctld Won't Start

```bash
# Check service status
sudo systemctl status slurmctld

# Check logs
sudo tail -50 /var/log/slurm/slurmctld.log
sudo journalctl -u slurmctld -n 50

# Validate configuration
sudo slurmctld -C

# Check file permissions
ls -la /var/spool/slurmctld
sudo chown slurm:slurm /var/spool/slurmctld

# Check if port is in use
sudo netstat -tlnp | grep 6817
```

#### slurmd Won't Start

```bash
# Check service status
sudo systemctl status slurmd

# Check logs
sudo tail -50 /var/log/slurm/slurmd.log
sudo journalctl -u slurmd -n 50

# Verify Munge
munge -n | unmunge

# Check network connectivity to controller
ping localhost
telnet localhost 6817

# Verify configuration matches controller
diff /etc/slurm/slurm.conf controller:/etc/slurm/slurm.conf
```

### Configuration Issues

#### Invalid Configuration

```bash
# Validate configuration syntax
sudo slurmctld -C

# Check for common errors
grep -E "SelectTypeParameters|AuthType" /etc/slurm/slurm.conf

# Verify node definitions
grep NodeName /etc/slurm/slurm.conf

# Verify partition definitions
grep PartitionName /etc/slurm/slurm.conf
```

#### Node Shows as DOWN

```bash
# Check node status
sinfo -N -l
scontrol show node localhost

# Check slurmd status
sudo systemctl status slurmd

# Verify Munge key matches
sudo md5sum /etc/munge/munge.key

# Restart services
sudo systemctl restart munge
sudo systemctl restart slurmd
sudo systemctl restart slurmctld
```

### Job Issues

#### Jobs Stuck in PENDING

```bash
# Check why job is pending (most important!)
squeue -j <JOBID> -o "%.18i %.9P %.8j %.8u %.2t %.10M %.6D %R"

# Check job details to see what it's requesting
scontrol show job <JOBID>

# Check partition configuration
sinfo -l
scontrol show partition compute

# Check node availability
sinfo -o "%P %a %l %D %t %N"

# Check resource availability
sinfo -o "%P %C %m %G"

# Common issue: Job requesting more nodes than available
# If job shows "PartitionConfig" reason, check:
# 1. Job is requesting correct number of nodes (--nodes=1 for single-node setup)
# 2. Partition allows the requested resources
# 3. Node count in slurm.conf matches actual setup
```

#### Jobs Fail Immediately

```bash
# Check job output
cat slurm-<JOBID>.out
cat slurm-<JOBID>.err

# Check job details
scontrol show job <JOBID>

# Verify job script is executable
chmod +x job.sh

# Check resource requests
# Ensure requested resources are available
```

#### Permission Denied Errors

```bash
# Check file permissions
ls -la job.sh
chmod +x job.sh

# Check output directory permissions
mkdir -p output
chmod 755 output

# Verify user can write
touch output/test && rm output/test
```

### Authentication Issues

#### Munge Authentication Fails

```bash
# Verify Munge key is same on all nodes
sudo md5sum /etc/munge/munge.key

# Restart Munge
sudo systemctl restart munge

# Test Munge
munge -n | unmunge

# Check Munge logs
sudo journalctl -u munge -n 20

# Verify key permissions
ls -la /etc/munge/munge.key
sudo chown munge:munge /etc/munge/munge.key
sudo chmod 600 /etc/munge/munge.key
```

### Network Issues

#### Cannot Connect to Controller

```bash
# Test network connectivity
ping <controller-hostname>
ping <controller-ip>

# Test port connectivity
telnet <controller-hostname> 6817
nc -zv <controller-hostname> 6817

# Check firewall
sudo firewall-cmd --list-all
sudo firewall-cmd --permanent --add-port=6817/tcp
sudo firewall-cmd --permanent --add-port=6818/tcp
sudo firewall-cmd --reload

# Check SELinux
sudo getenforce
sudo setenforce 0  # Temporarily disable for testing
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

### Debug Mode

Enable detailed logging:

```bash
# Edit slurm.conf
sudo vi /etc/slurm/slurm.conf

# Add debug flags
SlurmctldDebug=debug5
SlurmdDebug=debug5

# Restart services
sudo systemctl restart slurmctld
sudo systemctl restart slurmd

# Check detailed logs
sudo tail -f /var/log/slurm/slurmctld.log
sudo tail -f /var/log/slurm/slurmd.log
```

### Useful Diagnostic Commands

```bash
# Check Slurm version
scontrol --version

# Show full configuration
scontrol show config

# Show all nodes
scontrol show nodes

# Show all partitions
scontrol show partition

# Show all jobs
squeue

# Show specific job
scontrol show job <JOBID>

# Cancel job
scancel <JOBID>

# Check cluster status
sinfo
sinfo -l
sinfo -N -l

# Check accounting (if enabled)
sacct -j <JOBID>
sacct -u $USER
```

## Additional Resources

- [Slurm Official Documentation](https://slurm.schedmd.com/)
- [Slurm Quick Start Guide](https://slurm.schedmd.com/quickstart_admin.html)
- [Slurm Configuration Tool](https://slurm.schedmd.com/configurator.html)
- [Slurm FAQ](https://slurm.schedmd.com/faq.html)

