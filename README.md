# Slurm on RHEL

Complete guide for setting up and using Slurm workload manager on Red Hat Enterprise Linux (RHEL).

## Quick Links

- **[Quick Start Guide](QUICK_START.md)** - Get Slurm running in 5 steps
- **[Detailed Guide](docs/DETAILED_GUIDE.md)** - Comprehensive examples, configuration, and troubleshooting
- **[Architecture Guide](docs/ARCHITECTURE.md)** - Understand how Slurm works

## Overview

This repository provides a complete, reproducible setup for running Slurm workload manager on RHEL 9. Slurm is an open-source job scheduler and workload manager used in high-performance computing environments.

## Prerequisites

- RHEL 9 installed (in VM or physical machine)
- Internet connectivity
- Root or sudo access
- Minimum 2GB RAM, 2 CPU cores

## Quick Start

If you already have RHEL 9 installed, follow the [Quick Start Guide](QUICK_START.md):

1. Install EPEL repository
2. Install snapd
3. Install Munge (authentication)
4. Install Slurm
5. Configure and start services

## Documentation

### Quick Start Guide

The [Quick Start Guide](QUICK_START.md) provides a streamlined setup process to get Slurm running quickly. Perfect for:
- First-time setup
- Quick reference
- Basic single-node configuration

### Detailed Guide

The [Detailed Guide](docs/DETAILED_GUIDE.md) includes:
- Detailed installation steps
- Advanced configuration options
- 10+ practical examples
- Comprehensive testing procedures
- Complete troubleshooting guide

### Architecture Guide

The [Architecture Guide](docs/ARCHITECTURE.md) explains:
- How Slurm components work
- slurmctld vs slurmd
- Job flow through the system
- Communication patterns

## Getting RHEL

If you need RHEL 9:

1. **Register for Free Developer Subscription**: [developers.redhat.com](https://developers.redhat.com/)
2. **Download RHEL 9 ISO**: [access.redhat.com/downloads](https://access.redhat.com/downloads)
   - Select: Bare metal - Installer (.iso)
   - Architecture: aarch64 (ARM64) for Apple Silicon, x86_64 for Intel/AMD
3. **Install in VM**: Use VirtualBox, Parallels Desktop, or virt-install

## Repository Structure

```
slurm-on-rhel/
├── README.md                 # This file
├── QUICK_START.md           # Quick setup guide
├── configs/
│   └── slurm.conf.template  # Reference configuration template
├── docs/
│   ├── DETAILED_GUIDE.md    # Comprehensive guide
│   └── ARCHITECTURE.md      # Architecture explanation
└── video/
    └── VIDEO_SCRIPT.md      # Video tutorial script (for recording)
```

## Support

- Check the [Detailed Guide](docs/DETAILED_GUIDE.md) troubleshooting section
- Review [Slurm Official Documentation](https://slurm.schedmd.com/)
- Open an issue in this repository

## License

This repository and its contents are provided as-is for educational and reference purposes.
