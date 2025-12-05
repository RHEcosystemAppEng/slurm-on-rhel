# Slurm Architecture Guide

This document explains the architecture of Slurm workload manager, including the key components and how they work together.

## Table of Contents

1. [Overview](#overview)
2. [Key Components](#key-components)
3. [Architecture Diagram](#architecture-diagram)
4. [Single-Node vs Multi-Node](#single-node-vs-multi-node)
5. [How Jobs Flow Through the System](#how-jobs-flow-through-the-system)
6. [Communication Between Components](#communication-between-components)

## Overview

Slurm (Simple Linux Utility for Resource Management) is a workload manager and job scheduler for Linux clusters. It manages resources (CPUs, memory, nodes) and schedules jobs across compute nodes.

## Key Components

### slurmctld (Slurm Controller Daemon)

**The "Brain" of the Cluster**

- **Role**: Central controller and scheduler
- **Location**: Runs on the controller node (one per cluster)
- **Responsibilities**:
  - Job scheduling and queue management
  - Resource allocation decisions
  - Job state tracking and accounting
  - Node status monitoring
  - User authentication and authorization
  - Cluster-wide configuration management
- **Think of it as**: The manager that decides which jobs run where and when

**Key Features**:
- Maintains job queue
- Tracks resource availability
- Makes scheduling decisions
- Logs all job activities

### slurmd (Slurm Daemon)

**The "Worker" on Each Compute Node**

- **Role**: Local agent and job executor
- **Location**: Runs on every compute node (one per node)
- **Responsibilities**:
  - Executes jobs assigned by slurmctld
  - Reports node status (CPU, memory, availability)
  - Manages job processes on that node
  - Communicates node health to controller
  - Handles job cleanup after completion
- **Think of it as**: The worker that actually runs jobs on its node

**Key Features**:
- Monitors local resources
- Executes job steps
- Reports status to controller
- Manages job processes

### Client Tools

User-facing commands that interact with slurmctld:

- **srun**: Submit and run interactive jobs
- **sbatch**: Submit batch jobs
- **squeue**: View job queue
- **sinfo**: View node/partition status
- **scontrol**: Control and view cluster configuration
- **scancel**: Cancel jobs

## Architecture Diagram

### Single-Node Setup (Controller + Compute on Same Machine)

```
┌─────────────────────────────────────────────┐
│         Single Node (localhost)             │
│                                             │
│  ┌──────────────────────────────────────┐   │
│  │  slurmctld (Controller)              │   │
│  │  - Job scheduling                    │   │
│  │  - Resource management               │   │
│  │  - Queue management                  │   │
│  └──────────────┬───────────────────────┘   │
│                 │                           │
│                 │ Commands & Status         │
│                 │                           │
│  ┌──────────────▼───────────────────────┐   │
│  │  slurmd (Compute Daemon)             │   │
│  │  - Executes jobs                     │   │
│  │  - Reports node status               │   │
│  │  - Manages processes                 │   │
│  └──────────────────────────────────────┘   │
│                                             │
│  ┌──────────────────────────────────────┐   │
│  │  Client Tools                        │   │
│  │  (srun, sbatch, squeue, sinfo)       │   │
│  └──────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

### Multi-Node Cluster Setup

```
┌─────────────────────────────────────────────────────────────┐
│                    Controller Node                          │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  slurmctld (Controller Daemon)                       │   │
│  │  - Job scheduling                                    │   │
│  │  - Resource allocation                               │   │
│  │  - Queue management                                  │   │
│  │  - Cluster coordination                              │   │
│  └──────────┬───────────────────────────────────────────┘   │
│             │                                               │
│             │ Job Commands & Status Updates                 │
│             │                                               │
│  ┌──────────▼───────────────────────────────────────────┐   │
│  │  slurmd (also runs on controller if it's compute)    │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Client Tools                                        │   │
│  │  (srun, sbatch, squeue, sinfo, scontrol)             │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────┬──────────────────────────────────────────────┘
               │
               │ Network Communication
               │
    ┌──────────┴──────────┐
    │                     │
    ▼                     ▼
┌──────────────┐     ┌──────────────┐
│ Compute      │     │ Compute      │
│ Node 1       │     │ Node 2       │
│              │     │              │
│  ┌────────┐  │     │  ┌────────┐  │
│  │ slurmd │  │     │  │ slurmd │  │
│  │        │  │     │  │        │  │
│  │ Worker │  │     │  │ Worker │  │
│  └────────┘  │     │  └────────┘  │
│              │     │              │
│  CPUs: 4     │     │  CPUs: 8     │
│  RAM: 16G    │     │  RAM: 32G    │
└──────────────┘     └──────────────┘
    │                     │
    └──────────┬──────────┘
               │
               ▼
    ┌──────────────────────┐
    │  More Compute Nodes  │
    │  ...                 │
    └──────────────────────┘
```

## Single-Node vs Multi-Node

### Single-Node Setup

**When to use**: Development, testing, small workloads, learning

**Characteristics**:
- Controller and compute on same machine
- Both `slurmctld` and `slurmd` run on one node
- Simpler setup and configuration
- Limited to resources of one machine
- Good for understanding Slurm concepts

**Example**: Your current setup in the VM

### Multi-Node Setup

**When to use**: Production clusters, HPC workloads, large-scale computing

**Characteristics**:
- Separate controller node
- Multiple compute nodes
- One `slurmctld` on controller
- One `slurmd` on each compute node
- Network communication between nodes
- Scales to hundreds/thousands of nodes

**Example**: Production HPC cluster

## How Jobs Flow Through the System

### Job Submission Flow

```
1. User submits job
   └─> sbatch job.sh
       └─> Client tool sends request to slurmctld

2. slurmctld receives job
   └─> Validates job requirements
   └─> Adds to job queue
   └─> Scheduler evaluates resources

3. Scheduler makes decision
   └─> Checks available resources
   └─> Selects appropriate node(s)
   └─> Allocates resources

4. slurmctld sends job to slurmd
   └─> Communicates with slurmd on selected node
   └─> Provides job script and environment

5. slurmd executes job
   └─> Creates job environment
   └─> Runs job script
   └─> Monitors job execution

6. Job completes
   └─> slurmd reports completion to slurmctld
   └─> slurmctld updates job state
   └─> User can view results
```

### Detailed Example: Batch Job Submission

```
User Command: sbatch myjob.sh
    │
    ▼
┌─────────────────┐
│  sbatch (client)│
└────────┬────────┘
         │
         │ 1. Submit job request
         ▼
┌─────────────────┐
│   slurmctld     │
│                 │
│  ┌───────────┐  │
│  │  Queue    │  │ 2. Add to queue
│  └─────┬─────┘  │
│        │        │
│  ┌─────▼─────┐  │
│  │ Scheduler │  │ 3. Evaluate resources
│  └─────┬─────┘  │
│        │        │
│        │    4. Allocate resources
└────────┼────────┘
         │
         │ 5. Send job to node
         ▼
┌─────────────────┐
│   slurmd        │
│   (on node)     │
│                 │
│  ┌───────────┐  │
│  │ Execute   │  │ 6. Run job
│  │ Job       │  │
│  └─────┬─────┘  │
│        │        │
│        │   7. Report completion
└────────┼────────┘
         │
         ▼
    Job Complete
```

## Communication Between Components

### Ports and Protocols

- **slurmctld**: Listens on port 6817 (default)
- **slurmd**: Listens on port 6818 (default)
- **Authentication**: Uses Munge for secure communication
- **Protocol**: Custom Slurm protocol over TCP

### Communication Patterns

1. **Client → slurmctld**:
   - Job submission (sbatch, srun)
   - Status queries (squeue, sinfo)
   - Control commands (scontrol, scancel)

2. **slurmctld → slurmd**:
   - Job allocation commands
   - Resource queries
   - Configuration updates

3. **slurmd → slurmctld**:
   - Node status updates
   - Job completion notifications
   - Resource availability reports

### Authentication

- **Munge**: Used for authentication between components
- Same `munge.key` must be on all nodes
- Ensures secure communication in multi-node setups

## Component Comparison

| Feature | slurmctld | slurmd |
|---------|-----------|--------|
| **Purpose** | Scheduler/Manager | Worker/Executor |
| **Location** | Controller node only | Every compute node |
| **Count** | 1 per cluster | 1 per compute node |
| **Primary Function** | Decides what runs | Actually runs jobs |
| **User Interaction** | Indirect (via commands) | None (background) |
| **Port** | 6817 | 6818 |
| **Logs** | `/var/log/slurm/slurmctld.log` | `/var/log/slurm/slurmd.log` |
| **State Files** | `/var/spool/slurmctld/` | `/var/spool/slurmd/` |

## Real-World Analogy

Think of Slurm like a **restaurant**:

- **slurmctld** = Head Chef/Manager
  - Takes orders (job submissions)
  - Decides which cook handles which order
  - Manages the kitchen schedule
  - Tracks order status

- **slurmd** = Individual Cooks
  - Each cook (node) prepares dishes (jobs)
  - Reports when dishes are ready
  - Works on assigned tasks

- **Client Tools** = Waiters
  - Take orders from customers (users)
  - Deliver orders to kitchen (slurmctld)
  - Bring completed dishes back (job results)

## Summary

- **slurmctld**: The brain - makes decisions, manages resources, schedules jobs
- **slurmd**: The workers - execute jobs, report status, manage local resources
- **Both are essential**: You need slurmctld to manage the cluster and slurmd to actually run jobs
- **Single-node**: Both run on same machine (your current setup)
- **Multi-node**: One slurmctld on controller, one slurmd per compute node

Understanding this architecture helps with:
- Troubleshooting issues
- Configuring Slurm properly
- Optimizing job performance
- Scaling to larger clusters

## Additional Resources

- [Slurm Architecture Documentation](https://slurm.schedmd.com/overview.html)
- [Slurm Quick Start Guide](https://slurm.schedmd.com/quickstart_admin.html)
- [Configuration Guide](CONFIGURATION.md)
- [Troubleshooting Guide](TROUBLESHOOTING.md)

