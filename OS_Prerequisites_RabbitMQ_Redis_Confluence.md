# Operating System Prerequisites for RabbitMQ and Redis

## Document Information
| Field | Value |
|-------|-------|
| Platform | Red Hat Enterprise Linux 8.x |
| RabbitMQ Version | 4.1.x |
| Redis Version | 8.x |
| Last Updated | January 2026 |

---

## Table of Contents
1. [Introduction](#introduction)
2. [Common OS Prerequisites](#common-os-prerequisites)
3. [RabbitMQ-Specific Prerequisites](#rabbitmq-specific-prerequisites)
4. [Redis-Specific Prerequisites](#redis-specific-prerequisites)
5. [Quick Reference Checklist](#quick-reference-checklist)

---

## Introduction

This document outlines the essential operating system configurations required for production-ready RabbitMQ and Redis deployments on RHEL 8. Each configuration includes:
- **What** needs to be changed
- **Why** the change is necessary
- **How** to implement the change
- **How to verify** the change was applied correctly

> **Important**: All configurations have been tested and validated for enterprise environments. Apply these settings on all cluster nodes before installing RabbitMQ or Redis.

---

## Common OS Prerequisites

These settings apply to both RabbitMQ and Redis installations.

---

### 1. System Updates and Patch Management

#### Why This Is Needed
- Ensures system security by applying the latest security patches
- Provides bug fixes that may affect application stability
- Maintains compliance with security policies
- Prevents known vulnerabilities from being exploited

#### How to Configure
```bash
# Check current OS version
cat /etc/redhat-release

# Update system packages (schedule during maintenance window)
sudo dnf update -y

# Check if reboot is required after kernel updates
sudo needs-restarting -r

# If reboot required:
sudo reboot
```

#### Verification
```bash
cat /etc/redhat-release
# Expected: Red Hat Enterprise Linux release 8.x (Ootpa)
```

---

### 2. Required System Packages

#### Why This Is Needed
| Package | Purpose |
|---------|---------|
| `wget`, `curl` | Download packages and scripts from remote servers |
| `gnupg2` | Verify package signatures and ensure integrity |
| `socat` | Required for inter-process communication (especially RabbitMQ) |
| `logrotate` | Manage log file rotation to prevent disk exhaustion |
| `net-tools`, `telnet`, `nc` | Network troubleshooting and connectivity testing |
| `bind-utils` | DNS lookup tools (`nslookup`, `dig`) for hostname resolution |
| `sysstat`, `htop`, `iotop` | System monitoring and performance analysis |

#### How to Configure
```bash
# Install core dependencies
sudo dnf install -y wget curl gnupg2 socat logrotate

# Install network troubleshooting tools
sudo dnf install -y net-tools telnet nc bind-utils

# Install monitoring utilities
sudo dnf install -y sysstat htop iotop
```

#### Verification
```bash
rpm -qa | grep -E "wget|curl|socat|gnupg2"
```

---

### 3. Hostname and DNS Configuration

#### Why This Is Needed
- **Clustering requires hostname resolution**: RabbitMQ and Redis nodes identify each other using hostnames (FQDNs)
- **Incorrect hostname configuration is the #1 cause of cluster formation failures**
- Proper DNS ensures consistent node identification across network changes
- Log correlation and monitoring tools rely on consistent hostnames

#### How to Configure
```bash
# Set FQDN hostname (replace with your domain)
sudo hostnamectl set-hostname node1.example.com

# Verify hostname
hostname
hostname -f

# Configure /etc/hosts (if DNS is not available)
sudo vi /etc/hosts
```

Add cluster node entries:
```
# RabbitMQ/Redis Cluster Nodes
10.10.10.101 node1.example.com node1
10.10.10.102 node2.example.com node2
10.10.10.103 node3.example.com node3
```

#### Verification
```bash
# Test hostname resolution for all nodes
ping -c 2 node1.example.com
ping -c 2 node2.example.com
ping -c 2 node3.example.com

# Verify reverse DNS lookup
nslookup node1.example.com
```

> **Best Practice**: For production environments, use corporate DNS servers instead of `/etc/hosts` when possible.

---

### 4. Firewall Configuration

#### Why This Is Needed
- Allows legitimate traffic to reach RabbitMQ/Redis services
- Blocks unauthorized access to protect the cluster
- Enables inter-node communication for clustering
- Provides access to management interfaces for monitoring

#### RabbitMQ Ports
| Port | Protocol | Purpose |
|------|----------|---------|
| 5672 | TCP | AMQP protocol (main messaging port) |
| 15672 | TCP | Management HTTP API and Web UI |
| 25672 | TCP | Inter-node and CLI tools communication |
| 4369 | TCP | EPMD (Erlang Port Mapper Daemon) |
| 35672-35682 | TCP | Erlang distribution for inter-node communication |

#### Redis Ports
| Port | Protocol | Purpose |
|------|----------|---------|
| 6379 | TCP | Redis client connections |
| 16379 | TCP | Redis Cluster bus (node-to-node communication) |
| 26379 | TCP | Redis Sentinel (if using Sentinel for HA) |

#### How to Configure
```bash
# Check firewall status
sudo firewall-cmd --state

# RabbitMQ ports
sudo firewall-cmd --permanent --add-port=5672/tcp
sudo firewall-cmd --permanent --add-port=15672/tcp
sudo firewall-cmd --permanent --add-port=25672/tcp
sudo firewall-cmd --permanent --add-port=4369/tcp
sudo firewall-cmd --permanent --add-port=35672-35682/tcp

# Redis ports
sudo firewall-cmd --permanent --add-port=6379/tcp
sudo firewall-cmd --permanent --add-port=16379/tcp
sudo firewall-cmd --permanent --add-port=26379/tcp

# Reload firewall to apply changes
sudo firewall-cmd --reload
```

#### Verification
```bash
# List all configured rules
sudo firewall-cmd --list-all

# Test port connectivity from another node
telnet node1 5672
telnet node1 6379
nc -zv node1 5672
```

> **Security Best Practice**: For production, restrict management UI access to admin networks:
> ```bash
> sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.20.0.0/24" port port="15672" protocol="tcp" accept'
> ```

---

### 5. SELinux Configuration

#### Why This Is Needed
- SELinux provides mandatory access control for enhanced security
- Proper configuration allows RabbitMQ/Redis to operate while maintaining security
- **Disabling SELinux is NOT recommended for production** as it reduces security posture
- May be required for compliance (PCI-DSS, HIPAA, etc.)

#### How to Configure
```bash
# Check SELinux status
sestatus
getenforce

# RECOMMENDED: Keep SELinux enabled with proper policies
# Allow RabbitMQ to use necessary ports
sudo semanage port -a -t amqp_port_t -p tcp 5672
sudo semanage port -a -t http_port_t -p tcp 15672

# Allow Redis port
sudo semanage port -a -t redis_port_t -p tcp 6379

# Allow network connections
sudo setsebool -P nis_enabled 1
```

#### Troubleshooting SELinux Denials
```bash
# Check for SELinux denials
sudo ausearch -m avc -ts recent | grep -E "rabbitmq|redis"

# Generate custom policy if needed (consult security team first)
sudo grep rabbitmq /var/log/audit/audit.log | audit2allow -M rabbitmq_custom
sudo semodule -i rabbitmq_custom.pp
```

#### Verification
```bash
sestatus
# Expected: SELinux status: enabled, Current mode: enforcing
```

> **Warning**: Only set SELinux to permissive mode for troubleshooting in non-production environments:
> ```bash
> # FOR NON-PRODUCTION ONLY:
> sudo setenforce 0
> ```

---

### 6. System Limits Configuration (ulimits)

#### Why This Is Needed
- **Default Linux limits (1024 open files) are insufficient for production**
- RabbitMQ/Redis require elevated file descriptor limits to handle thousands of concurrent connections
- Each client connection consumes file descriptors
- Insufficient limits cause "too many open files" errors and connection failures
- Recommended limit: **65536** for general use, higher for large-scale deployments

#### How to Configure

**Step 1: Configure system-wide limits**
```bash
sudo vi /etc/security/limits.conf
```

Add the following lines:
```
# RabbitMQ limits
rabbitmq soft nofile 65536
rabbitmq hard nofile 65536
rabbitmq soft nproc 4096
rabbitmq hard nproc 4096

# Redis limits
redis soft nofile 65536
redis hard nofile 65536
redis soft nproc 4096
redis hard nproc 4096
```

**Step 2: Configure systemd service limits for RabbitMQ**
```bash
sudo mkdir -p /etc/systemd/system/rabbitmq-server.service.d/
sudo vi /etc/systemd/system/rabbitmq-server.service.d/limits.conf
```

Add:
```ini
[Service]
LimitNOFILE=65536
LimitNPROC=4096
```

**Step 3: Configure systemd service limits for Redis**
```bash
sudo mkdir -p /etc/systemd/system/redis.service.d/
sudo vi /etc/systemd/system/redis.service.d/limits.conf
```

Add:
```ini
[Service]
LimitNOFILE=65536
LimitNPROC=4096
```

**Step 4: Reload systemd**
```bash
sudo systemctl daemon-reload
```

#### Verification
```bash
# After starting the service, verify limits:
# For RabbitMQ:
cat /proc/$(pidof beam.smp)/limits | grep "open files"

# For Redis:
cat /proc/$(pidof redis-server)/limits | grep "open files"

# Expected output: Max open files  65536  65536  files
```

---

### 7. Time Synchronization (NTP/Chrony)

#### Why This Is Needed
- **Accurate time synchronization is critical for cluster operations**
- Prevents cluster instability caused by time drift between nodes
- Required for message ordering and event correlation
- SSL/TLS certificate validation depends on accurate time
- Log correlation across nodes requires synchronized timestamps

#### How to Configure
```bash
# Install chrony (NTP client)
sudo dnf install -y chrony

# Configure NTP servers (use your corporate NTP servers)
sudo vi /etc/chrony.conf
```

Modify server lines:
```
server ntp1.example.com iburst
server ntp2.example.com iburst
# Or use public NTP pools:
# pool pool.ntp.org iburst
```

Enable and start the service:
```bash
sudo systemctl enable chronyd
sudo systemctl start chronyd
```

#### Verification
```bash
# Check synchronization status
chronyc tracking

# View NTP sources
chronyc sources -v

# Check system time
timedatectl

# Compare time across all cluster nodes
# Time should be within milliseconds
```

---

### 8. Disable Transparent Huge Pages (THP)

#### Why This Is Needed
- **THP can cause memory allocation latency spikes**
- Both RabbitMQ and Redis recommend disabling THP for better performance
- Reduces memory fragmentation issues
- Improves latency consistency for time-sensitive applications
- Prevents unpredictable memory allocation delays

#### How to Configure

**Immediate (temporary) disable:**
```bash
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/defrag
```

**Persistent across reboots:**
```bash
sudo vi /etc/rc.d/rc.local
```

Add:
```bash
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
```

Make executable:
```bash
sudo chmod +x /etc/rc.d/rc.local
```

**Alternative: Using tuned profile**
```bash
# Create custom tuned profile
sudo mkdir -p /etc/tuned/no-thp
sudo vi /etc/tuned/no-thp/tuned.conf
```

Add:
```ini
[main]
include=throughput-performance

[vm]
transparent_hugepages=never
```

Apply:
```bash
sudo tuned-adm profile no-thp
```

#### Verification
```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
# Expected: always madvise [never]
```

---

## RabbitMQ-Specific Prerequisites

These settings are specific to RabbitMQ installations.

---

### 9. Erlang/OTP Runtime

#### Why This Is Needed
- RabbitMQ is built on the Erlang/OTP platform
- Erlang provides the concurrency model and clustering capabilities
- Version compatibility is critical (RabbitMQ 4.1.x requires Erlang 26.x)

#### How to Configure
```bash
# Install Erlang from RabbitMQ's repository
sudo dnf install -y https://github.com/rabbitmq/erlang-rpm/releases/download/v26.2.1/erlang-26.2.1-1.el8.x86_64.rpm

# Alternatively, use PackageCloud repository:
curl -s https://packagecloud.io/install/repositories/rabbitmq/erlang/script.rpm.sh | sudo bash
sudo dnf install -y erlang
```

#### Verification
```bash
erl -version
# Expected: Erlang (SMP,ASYNC_THREADS) (BEAM) emulator version 14.x
```

---

### 10. Erlang Cookie Security

#### Why This Is Needed
- **The Erlang cookie is the authentication mechanism between cluster nodes**
- All nodes in a cluster must share the same cookie
- Cookie permissions must be restricted (mode 400)
- Incorrect cookie causes "connection refused" between nodes

#### How to Verify
```bash
# Check cookie exists and has correct permissions
ls -la /var/lib/rabbitmq/.erlang.cookie
# Expected: -r-------- 1 rabbitmq rabbitmq

# Fix permissions if needed
sudo chmod 400 /var/lib/rabbitmq/.erlang.cookie
sudo chown rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie
```

---

## Redis-Specific Prerequisites

These settings are specific to Redis installations.

---

### 11. Memory Overcommit Setting

#### Why This Is Needed
- **Redis uses fork() for background saves (RDB/AOF)**
- Without memory overcommit, fork() may fail if not enough memory is available
- This causes background save failures and potential data loss
- Redis logs warning: "Background saving failed with error"

#### How to Configure
```bash
# Set immediately
sudo sysctl vm.overcommit_memory=1

# Make persistent
echo "vm.overcommit_memory = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

#### Verification
```bash
cat /proc/sys/vm/overcommit_memory
# Expected: 1
```

---

### 12. Somaxconn and TCP Backlog

#### Why This Is Needed
- **somaxconn controls the maximum number of pending connections**
- Default value (128) is too low for high-traffic Redis servers
- Low value causes connection drops under load
- Redis logs warning: "The TCP backlog setting of 511 cannot be enforced"

#### How to Configure
```bash
# Set immediately
sudo sysctl net.core.somaxconn=65535
sudo sysctl net.ipv4.tcp_max_syn_backlog=65535

# Make persistent
echo "net.core.somaxconn = 65535" | sudo tee -a /etc/sysctl.conf
echo "net.ipv4.tcp_max_syn_backlog = 65535" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

#### Verification
```bash
cat /proc/sys/net/core/somaxconn
# Expected: 65535
```

---

### 13. Disable Swap (Optional but Recommended)

#### Why This Is Needed
- **Redis should run entirely in memory for best performance**
- Swapping causes severe latency spikes
- If Redis data is swapped to disk, response times become unpredictable
- Better to have Redis fail fast than to run slowly with swapping

#### How to Configure

**Option 1: Reduce swappiness (recommended)**
```bash
# Set immediately
sudo sysctl vm.swappiness=1

# Make persistent
echo "vm.swappiness = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

**Option 2: Disable swap entirely (use with caution)**
```bash
# Disable all swap
sudo swapoff -a

# Remove swap entries from /etc/fstab to persist
```

#### Verification
```bash
cat /proc/sys/vm/swappiness
# Expected: 1 (or 0 if swap is disabled)
```

---

## Quick Reference Checklist

Use this checklist to verify all prerequisites are configured:

### Common Settings (Both RabbitMQ and Redis)

| # | Setting | Command to Verify | Expected Result |
|---|---------|-------------------|-----------------|
| 1 | OS Updated | `cat /etc/redhat-release` | RHEL 8.x |
| 2 | Required packages | `rpm -qa \| grep socat` | Package installed |
| 3 | Hostname set | `hostname -f` | FQDN returned |
| 4 | DNS resolution | `ping node1.example.com` | Successful ping |
| 5 | Firewall ports | `firewall-cmd --list-all` | Ports listed |
| 6 | SELinux | `getenforce` | Enforcing (with policies) |
| 7 | File limits | `ulimit -n` | 65536 |
| 8 | Time sync | `chronyc tracking` | Synchronized |
| 9 | THP disabled | `cat /sys/.../transparent_hugepage/enabled` | [never] |

### RabbitMQ-Specific

| # | Setting | Command to Verify | Expected Result |
|---|---------|-------------------|-----------------|
| 10 | Erlang installed | `erl -version` | Version 26.x |
| 11 | Cookie permissions | `ls -la /var/lib/rabbitmq/.erlang.cookie` | -r-------- rabbitmq |

### Redis-Specific

| # | Setting | Command to Verify | Expected Result |
|---|---------|-------------------|-----------------|
| 12 | Memory overcommit | `cat /proc/sys/vm/overcommit_memory` | 1 |
| 13 | Somaxconn | `cat /proc/sys/net/core/somaxconn` | 65535 |
| 14 | Swappiness | `cat /proc/sys/vm/swappiness` | 1 or 0 |

---

## Summary of All sysctl Settings

Create a single file with all kernel parameter changes:

```bash
sudo vi /etc/sysctl.d/99-rabbitmq-redis.conf
```

Add all settings:
```ini
# Memory overcommit for Redis
vm.overcommit_memory = 1

# Reduce swappiness
vm.swappiness = 1

# Increase connection backlog
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
```

Apply:
```bash
sudo sysctl -p /etc/sysctl.d/99-rabbitmq-redis.conf
```

---

## Troubleshooting Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| "Too many open files" | ulimit not configured | Configure file limits in limits.conf and systemd |
| "Connection refused" between nodes | Firewall blocking ports | Open required ports in firewalld |
| Cluster nodes can't join | Hostname resolution failure | Configure /etc/hosts or DNS |
| "Background save failed" (Redis) | Memory overcommit disabled | Set vm.overcommit_memory=1 |
| High latency spikes | THP enabled | Disable Transparent Huge Pages |
| Time-based errors | NTP not configured | Configure and enable chronyd |
| SELinux denials | Missing policies | Add SELinux policies or investigate audit logs |

---

## Contact and Support

For questions about these configurations, contact:
- Infrastructure Team: [team-email]
- Documentation Owner: [owner-name]

---

*Document Version: 1.0 | Last Updated: January 2026*
