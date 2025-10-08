# Ops Toolbox 🔧

A practical collection of DevOps scripts, runbooks, and cheatsheets for day-to-day operations and troubleshooting in Azure cloud environments.

## 📁 Contents

### Scripts
Bash utilities for common operational tasks:

- **log-rotate.sh** - Automated log rotation with compression and retention policies
- **simple-backup.sh** - Lightweight backup solution for configuration files and databases
- **health-check.sh** - System health monitoring script (CPU, memory, disk, services)

### Runbooks
- **linux-vm-triage.md** - Step-by-step guide for diagnosing Linux VM issues with Azure CLI commands

### Cheatsheets
- **azure-troubleshooting.png** - Quick reference architecture for common Azure troubleshooting workflows

## 🚀 Quick Start
```bash
# Clone the repository
git clone https://github.com/yourusername/ops-toolbox.git
cd ops-toolbox

# Make scripts executable
chmod +x scripts/*.sh

# Run health check
./scripts/health-check.sh
