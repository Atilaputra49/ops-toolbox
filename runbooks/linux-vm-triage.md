# Linux VM Triage Runbook

## Overview
Step-by-step guide for diagnosing and troubleshooting common Linux VM issues in Azure cloud environments.

## Prerequisites
- SSH access to the VM
- Azure CLI installed (`az' command)
- Appropriate RBAC permissions

---`

## 🚨 Quick Triage Checklist

When a Linux VM issue is reported, check in this order:
1. Can you connect to it? (Network/SSH)
2. Is it running? (VM status)
3. Are resources exhausted? (CPU/Memory/Disk)
4. Are critical services running? (systemd services)
5. Are there recent changes? (Logs/deployments)

---

## 1. Initial Assessment

### Check VM Status from Azure
```bash
# List all VMs in subscription
az vm list -o table

# Get specific VM status
az vm get-instance-view \
  --resource-group <rg-name> \
  --name <vm-name> \
  --query "instanceView.statuses[?starts_with(code, 'PowerState/')].displayStatus" \
  -o tsv

# Get VM details
az vm show \
  --resource-group <rg-name> \
  --name <vm-name> \
  -o table
```

### Check Network Connectivity
```bash
# Check NSG rules
az network nsg list --resource-group <rg-name> -o table

# Check effective NSG rules on NIC
az network nic list-effective-nsg \
  --resource-group <rg-name> \
  --name <nic-name>

# Test connectivity (requires Network Watcher)
az network watcher test-connectivity \
  --resource-group <rg-name> \
  --source-resource <vm-name> \
  --dest-address <destination-ip> \
  --dest-port 22
```

---

## 2. SSH Connection Issues

### If SSH fails:

```bash
# Option 1: Use Azure Serial Console
# Go to Azure Portal → VM → Serial Console

# Option 2: Use Run Command from Azure CLI
az vm run-command invoke \
  --resource-group <rg-name> \
  --name <vm-name> \
  --command-id RunShellScript \
  --scripts "systemctl status sshd"

# Option 3: Reset SSH configuration
az vm user reset-ssh \
  --resource-group <rg-name> \
  --name <vm-name>
```

### Common SSH troubleshooting:
```bash
# On the VM (via Serial Console or Run Command):

# Check SSH service status
sudo systemctl status sshd

# Check SSH logs
sudo tail -f /var/log/auth.log     # Debian/Ubuntu
sudo tail -f /var/log/secure       # RHEL/CentOS

# Verify SSH is listening
sudo netstat -tlnp | grep :22
# or
sudo ss -tlnp | grep :22

# Check SSH config
sudo sshd -t  # Test configuration
```

---

## 3. High CPU Usage

### Identify the culprit:
```bash
# Real-time CPU monitoring
top
# Press Shift+P to sort by CPU usage
# Press Shift+M to sort by memory usage

# Better alternative
htop  # If installed

# Get current CPU usage snapshot
mpstat 1 5  # 5 samples, 1 second apart

# Find top CPU-consuming processes
ps aux --sort=-%cpu | head -10

# Get detailed process info
ps -p <PID> -o pid,user,%cpu,%mem,cmd,start_time

# Check process tree
pstree -p <PID>
```

### Historical CPU data:
```bash
# View CPU usage history (if sar is installed)
sar -u        # CPU usage
sar -u 1 10   # Current CPU, 10 samples

# Check system load
uptime
# Load average: 1min, 5min, 15min
# Rule of thumb: load > CPU cores = potential issue
```

### Azure-specific CPU monitoring:
```bash
# Get VM metrics from Azure
az monitor metrics list \
  --resource /subscriptions/<sub-id>/resourceGroups/<rg-name>/providers/Microsoft.Compute/virtualMachines/<vm-name> \
  --metric "Percentage CPU" \
  --start-time 2024-10-08T00:00:00Z \
  --end-time 2024-10-08T23:59:59Z \
  --interval PT1H
```

### Resolution steps:
```bash
# If process is runaway/stuck:
kill -15 <PID>  # Graceful shutdown
# Wait 10 seconds
kill -9 <PID>   # Force kill if needed

# If it's a service:
sudo systemctl restart <service-name>

# Check for zombie processes
ps aux | grep Z

# Investigate application logs
sudo journalctl -u <service-name> --since "1 hour ago"
```

---

## 4. High Memory Usage

### Check memory usage:
```bash
# Overall memory status
free -h

# Detailed memory info
cat /proc/meminfo

# Check for memory leaks
ps aux --sort=-%mem | head -10

# Per-process memory breakdown
pmap -x <PID>

# Check swap usage
swapon --show
vmstat 1 5
```

### If memory is exhausted:
```bash
# Check OOM killer logs
sudo dmesg | grep -i "out of memory"
sudo journalctl -k | grep -i "killed process"

# Find memory-hungry processes
smem -rs uss  # If installed

# Clear cache (safe, won't lose data)
sudo sync
sudo sh -c 'echo 3 > /proc/sys/vm/drop_caches'
```

---

## 5. Disk Space Issues

### Check disk usage:
```bash
# Overall disk space
df -h

# Find largest directories
du -h / --max-depth=1 | sort -rh | head -10
du -sh /var/* | sort -rh | head -10

# Find large files
find / -type f -size +100M -exec ls -lh {} \; 2>/dev/null

# Check inode usage (can fill up even with space available)
df -i
```

### Azure disk-specific:
```bash
# List attached disks
lsblk

# Check disk performance metrics
iostat -x 1 5  # If sysstat installed

# Azure CLI: Get disk info
az disk list --resource-group <rg-name> -o table
```

### Clean up space:
```bash
# Clear old logs
sudo journalctl --vacuum-time=7d
sudo journalctl --vacuum-size=500M

# Clear apt cache (Ubuntu/Debian)
sudo apt-get clean
sudo apt-get autoclean

# Clear yum cache (RHEL/CentOS)
sudo yum clean all

# Remove old kernels (be careful!)
# Ubuntu:
sudo apt autoremove --purge

# Find and delete core dumps
find /var -name "core.*" -type f -delete
```

---

## 6. Service Health Check

### Check critical services:
```bash
# List all services
systemctl list-units --type=service --all

# Check specific service status
sudo systemctl status <service-name>

# View service logs
sudo journalctl -u <service-name> -n 50 --no-pager

# Check failed services
systemctl --failed

# Restart a service
sudo systemctl restart <service-name>

# Enable service to start on boot
sudo systemctl enable <service-name>
```

### Common services to check:
- `sshd` - SSH daemon
- `nginx` / `apache2` - Web servers
- `docker` - Container runtime
- `kubelet` - Kubernetes agent
- Application-specific services

---

## 7. Network Troubleshooting

### Basic connectivity:
```bash
# Check network interfaces
ip addr show
ifconfig  # If net-tools installed

# Check routing table
ip route show
route -n

# Test DNS resolution
nslookup google.com
dig google.com

# Check listening ports
sudo netstat -tlnp
sudo ss -tlnp

# Test outbound connectivity
curl -I https://google.com
wget --spider https://google.com
```

### Firewall checks:
```bash
# Check iptables rules
sudo iptables -L -n -v

# Check firewalld (RHEL/CentOS)
sudo firewall-cmd --list-all

# Check ufw (Ubuntu)
sudo ufw status verbose
```

---

## 8. Log Analysis

### System logs:
```bash
# View system logs
sudo journalctl -n 100 --no-pager

# Follow logs in real-time
sudo journalctl -f

# Logs since boot
sudo journalctl -b

# Logs for specific priority (error and above)
sudo journalctl -p err -n 50

# Kernel logs
dmesg | tail -50
```

### Common log locations:
```bash
/var/log/syslog        # System logs (Debian/Ubuntu)
/var/log/messages      # System logs (RHEL/CentOS)
/var/log/auth.log      # Authentication logs
/var/log/secure        # Authentication logs (RHEL/CentOS)
/var/log/kern.log      # Kernel logs
/var/log/boot.log      # Boot logs
```

---

## 9. Azure-Specific Diagnostics

### Boot diagnostics:
```bash
# Enable boot diagnostics
az vm boot-diagnostics enable \
  --resource-group <rg-name> \
  --name <vm-name> \
  --storage <storage-account-url>

# Get boot diagnostics log
az vm boot-diagnostics get-boot-log \
  --resource-group <rg-name> \
  --name <vm-name>
```

### VM Extensions:
```bash
# List installed extensions
az vm extension list \
  --resource-group <rg-name> \
  --vm-name <vm-name> \
  -o table

# Get extension status
az vm extension show \
  --resource-group <rg-name> \
  --vm-name <vm-name> \
  --name <extension-name>
```

### Activity logs:
```bash
# Get recent activity logs for the VM
az monitor activity-log list \
  --resource-group <rg-name> \
  --resource-id /subscriptions/<sub-id>/resourceGroups/<rg-name>/providers/Microsoft.Compute/virtualMachines/<vm-name> \
  --start-time 2024-10-08T00:00:00Z \
  --offset 24h
```

---

## 10. Performance Baseline

### Establish baseline metrics:
```bash
# CPU info
lscpu
cat /proc/cpuinfo

# Memory info
free -h
cat /proc/meminfo

# Disk info
lsblk
df -h

# Network throughput (if iperf3 installed)
iperf3 -c <target-server>

# System uptime and load
uptime
w
```

---

## Quick Reference Commands

| Issue         | Command                                      |
|---------------|----------------------------------------------|
| High CPU      | `top`, `ps aux --sort=-%cpu \| head -10`     |
| High Memory   | `free -h`, `ps aux --sort=-%mem \| head -10` |
| Disk Full     | `df -h`, `du -sh /* \| sort -rh`             |
| Service Down  | `systemctl status <service>`                 |
| Network Issue | `ip addr`, `ss -tlnp`, `ping`                |
| Check Logs    | `journalctl -n 100`, `dmesg`                 |

---

## Escalation Criteria

Escalate to senior engineer or Azure support if:
- VM is unresponsive and cannot be accessed via serial console
- Repeated crashes or kernel panics
- Disk corruption suspected
- Performance issues persist after basic troubleshooting
- Azure platform issue suspected (check Azure Status)

---

## Additional Resources

- [Azure VM Troubleshooting Documentation](https://learn.microsoft.com/azure/virtual-machines/troubleshooting/)
- [Linux System Administration Guide](https://www.tldp.org/LDP/sag/html/)
- [Azure CLI Reference](https://learn.microsoft.com/cli/azure/)

---

**Last Updated:** October 2024  
**Maintained by:** [Your Name]
