# Redis Cluster Failover

## Scenarios, Test Cases, and Procedures

### Three-Node Cluster Configuration with Redis Sentinel

---

## Document Information

| Field | Value |
|-------|-------|
| Document Version | 1.0 |
| Last Updated | January 2026 |
| Redis Version | 8.x |
| Operating System | Red Hat Enterprise Linux 8.x |
| Document Type | Test Procedures and Scenarios |
| Target Audience | DevOps Engineers, SREs, System Administrators |

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Test Environment Setup](#2-test-environment-setup)
3. [Prerequisites and Assumptions](#3-prerequisites-and-assumptions)
4. [Redis Architecture Overview](#4-redis-architecture-overview)
5. [Failover Scenarios Overview](#5-failover-scenarios-overview)
6. [Test Case 1: Master Node Failure](#6-test-case-1-master-node-failure)
7. [Test Case 2: Replica Node Failure](#7-test-case-2-replica-node-failure)
8. [Test Case 3: Sentinel Node Failure](#8-test-case-3-sentinel-node-failure)
9. [Test Case 4: Network Partition (Split-Brain)](#9-test-case-4-network-partition-split-brain)
10. [Test Case 5: Graceful Node Shutdown](#10-test-case-5-graceful-node-shutdown)
11. [Test Case 6: Multiple Node Failure](#11-test-case-6-multiple-node-failure)
12. [Test Case 7: Disk Space Exhaustion](#12-test-case-7-disk-space-exhaustion)
13. [Test Case 8: Memory Exhaustion](#13-test-case-8-memory-exhaustion)
14. [Test Case 9: Rolling Restart](#14-test-case-9-rolling-restart)
15. [Test Case 10: Data Replication Validation](#15-test-case-10-data-replication-validation)
16. [Test Case 11: Client Connection Failover](#16-test-case-11-client-connection-failover)
17. [Test Case 12: Persistence and Recovery](#17-test-case-12-persistence-and-recovery)
18. [Monitoring and Validation Commands](#18-monitoring-and-validation-commands)
19. [Common Issues and Troubleshooting](#19-common-issues-and-troubleshooting)
20. [Recovery Procedures](#20-recovery-procedures)
21. [Best Practices for Production](#21-best-practices-for-production)
22. [Test Results Template](#22-test-results-template)

---

## 1. Introduction

This document provides comprehensive failover scenarios and test cases for a three-node Redis deployment with Redis Sentinel for high availability. It includes detailed step-by-step procedures to validate automatic failover, data replication, persistence, and disaster recovery capabilities.

### Purpose of Failover Testing

- Validate automatic failover mechanisms work as expected
- Ensure zero or minimal data loss during node failures
- Test cluster behavior under various failure conditions
- Verify client reconnection and data continuity
- Identify potential weaknesses in the HA configuration
- Document recovery procedures for production incidents
- Build confidence in the high availability setup
- Validate Sentinel quorum and leader election

---

## 2. Test Environment Setup

### Required Infrastructure

| Component | Specification | Purpose |
|-----------|---------------|---------|
| 3 RHEL 8 Servers | 4 CPU, 8GB RAM each | Redis nodes (1 Master, 2 Replicas) |
| 3 Sentinel Processes | Co-located or separate | Monitoring and failover orchestration |
| Test Client Machine | 2 CPU, 4GB RAM | Application testing |
| Network Access | All nodes can communicate | Replication and Sentinel communication |
| Monitoring Tools | Redis CLI, redis-stat | Cluster status monitoring |

### Cluster Node Details

| Hostname | IP Address | Redis Role | Sentinel |
|----------|------------|------------|----------|
| redis-node1 | 192.168.1.101 | Master | Yes (Sentinel 1) |
| redis-node2 | 192.168.1.102 | Replica | Yes (Sentinel 2) |
| redis-node3 | 192.168.1.103 | Replica | Yes (Sentinel 3) |

### Port Configuration

| Port | Service | Purpose |
|------|---------|---------|
| 6379 | Redis | Client connections and replication |
| 26379 | Sentinel | Sentinel communication and client discovery |

### Software Versions

```
Redis Version: 8.x
Operating System: RHEL 8.x
```

---

## 3. Prerequisites and Assumptions

### Prerequisites

- Redis is installed and running on all 3 nodes
- Redis Sentinel is configured and running on all nodes
- Master-Replica replication is established and synchronized
- Sentinel quorum is configured (minimum 2 out of 3)
- Administrative access to all cluster nodes
- Monitoring tools are configured and accessible
- Test data is loaded for validation
- Test client applications are ready

### Initial Cluster Verification

```bash
# Check Redis server status on all nodes
redis-cli -h 192.168.1.101 -p 6379 ping
redis-cli -h 192.168.1.102 -p 6379 ping
redis-cli -h 192.168.1.103 -p 6379 ping

# Check replication status on master
redis-cli -h 192.168.1.101 -p 6379 info replication

# Expected output:
# role:master
# connected_slaves:2
# slave0:ip=192.168.1.102,port=6379,state=online,offset=xxx,lag=0
# slave1:ip=192.168.1.103,port=6379,state=online,offset=xxx,lag=0

# Check Sentinel status
redis-cli -h 192.168.1.101 -p 26379 sentinel masters

# Check Sentinel's view of replicas
redis-cli -h 192.168.1.101 -p 26379 sentinel replicas mymaster
```

---

## 4. Redis Architecture Overview

### Master-Replica with Sentinel Configuration

In a Redis high availability setup with Sentinel:
- **One Master**: Handles all write operations
- **Two Replicas**: Maintain synchronized copies of data, handle read operations
- **Three Sentinels**: Monitor nodes, perform automatic failover, provide service discovery

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Master** | Primary node that accepts writes and replicates to replicas |
| **Replica** | Secondary nodes that maintain copies of master data |
| **Sentinel** | Monitoring process that handles automatic failover |
| **Quorum** | Minimum Sentinels required to agree on failover (typically majority) |
| **Failover** | Automatic promotion of a replica to master when master fails |
| **SDOWN** | Subjectively Down - single Sentinel thinks node is down |
| **ODOWN** | Objectively Down - quorum of Sentinels agree node is down |

### Sentinel Configuration

```bash
# View current Sentinel configuration
redis-cli -h 192.168.1.101 -p 26379 sentinel master mymaster

# Key configuration parameters:
# sentinel monitor mymaster 192.168.1.101 6379 2
# sentinel down-after-milliseconds mymaster 5000
# sentinel failover-timeout mymaster 60000
# sentinel parallel-syncs mymaster 1
```

### Replication Topology

```
                    ┌─────────────────┐
                    │   Master        │
                    │ redis-node1     │
                    │ 192.168.1.101   │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
              ▼                             ▼
    ┌─────────────────┐           ┌─────────────────┐
    │   Replica 1     │           │   Replica 2     │
    │ redis-node2     │           │ redis-node3     │
    │ 192.168.1.102   │           │ 192.168.1.103   │
    └─────────────────┘           └─────────────────┘

    Sentinel 1, 2, 3 run on each node monitoring all Redis instances
```

---

## 5. Failover Scenarios Overview

### Test Scenarios Summary

| Scenario | Type | Impact Level | Expected Recovery Time |
|----------|------|--------------|------------------------|
| Master Node Failure | Hard Failure | High | 10-30 seconds |
| Replica Node Failure | Hard Failure | Low | Immediate (no impact) |
| Sentinel Node Failure | Monitoring | Low | Immediate (if quorum maintained) |
| Network Partition | Split-Brain | Critical | 30-60 seconds |
| Graceful Shutdown | Planned | Low | 10-15 seconds |
| Multiple Node Failure | Catastrophic | Critical | Manual intervention |
| Disk Space Exhaustion | Resource | High | After disk cleanup |
| Memory Exhaustion | Resource | High | After memory freed |
| Rolling Restart | Maintenance | None | N/A |
| Replication Validation | Validation | None | N/A |
| Client Failover | Application | Low | < 10 seconds |
| Persistence Recovery | Data Integrity | Medium | Depends on data size |

---

## 6. Test Case 1: Master Node Failure

### Objective
Simulate a catastrophic failure of the Redis master node and verify automatic failover to a replica with minimal data loss and downtime.

### Severity: HIGH

### Expected Outcome
Automatic failover within 10-30 seconds (based on Sentinel configuration)

### Pre-Test Setup

```bash
# 1. Verify current master
redis-cli -h 192.168.1.101 -p 26379 sentinel get-master-addr-by-name mymaster
# Expected: 192.168.1.101 6379

# 2. Load test data
redis-cli -h 192.168.1.101 -p 6379 SET test:failover:key "initial_value"
redis-cli -h 192.168.1.101 -p 6379 SET test:counter 1000

# 3. Start continuous write test (in background)
# This script writes incrementing values to detect data loss
while true; do
  redis-cli -h 192.168.1.101 -p 6379 INCR test:counter 2>/dev/null
  sleep 0.1
done &

# 4. Record pre-test state
redis-cli -h 192.168.1.101 -p 6379 GET test:counter
redis-cli -h 192.168.1.101 -p 6379 info replication
```

### Test Steps

#### Step 1: Record Initial State

```bash
# On any node, check Sentinel's view
redis-cli -h 192.168.1.101 -p 26379 sentinel master mymaster

# Record key metrics:
# - Master IP/Port
# - Number of replicas
# - Quorum setting
# - down-after-milliseconds
```

#### Step 2: Verify Replication Lag

```bash
# Check replication is synchronized
redis-cli -h 192.168.1.101 -p 6379 info replication | grep -E "slave|offset|lag"

# All replicas should have lag=0 or lag=1
```

#### Step 3: Simulate Hard Failure (Kill Master)

```bash
# On redis-node1 (master):
# Method 1: Stop Redis service
sudo systemctl stop redis

# Method 2: Force kill process (more realistic failure simulation)
sudo kill -9 $(pidof redis-server)

# Method 3: Crash simulation
sudo killall -SEGV redis-server
```

#### Step 4: Monitor Failover (On Another Node)

```bash
# Watch Sentinel logs in real-time
sudo tail -f /var/log/redis/sentinel.log

# Or monitor via Sentinel CLI
watch -n 1 "redis-cli -h 192.168.1.102 -p 26379 sentinel master mymaster | grep -E 'ip|port|flags|num-slaves'"

# Key events to observe:
# +sdown master mymaster 192.168.1.101 6379
# +odown master mymaster 192.168.1.101 6379 #quorum 2/2
# +try-failover master mymaster 192.168.1.101 6379
# +elected-leader master mymaster 192.168.1.101 6379
# +promoted-slave slave 192.168.1.102:6379
# +switch-master mymaster 192.168.1.101 6379 192.168.1.102 6379
```

#### Step 5: Verify New Master Election

```bash
# Check new master address
redis-cli -h 192.168.1.102 -p 26379 sentinel get-master-addr-by-name mymaster
# Should return one of the replica IPs (e.g., 192.168.1.102 6379)

# Verify new master accepts writes
redis-cli -h 192.168.1.102 -p 6379 SET test:new_master "verified"
redis-cli -h 192.168.1.102 -p 6379 GET test:new_master
```

#### Step 6: Check Replica Reconfiguration

```bash
# Verify remaining replica is now following new master
redis-cli -h 192.168.1.103 -p 6379 info replication
# Should show: role:slave, master_host:192.168.1.102
```

#### Step 7: Verify Data Integrity

```bash
# Check test data on new master
redis-cli -h 192.168.1.102 -p 6379 GET test:failover:key
# Expected: "initial_value"

# Check counter value (may have some loss due to async replication)
redis-cli -h 192.168.1.102 -p 6379 GET test:counter

# Compare with expected value to measure data loss
```

#### Step 8: Monitor Sentinel Consensus

```bash
# All Sentinels should agree on new master
redis-cli -h 192.168.1.101 -p 26379 sentinel get-master-addr-by-name mymaster 2>/dev/null
redis-cli -h 192.168.1.102 -p 26379 sentinel get-master-addr-by-name mymaster
redis-cli -h 192.168.1.103 -p 26379 sentinel get-master-addr-by-name mymaster
# All should return the same new master IP
```

### Expected Results

- Master node is detected as down within `down-after-milliseconds` (default 5 seconds)
- Sentinels reach quorum and mark master as ODOWN
- One replica is promoted to master automatically
- Remaining replica reconfigures to follow new master
- Data is preserved (minimal loss from async replication)
- Write operations resume on new master
- Sentinel configuration is updated automatically
- No manual intervention required

### Post-Test Recovery

```bash
# On the failed node (redis-node1):
sudo systemctl start redis

# The node will automatically:
# 1. Come up as a replica
# 2. Connect to the new master
# 3. Sync data from master

# Verify node rejoined as replica
redis-cli -h 192.168.1.101 -p 6379 info replication
# Expected: role:slave, master_host:192.168.1.102

# Verify Sentinel recognizes the node
redis-cli -h 192.168.1.102 -p 26379 sentinel replicas mymaster
```

### Data Loss Assessment

> **Important**: Redis uses asynchronous replication by default. Data written to the master but not yet replicated to any replica will be lost during a hard failure. To minimize data loss:
> - Use `WAIT` command for critical writes
> - Configure `min-replicas-to-write` and `min-replicas-max-lag`
> - Enable AOF with `appendfsync everysec` or `appendfsync always`

---

## 7. Test Case 2: Replica Node Failure

### Objective
Verify that failure of a replica node has minimal impact on cluster operations and data availability continues normally.

### Severity: LOW

### Expected Outcome
No impact on write availability; reduced read capacity

### Test Steps

#### Step 1: Identify Replica Node

```bash
# Check replication topology
redis-cli -h 192.168.1.101 -p 6379 info replication

# Choose a replica (e.g., redis-node2 at 192.168.1.102)
```

#### Step 2: Stop Replica Node

```bash
# On redis-node2:
sudo systemctl stop redis

# Or force kill:
sudo kill -9 $(pidof redis-server)
```

#### Step 3: Verify Master Detects Replica Loss

```bash
# On master (redis-node1):
redis-cli -h 192.168.1.101 -p 6379 info replication

# Expected:
# connected_slaves:1 (reduced from 2)
# slave0 should only show redis-node3
```

#### Step 4: Verify Write Operations Continue

```bash
# Write operations should work normally
redis-cli -h 192.168.1.101 -p 6379 SET test:replica:failure "test_value"
redis-cli -h 192.168.1.101 -p 6379 GET test:replica:failure
# Expected: "test_value"
```

#### Step 5: Check Sentinel Status

```bash
# Sentinels will detect replica as down
redis-cli -h 192.168.1.101 -p 26379 sentinel replicas mymaster

# Failed replica should show s_down flag
```

#### Step 6: Verify Remaining Replica

```bash
# Remaining replica should be synchronized
redis-cli -h 192.168.1.103 -p 6379 info replication
# Should show: role:slave, master_link_status:up

# Data should be replicated
redis-cli -h 192.168.1.103 -p 6379 GET test:replica:failure
# Expected: "test_value"
```

#### Step 7: Restart Failed Replica

```bash
# On redis-node2:
sudo systemctl start redis

# Replica will automatically reconnect and sync
```

#### Step 8: Verify Resynchronization

```bash
# Check replica reconnected
redis-cli -h 192.168.1.101 -p 6379 info replication
# Expected: connected_slaves:2

# Verify data is synchronized
redis-cli -h 192.168.1.102 -p 6379 GET test:replica:failure
# Expected: "test_value"
```

### Expected Results

- No impact on write operations
- Master continues operating normally
- Remaining replica maintains synchronization
- Failed replica rejoins and resynchronizes automatically
- No data loss occurs
- Sentinel updates replica status correctly

> **Note**: Replica failure is the least impactful failure scenario. The cluster can lose all replicas and still accept writes (though this is not recommended for production).

---

## 8. Test Case 3: Sentinel Node Failure

### Objective
Verify that failure of a Sentinel node does not impact Redis operations and failover capability is maintained with remaining Sentinels.

### Severity: LOW (if quorum maintained)

### Expected Outcome
No impact on Redis operations; failover still possible with remaining Sentinels

### Test Steps

#### Step 1: Check Initial Sentinel Status

```bash
# Verify all Sentinels are running
redis-cli -h 192.168.1.101 -p 26379 ping
redis-cli -h 192.168.1.102 -p 26379 ping
redis-cli -h 192.168.1.103 -p 26379 ping

# Check Sentinel view of cluster
redis-cli -h 192.168.1.101 -p 26379 sentinel master mymaster
# Note: num-other-sentinels should be 2
```

#### Step 2: Stop One Sentinel

```bash
# On redis-node2:
sudo systemctl stop redis-sentinel

# Or if Sentinel is part of Redis service:
sudo kill $(cat /var/run/redis/sentinel.pid)
```

#### Step 3: Verify Remaining Sentinels Detect Failure

```bash
# Check from another Sentinel
redis-cli -h 192.168.1.101 -p 26379 sentinel sentinels mymaster

# Failed Sentinel should show s_down flag
# num-other-sentinels should be 1
```

#### Step 4: Verify Redis Operations Continue

```bash
# Redis should work normally
redis-cli -h 192.168.1.101 -p 6379 SET test:sentinel:failure "test"
redis-cli -h 192.168.1.101 -p 6379 GET test:sentinel:failure
```

#### Step 5: Test Failover Capability (Optional)

```bash
# With 2 Sentinels remaining (quorum=2), failover should still work
# Trigger manual failover to verify:
redis-cli -h 192.168.1.101 -p 26379 sentinel failover mymaster

# Monitor failover progress
redis-cli -h 192.168.1.101 -p 26379 sentinel get-master-addr-by-name mymaster
```

#### Step 6: Restart Failed Sentinel

```bash
# On redis-node2:
sudo systemctl start redis-sentinel

# Verify Sentinel rejoined
redis-cli -h 192.168.1.101 -p 26379 sentinel sentinels mymaster
# Should show 2 other Sentinels again
```

### Expected Results

- No impact on Redis read/write operations
- Remaining Sentinels maintain quorum (2 out of 3)
- Automatic failover still possible
- Failed Sentinel rejoins cluster automatically
- No data loss occurs

> **Warning**: If more than one Sentinel fails and quorum is lost, automatic failover becomes impossible. Manual intervention would be required.

---

## 9. Test Case 4: Network Partition (Split-Brain)

### Objective
Test cluster behavior when network connectivity is lost between nodes, creating a partition scenario. Verify partition handling and data consistency.

### Severity: CRITICAL

### Expected Outcome
Master on minority side stops accepting writes; failover occurs on majority side

### Prerequisites

```bash
# Verify current configuration
redis-cli -h 192.168.1.101 -p 6379 config get min-replicas-to-write
redis-cli -h 192.168.1.101 -p 6379 config get min-replicas-max-lag

# Recommended settings for partition safety:
# min-replicas-to-write = 1 (at least 1 replica must acknowledge)
# min-replicas-max-lag = 10 (replica must be within 10 seconds)
```

> **Warning**: Network partitions are dangerous. Improper handling can lead to data inconsistency and split-brain scenarios where two masters exist simultaneously.

### Test Steps

#### Step 1: Record Pre-Partition State

```bash
# Document current master and replication status
redis-cli -h 192.168.1.101 -p 6379 info replication
redis-cli -h 192.168.1.101 -p 26379 sentinel master mymaster

# Record current data
redis-cli -h 192.168.1.101 -p 6379 GET test:counter
```

#### Step 2: Create Network Partition

```bash
# Method: Using iptables to isolate master (node1) from replicas (node2, node3)

# On redis-node1 (master), block traffic from other nodes:
sudo iptables -A INPUT -s 192.168.1.102 -j DROP
sudo iptables -A INPUT -s 192.168.1.103 -j DROP
sudo iptables -A OUTPUT -d 192.168.1.102 -j DROP
sudo iptables -A OUTPUT -d 192.168.1.103 -j DROP

# This creates partition: [node1] | [node2, node3]
# node1 (old master) is isolated
# node2, node3 (replicas + 2 Sentinels) can reach each other
```

#### Step 3: Observe Partition Detection

```bash
# On node1 (isolated):
redis-cli -h 192.168.1.101 -p 6379 info replication
# connected_slaves will drop to 0

# On node2 or node3:
redis-cli -h 192.168.1.102 -p 26379 sentinel master mymaster
# Master should show as s_down or o_down
```

#### Step 4: Monitor Failover on Majority Side

```bash
# On node2 or node3, watch for failover:
sudo tail -f /var/log/redis/sentinel.log

# Key events:
# +sdown master mymaster 192.168.1.101 6379
# +odown master mymaster 192.168.1.101 6379 #quorum 2/2
# +failover-state-select-slave master mymaster
# +selected-slave slave 192.168.1.102:6379
# +switch-master mymaster 192.168.1.101 6379 192.168.1.102 6379

# Verify new master
redis-cli -h 192.168.1.102 -p 26379 sentinel get-master-addr-by-name mymaster
```

#### Step 5: Test Write Behavior on Both Sides

```bash
# On isolated old master (node1):
redis-cli -h 192.168.1.101 -p 6379 SET partition:old_master "data_on_old"
# Should FAIL if min-replicas-to-write is configured
# Error: NOREPLICAS Not enough good replicas to write

# On new master (node2):
redis-cli -h 192.168.1.102 -p 6379 SET partition:new_master "data_on_new"
# Should SUCCEED
```

#### Step 6: Restore Network Connectivity

```bash
# On node1, remove iptables rules:
sudo iptables -D INPUT -s 192.168.1.102 -j DROP
sudo iptables -D INPUT -s 192.168.1.103 -j DROP
sudo iptables -D OUTPUT -d 192.168.1.102 -j DROP
sudo iptables -D OUTPUT -d 192.168.1.103 -j DROP

# Or flush all iptables rules:
sudo iptables -F
```

#### Step 7: Verify Partition Healing

```bash
# Old master (node1) should:
# 1. Detect it's no longer master
# 2. Convert to replica
# 3. Sync with new master

redis-cli -h 192.168.1.101 -p 6379 info replication
# Expected: role:slave, master_host:192.168.1.102

# Verify cluster is healthy
redis-cli -h 192.168.1.102 -p 6379 info replication
# Expected: connected_slaves:2
```

#### Step 8: Check Data Consistency

```bash
# Verify data on all nodes
redis-cli -h 192.168.1.102 -p 6379 GET partition:new_master
redis-cli -h 192.168.1.101 -p 6379 GET partition:new_master
redis-cli -h 192.168.1.103 -p 6379 GET partition:new_master
# All should return: "data_on_new"

# Data written to old master during partition (if min-replicas-to-write=0) may be lost
redis-cli -h 192.168.1.102 -p 6379 GET partition:old_master
# May return: (nil) - data lost
```

### Expected Results

- Partition is detected by Sentinels within `down-after-milliseconds`
- Majority partition (2 nodes) elects new master
- Old master (minority) stops accepting writes (if properly configured)
- After partition heals, old master becomes replica
- Data written to majority side is preserved
- Data written to isolated old master may be lost

> **Critical Configuration**: To prevent split-brain data loss, configure:
> ```
> min-replicas-to-write 1
> min-replicas-max-lag 10
> ```
> This ensures master stops accepting writes when it can't reach any replica.

---

## 10. Test Case 5: Graceful Node Shutdown

### Objective
Verify that planned maintenance shutdown of a node (master or replica) allows proper failover and data synchronization without data loss.

### Severity: LOW

### Expected Outcome
Zero data loss, controlled failover for master shutdown

### Test Steps

#### Step 1: Graceful Master Shutdown

```bash
# Verify current state
redis-cli -h 192.168.1.101 -p 6379 info replication
redis-cli -h 192.168.1.101 -p 6379 DBSIZE

# Option 1: Trigger manual failover first (recommended)
redis-cli -h 192.168.1.101 -p 26379 sentinel failover mymaster

# Wait for failover to complete
sleep 10
redis-cli -h 192.168.1.102 -p 26379 sentinel get-master-addr-by-name mymaster

# Then stop the old master (now a replica)
sudo systemctl stop redis
```

```bash
# Option 2: Use Redis SHUTDOWN command (cleanest)
# This triggers proper data saving and notifies Sentinels
redis-cli -h 192.168.1.101 -p 6379 SHUTDOWN SAVE
```

#### Step 2: Verify Failover Occurred

```bash
# Check new master
redis-cli -h 192.168.1.102 -p 26379 sentinel get-master-addr-by-name mymaster

# Verify replication
redis-cli -h 192.168.1.102 -p 6379 info replication
```

#### Step 3: Verify Data Integrity

```bash
# All data should be preserved
redis-cli -h 192.168.1.102 -p 6379 DBSIZE
# Should match pre-shutdown count

redis-cli -h 192.168.1.102 -p 6379 GET test:key
# Should return expected value
```

#### Step 4: Restart Node

```bash
# On shutdown node:
sudo systemctl start redis

# Node will join as replica
redis-cli -h 192.168.1.101 -p 6379 info replication
# Expected: role:slave
```

#### Step 5: Graceful Replica Shutdown

```bash
# For replica, simple shutdown is sufficient
redis-cli -h 192.168.1.103 -p 6379 SHUTDOWN SAVE

# Or via systemctl
sudo systemctl stop redis
```

### Expected Results

- No data loss with graceful shutdown
- Manual failover before master shutdown ensures clean transition
- SHUTDOWN SAVE ensures data is persisted
- Node rejoins as replica after restart
- Replication resumes automatically

> **Best Practice**: Always use `SHUTDOWN SAVE` or trigger manual failover before planned master maintenance.

---

## 11. Test Case 6: Multiple Node Failure

### Objective
Test cluster behavior when multiple nodes fail simultaneously or in quick succession. This is a catastrophic scenario requiring manual intervention.

### Severity: CRITICAL

### Expected Outcome
Cluster unavailable if majority fails; manual recovery required

> **Danger**: This test will make Redis unavailable. Only perform in test/staging environment.

### Scenario A: Two Replicas Fail (Master Survives)

```bash
# Stop both replicas
# On redis-node2:
sudo systemctl stop redis

# On redis-node3:
sudo systemctl stop redis

# Master continues operating but with degraded safety
redis-cli -h 192.168.1.101 -p 6379 info replication
# connected_slaves:0

# If min-replicas-to-write > 0, writes will fail
redis-cli -h 192.168.1.101 -p 6379 SET test:key "value"
# Error: NOREPLICAS (if configured)
```

### Scenario B: Master + One Replica Fail

```bash
# Stop master
sudo systemctl stop redis  # on redis-node1

# Stop one replica
sudo systemctl stop redis  # on redis-node2

# Remaining node (redis-node3) has:
# - 1 Redis server
# - 1 Sentinel
# Cannot achieve quorum (need 2 Sentinels)
# Cannot failover automatically

# Check Sentinel
redis-cli -h 192.168.1.103 -p 26379 sentinel master mymaster
# Will show master as down but cannot failover
```

### Recovery Procedure

```bash
# 1. Start nodes in order - replicas first
# On redis-node2 (replica):
sudo systemctl start redis

# On redis-node3 (replica):
sudo systemctl start redis

# 2. Start master last
# On redis-node1 (master):
sudo systemctl start redis

# 3. Verify cluster recovery
redis-cli -h 192.168.1.101 -p 6379 info replication
redis-cli -h 192.168.1.101 -p 26379 sentinel master mymaster

# 4. If master changed during outage, verify data consistency
```

### Manual Failover (If Needed)

```bash
# If old master is down and you need to promote a replica manually:

# 1. On the replica you want to promote:
redis-cli -h 192.168.1.102 -p 6379 REPLICAOF NO ONE

# 2. Point other replicas to new master:
redis-cli -h 192.168.1.103 -p 6379 REPLICAOF 192.168.1.102 6379

# 3. Update Sentinel configuration manually or restart Sentinels
```

> **Warning**: Manual failover may result in data loss if the promoted replica was behind the failed master.

---

## 12. Test Case 7: Disk Space Exhaustion

### Objective
Verify Redis behavior when disk space is exhausted and test recovery procedures.

### Severity: HIGH

### Expected Outcome
Redis stops accepting writes; recovers after disk space freed

### Test Steps

#### Step 1: Check Current Disk Usage

```bash
# Check disk space
df -h /var/lib/redis

# Check Redis persistence configuration
redis-cli -h 192.168.1.101 -p 6379 CONFIG GET save
redis-cli -h 192.168.1.101 -p 6379 CONFIG GET appendonly
```

#### Step 2: Simulate Disk Exhaustion

```bash
# Create large file to fill disk
sudo fallocate -l 40G /var/lib/redis/large-file

# Or fill with data through Redis
# (This will trigger RDB/AOF write failures)
```

#### Step 3: Trigger Persistence Operation

```bash
# Trigger background save
redis-cli -h 192.168.1.101 -p 6379 BGSAVE

# Check last save status
redis-cli -h 192.168.1.101 -p 6379 LASTSAVE
redis-cli -h 192.168.1.101 -p 6379 INFO persistence
```

#### Step 4: Observe Behavior

```bash
# Check Redis logs for errors
sudo tail -f /var/log/redis/redis.log

# Expected errors:
# MISCONF Redis is configured to save RDB snapshots
# Can't save in background: fork: Cannot allocate memory
# Background saving error

# Redis may stop accepting writes if:
# stop-writes-on-bgsave-error yes (default)
redis-cli -h 192.168.1.101 -p 6379 SET test:disk:full "value"
# Error: MISCONF
```

#### Step 5: Free Disk Space

```bash
# Remove the test file
sudo rm /var/lib/redis/large-file

# Verify disk space
df -h /var/lib/redis
```

#### Step 6: Verify Recovery

```bash
# Trigger save again
redis-cli -h 192.168.1.101 -p 6379 BGSAVE

# Check persistence status
redis-cli -h 192.168.1.101 -p 6379 INFO persistence
# rdb_last_bgsave_status:ok

# Writes should work again
redis-cli -h 192.168.1.101 -p 6379 SET test:recovered "yes"
```

### Expected Results

- Redis stops accepting writes when disk is full (if `stop-writes-on-bgsave-error yes`)
- Error messages appear in logs
- Writes resume after disk space is freed
- Manual BGSAVE may be needed to clear error state
- No data corruption occurs

---

## 13. Test Case 8: Memory Exhaustion

### Objective
Test Redis behavior under high memory usage and verify eviction policies and recovery.

### Severity: HIGH

### Expected Outcome
Redis applies eviction policy or returns OOM errors

### Test Steps

#### Step 1: Check Memory Configuration

```bash
# Check current memory settings
redis-cli -h 192.168.1.101 -p 6379 CONFIG GET maxmemory
redis-cli -h 192.168.1.101 -p 6379 CONFIG GET maxmemory-policy

# Common policies:
# noeviction - return error on writes when memory limit reached
# allkeys-lru - evict least recently used keys
# volatile-lru - evict LRU keys with TTL set
# allkeys-random - evict random keys
```

#### Step 2: Set Memory Limit for Testing

```bash
# Set a low limit for testing (e.g., 100MB)
redis-cli -h 192.168.1.101 -p 6379 CONFIG SET maxmemory 100mb
redis-cli -h 192.168.1.101 -p 6379 CONFIG SET maxmemory-policy allkeys-lru
```

#### Step 3: Fill Memory

```bash
# Generate large amount of data
for i in $(seq 1 100000); do
    redis-cli -h 192.168.1.101 -p 6379 SET key:$i "$(head -c 1000 /dev/urandom | base64)"
done

# Monitor memory usage
redis-cli -h 192.168.1.101 -p 6379 INFO memory
```

#### Step 4: Observe Eviction Behavior

```bash
# With allkeys-lru:
# Older keys should be evicted to make room

# Check eviction stats
redis-cli -h 192.168.1.101 -p 6379 INFO stats | grep evicted
# evicted_keys: <number>

# With noeviction:
# Writes should fail with OOM error
redis-cli -h 192.168.1.101 -p 6379 SET new:key "value"
# Error: OOM command not allowed when used memory > 'maxmemory'
```

#### Step 5: Verify Read Operations Continue

```bash
# Reads should always work
redis-cli -h 192.168.1.101 -p 6379 GET key:1
# Should return value (if not evicted)
```

#### Step 6: Recovery

```bash
# Option 1: Increase memory limit
redis-cli -h 192.168.1.101 -p 6379 CONFIG SET maxmemory 2gb

# Option 2: Delete data
redis-cli -h 192.168.1.101 -p 6379 FLUSHDB

# Option 3: Let eviction handle it (if eviction policy set)
```

### Expected Results

- Redis respects maxmemory limit
- Eviction policy is applied correctly
- OOM errors returned when appropriate
- Read operations continue regardless of memory state
- No crash or data corruption

---

## 14. Test Case 9: Rolling Restart

### Objective
Perform a rolling restart of all cluster nodes with zero downtime.

### Severity: LOW

### Expected Outcome
Zero downtime, continuous read/write availability

### Test Steps

#### Step 1: Verify Initial State

```bash
# Check cluster health
redis-cli -h 192.168.1.101 -p 6379 info replication
redis-cli -h 192.168.1.101 -p 26379 sentinel master mymaster

# Start continuous test in background
while true; do
    redis-cli -h 192.168.1.101 -p 6379 INCR rolling:restart:counter
    sleep 0.5
done &
```

#### Step 2: Restart Replica 1 (Node 3)

```bash
# On redis-node3:
sudo systemctl restart redis

# Wait for node to come back online
sleep 5

# Verify node is synced
redis-cli -h 192.168.1.103 -p 6379 info replication
# Should show: role:slave, master_link_status:up
```

#### Step 3: Restart Replica 2 (Node 2)

```bash
# On redis-node2:
sudo systemctl restart redis

# Wait for node to come back online
sleep 5

# Verify node is synced
redis-cli -h 192.168.1.102 -p 6379 info replication
```

#### Step 4: Failover Master (Node 1)

```bash
# Trigger manual failover before restarting master
redis-cli -h 192.168.1.101 -p 26379 sentinel failover mymaster

# Wait for failover to complete
sleep 10

# Verify new master
redis-cli -h 192.168.1.102 -p 26379 sentinel get-master-addr-by-name mymaster
```

#### Step 5: Restart Old Master (Now Replica)

```bash
# On redis-node1 (now a replica):
sudo systemctl restart redis

# Verify node rejoined
redis-cli -h 192.168.1.101 -p 6379 info replication
# Should show: role:slave
```

#### Step 6: Verify Zero Data Loss

```bash
# Stop the test counter
# Check final value
redis-cli -h 192.168.1.102 -p 6379 GET rolling:restart:counter

# All increments should be accounted for
```

### Best Practices for Rolling Restart

- Always restart replicas before master
- Trigger manual failover before restarting current master
- Wait for each node to fully sync before proceeding
- Monitor replication lag during process
- Schedule during low-traffic periods
- Have rollback plan ready

---

## 15. Test Case 10: Data Replication Validation

### Objective
Verify that data is correctly replicated from master to all replicas in real-time.

### Severity: MEDIUM

### Expected Outcome
All data correctly replicated with minimal lag

### Test Steps

#### Step 1: Check Replication Status

```bash
# On master:
redis-cli -h 192.168.1.101 -p 6379 info replication

# Key metrics:
# - connected_slaves: 2
# - slave0, slave1: state=online, lag=0
```

#### Step 2: Write Test Data on Master

```bash
# Write various data types
redis-cli -h 192.168.1.101 -p 6379 SET string:key "test_value"
redis-cli -h 192.168.1.101 -p 6379 HSET hash:key field1 value1 field2 value2
redis-cli -h 192.168.1.101 -p 6379 LPUSH list:key item1 item2 item3
redis-cli -h 192.168.1.101 -p 6379 SADD set:key member1 member2 member3
redis-cli -h 192.168.1.101 -p 6379 ZADD zset:key 1 one 2 two 3 three
```

#### Step 3: Verify Data on Replicas

```bash
# On replica 1 (redis-node2):
redis-cli -h 192.168.1.102 -p 6379 GET string:key
redis-cli -h 192.168.1.102 -p 6379 HGETALL hash:key
redis-cli -h 192.168.1.102 -p 6379 LRANGE list:key 0 -1
redis-cli -h 192.168.1.102 -p 6379 SMEMBERS set:key
redis-cli -h 192.168.1.102 -p 6379 ZRANGE zset:key 0 -1 WITHSCORES

# On replica 2 (redis-node3):
# Repeat same commands
```

#### Step 4: Test Replication Lag Under Load

```bash
# Generate high write load on master
redis-benchmark -h 192.168.1.101 -p 6379 -t set -n 100000 -q

# Monitor replication lag during load
watch -n 1 "redis-cli -h 192.168.1.101 -p 6379 info replication | grep lag"
```

#### Step 5: Verify Offset Synchronization

```bash
# Compare master and replica offsets
redis-cli -h 192.168.1.101 -p 6379 info replication | grep master_repl_offset
redis-cli -h 192.168.1.102 -p 6379 info replication | grep slave_repl_offset
redis-cli -h 192.168.1.103 -p 6379 info replication | grep slave_repl_offset

# Offsets should be close (within a few bytes)
```

### Expected Results

- All data types correctly replicated
- Replication lag minimal (< 1 second under normal load)
- Offsets synchronized across nodes
- No data corruption during replication

---

## 16. Test Case 11: Client Connection Failover

### Objective
Verify that client applications can reconnect automatically when the connected node fails.

### Severity: MEDIUM

### Expected Outcome
Clients reconnect within 10 seconds using Sentinel-aware drivers

### Prerequisites

Client application must use Sentinel-aware Redis driver that:
- Queries Sentinel for current master address
- Automatically reconnects on connection failure
- Follows Sentinel redirects

### Example Client Configurations

#### Python (redis-py)

```python
from redis.sentinel import Sentinel

sentinel = Sentinel([
    ('192.168.1.101', 26379),
    ('192.168.1.102', 26379),
    ('192.168.1.103', 26379)
], socket_timeout=0.5)

# Get master connection
master = sentinel.master_for('mymaster', socket_timeout=0.5)

# Get replica connection (for reads)
replica = sentinel.slave_for('mymaster', socket_timeout=0.5)

# Write to master
master.set('key', 'value')

# Read from replica
value = replica.get('key')
```

#### Java (Jedis)

```java
Set<String> sentinels = new HashSet<>();
sentinels.add("192.168.1.101:26379");
sentinels.add("192.168.1.102:26379");
sentinels.add("192.168.1.103:26379");

JedisSentinelPool pool = new JedisSentinelPool("mymaster", sentinels);
try (Jedis jedis = pool.getResource()) {
    jedis.set("key", "value");
}
```

#### Node.js (ioredis)

```javascript
const Redis = require('ioredis');

const redis = new Redis({
    sentinels: [
        { host: '192.168.1.101', port: 26379 },
        { host: '192.168.1.102', port: 26379 },
        { host: '192.168.1.103', port: 26379 }
    ],
    name: 'mymaster'
});
```

### Test Steps

#### Step 1: Start Test Client

```bash
# Run client application with Sentinel configuration
python3 sentinel_client.py
```

#### Step 2: Verify Initial Connection

```bash
# Check client is connected
redis-cli -h 192.168.1.101 -p 6379 client list | grep <client_ip>
```

#### Step 3: Trigger Master Failover

```bash
# Stop current master
sudo systemctl stop redis  # on master node

# Or trigger manual failover
redis-cli -h 192.168.1.101 -p 26379 sentinel failover mymaster
```

#### Step 4: Monitor Client Reconnection

```bash
# Watch client logs for:
# - Connection lost message
# - Sentinel query for new master
# - Reconnection to new master

# Check new master for client connection
redis-cli -h 192.168.1.102 -p 6379 client list
```

#### Step 5: Verify Operations Continue

```bash
# Verify client continues operations on new master
# Check application logs for any errors
```

### Expected Results

- Client detects connection failure within 1-2 seconds
- Client queries Sentinel for new master address
- Successful reconnection within 10 seconds
- Operations resume on new master
- No data loss (with proper error handling)

---

## 17. Test Case 12: Persistence and Recovery

### Objective
Verify data integrity through persistence mechanisms (RDB and AOF) and test recovery from persistence files.

### Severity: MEDIUM

### Expected Outcome
Data fully recovered from persistence files after restart

### Test Steps

#### Step 1: Check Persistence Configuration

```bash
redis-cli -h 192.168.1.101 -p 6379 CONFIG GET save
# RDB snapshots configuration

redis-cli -h 192.168.1.101 -p 6379 CONFIG GET appendonly
# AOF enabled/disabled

redis-cli -h 192.168.1.101 -p 6379 CONFIG GET appendfsync
# AOF sync policy
```

#### Step 2: Write Test Data

```bash
# Write known test data
redis-cli -h 192.168.1.101 -p 6379 SET persistence:test:1 "data_1"
redis-cli -h 192.168.1.101 -p 6379 SET persistence:test:2 "data_2"
redis-cli -h 192.168.1.101 -p 6379 SET persistence:test:3 "data_3"

# Record DBSIZE
redis-cli -h 192.168.1.101 -p 6379 DBSIZE
```

#### Step 3: Force Persistence

```bash
# Trigger RDB save
redis-cli -h 192.168.1.101 -p 6379 BGSAVE

# Wait for completion
redis-cli -h 192.168.1.101 -p 6379 LASTSAVE

# If AOF enabled, trigger rewrite
redis-cli -h 192.168.1.101 -p 6379 BGREWRITEAOF
```

#### Step 4: Simulate Crash and Recovery

```bash
# Force kill Redis (simulating crash)
sudo kill -9 $(pidof redis-server)

# Verify persistence files exist
ls -la /var/lib/redis/dump.rdb
ls -la /var/lib/redis/appendonly.aof

# Restart Redis
sudo systemctl start redis
```

#### Step 5: Verify Data Recovery

```bash
# Check DBSIZE matches pre-crash
redis-cli -h 192.168.1.101 -p 6379 DBSIZE

# Verify test data
redis-cli -h 192.168.1.101 -p 6379 GET persistence:test:1
redis-cli -h 192.168.1.101 -p 6379 GET persistence:test:2
redis-cli -h 192.168.1.101 -p 6379 GET persistence:test:3
```

### Expected Results

- All data persisted in RDB/AOF files
- Data fully recovered after restart
- DBSIZE matches pre-crash count
- No data corruption

> **Note**: Data written after last RDB snapshot or AOF sync may be lost. Use `appendfsync always` for minimal data loss (at performance cost).

---

## 18. Monitoring and Validation Commands

### Essential Redis Commands

| Command | Purpose |
|---------|---------|
| `INFO` | Comprehensive server information |
| `INFO replication` | Replication status |
| `INFO memory` | Memory usage details |
| `INFO persistence` | RDB/AOF status |
| `INFO stats` | Server statistics |
| `CLIENT LIST` | Connected clients |
| `SLOWLOG GET 10` | Recent slow queries |
| `DEBUG SLEEP 0.1` | Test client timeout handling |

### Cluster Status Commands

```bash
# Master information
redis-cli -h 192.168.1.101 -p 6379 info replication

# Sentinel status
redis-cli -h 192.168.1.101 -p 26379 sentinel master mymaster
redis-cli -h 192.168.1.101 -p 26379 sentinel replicas mymaster
redis-cli -h 192.168.1.101 -p 26379 sentinel sentinels mymaster

# Get current master
redis-cli -h 192.168.1.101 -p 26379 sentinel get-master-addr-by-name mymaster

# Check Sentinel state
redis-cli -h 192.168.1.101 -p 26379 sentinel ckquorum mymaster
```

### Health Check Commands

```bash
# Basic connectivity
redis-cli -h 192.168.1.101 -p 6379 PING

# Memory status
redis-cli -h 192.168.1.101 -p 6379 INFO memory | grep -E "used_memory|maxmemory"

# Persistence status
redis-cli -h 192.168.1.101 -p 6379 INFO persistence | grep -E "rdb_|aof_"

# Replication lag
redis-cli -h 192.168.1.101 -p 6379 INFO replication | grep lag

# Client connections
redis-cli -h 192.168.1.101 -p 6379 CLIENT LIST | wc -l
```

### Performance Monitoring

```bash
# Real-time commands
redis-cli -h 192.168.1.101 -p 6379 MONITOR  # (Use briefly - high overhead)

# Command statistics
redis-cli -h 192.168.1.101 -p 6379 INFO commandstats

# Latency monitoring
redis-cli -h 192.168.1.101 -p 6379 --latency
redis-cli -h 192.168.1.101 -p 6379 --latency-history

# Memory analysis
redis-cli -h 192.168.1.101 -p 6379 MEMORY DOCTOR
redis-cli -h 192.168.1.101 -p 6379 MEMORY STATS
```

---

## 19. Common Issues and Troubleshooting

| Issue | Symptoms | Resolution |
|-------|----------|------------|
| Failover not triggering | Master down but no promotion | Check Sentinel quorum; verify network connectivity |
| Replication lag | High lag values in INFO replication | Check network bandwidth; reduce write load |
| Split-brain | Two masters exist | Configure min-replicas-to-write; fix network |
| Memory full | OOM errors | Increase maxmemory; enable eviction policy |
| Slow failover | Takes > 60 seconds | Reduce down-after-milliseconds; check network |
| Client disconnects | Frequent reconnections | Check client timeout settings; enable TCP keepalive |
| RDB save fails | Background save error | Check disk space; verify permissions |
| AOF corruption | Redis won't start | Use redis-check-aof to repair |
| Sentinel not detecting failure | s_down not triggering | Verify Sentinel can reach Redis; check down-after-milliseconds |

### Diagnostic Steps

```bash
# Check Redis logs
sudo tail -f /var/log/redis/redis.log

# Check Sentinel logs
sudo tail -f /var/log/redis/sentinel.log

# Check system resources
free -h
df -h
top

# Network connectivity test
redis-cli -h 192.168.1.101 -p 6379 PING
redis-cli -h 192.168.1.101 -p 26379 PING

# Check for blocked clients
redis-cli -h 192.168.1.101 -p 6379 CLIENT LIST | grep blocked
```

---

## 20. Recovery Procedures

### Scenario: Node Won't Start After Crash

```bash
# 1. Check logs for errors
sudo journalctl -u redis -n 100
sudo tail -100 /var/log/redis/redis.log

# 2. Check persistence files
ls -la /var/lib/redis/

# 3. If RDB corrupted, try AOF (if enabled)
redis-server --appendonly yes --dbfilename ""

# 4. If AOF corrupted, repair it
redis-check-aof --fix /var/lib/redis/appendonly.aof

# 5. If RDB corrupted, check backup
redis-check-rdb /var/lib/redis/dump.rdb

# 6. Start with empty database if persistence files corrupted
# (Data will sync from master if this is a replica)
sudo rm /var/lib/redis/dump.rdb
sudo rm /var/lib/redis/appendonly.aof
sudo systemctl start redis
```

### Scenario: Sentinel Not Triggering Failover

```bash
# 1. Check Sentinel can reach Redis
redis-cli -h 192.168.1.101 -p 26379 sentinel ckquorum mymaster

# 2. Verify Sentinel configuration
redis-cli -h 192.168.1.101 -p 26379 sentinel master mymaster

# 3. Check for s_down detection
redis-cli -h 192.168.1.101 -p 26379 sentinel master mymaster | grep flags

# 4. If o_down not achieved, check quorum
# Ensure majority of Sentinels can communicate

# 5. Force failover manually
redis-cli -h 192.168.1.101 -p 26379 sentinel failover mymaster
```

### Scenario: Split-Brain Recovery

```bash
# 1. Identify which master has latest data
redis-cli -h 192.168.1.101 -p 6379 INFO replication | grep master_repl_offset
redis-cli -h 192.168.1.102 -p 6379 INFO replication | grep master_repl_offset

# 2. Stop the old/stale master
sudo systemctl stop redis  # on stale master

# 3. Ensure correct master is promoted
redis-cli -h <correct_master> -p 6379 REPLICAOF NO ONE

# 4. Point replicas to correct master
redis-cli -h <replica> -p 6379 REPLICAOF <correct_master_ip> 6379

# 5. Restart Sentinels to reset state
sudo systemctl restart redis-sentinel  # on all nodes

# 6. Restart old master as replica
sudo systemctl start redis  # will join as replica
```

### Scenario: Complete Cluster Recovery

```bash
# If all nodes are down:

# 1. Start master first
sudo systemctl start redis  # on master node

# 2. Verify master is up
redis-cli -h 192.168.1.101 -p 6379 PING

# 3. Start replicas
sudo systemctl start redis  # on replica nodes

# 4. Verify replication
redis-cli -h 192.168.1.101 -p 6379 INFO replication

# 5. Start Sentinels
sudo systemctl start redis-sentinel  # on all nodes

# 6. Verify Sentinel cluster
redis-cli -h 192.168.1.101 -p 26379 sentinel master mymaster
```

---

## 21. Best Practices for Production

### Cluster Configuration

- Use odd number of Sentinel nodes (3 or 5) for proper quorum
- Configure `min-replicas-to-write 1` to prevent split-brain writes
- Set `min-replicas-max-lag 10` for replication safety
- Enable both RDB and AOF for data safety
- Use `appendfsync everysec` for balance of safety and performance

### Monitoring

- Implement comprehensive monitoring (Prometheus + Grafana)
- Set up alerts for:
  - Replication lag > 10 seconds
  - Memory usage > 80%
  - Connection count spikes
  - Failover events
- Monitor Sentinel logs for failover events
- Track slow queries with SLOWLOG

### Client Applications

- Use Sentinel-aware drivers (not direct Redis connections)
- Implement retry logic with exponential backoff
- Set appropriate connection timeouts
- Use connection pooling
- Handle failover gracefully with proper error handling

### Resource Management

- Set appropriate `maxmemory` limit (leave 30% for overhead)
- Configure eviction policy based on use case
- Monitor and rotate logs regularly
- Use SSDs for persistence files
- Plan for 2x peak load capacity

### Disaster Recovery

- Regular backup of RDB files to remote storage
- Test restore procedures quarterly
- Document all recovery procedures
- Maintain runbooks for common scenarios
- Keep configuration in version control

### Maintenance

- Use rolling restarts for updates
- Trigger manual failover before master maintenance
- Schedule maintenance during low-traffic periods
- Test changes in staging first
- Keep Redis and OS up to date

---

## 22. Test Results Template

### Test Execution Record

| Field | Value |
|-------|-------|
| Test Date | |
| Tester Name | |
| Environment | Test / Staging / Production |
| Redis Version | |
| Cluster Configuration | 1 Master + 2 Replicas + 3 Sentinels |
| Sentinel Quorum | 2 |

### Test Case Results

| Test Case | Status | Duration | Message Loss | Notes |
|-----------|--------|----------|--------------|-------|
| TC1: Master Node Failure | Pass/Fail | | | |
| TC2: Replica Node Failure | Pass/Fail | | N/A | |
| TC3: Sentinel Node Failure | Pass/Fail | | N/A | |
| TC4: Network Partition | Pass/Fail | | | |
| TC5: Graceful Shutdown | Pass/Fail | | | |
| TC6: Multiple Node Failure | Pass/Fail | | | |
| TC7: Disk Space Exhaustion | Pass/Fail | | | |
| TC8: Memory Exhaustion | Pass/Fail | | | |
| TC9: Rolling Restart | Pass/Fail | | | |
| TC10: Data Replication | Pass/Fail | | N/A | |
| TC11: Client Failover | Pass/Fail | | | |
| TC12: Persistence Recovery | Pass/Fail | | | |

### Success Criteria

- All automated failovers complete within expected time (< 30 seconds)
- Data loss is zero or within acceptable limits (< 0.1%)
- Cluster recovers to full health after each test
- No manual intervention required for automatic scenarios
- Clients reconnect successfully within 10 seconds
- Monitoring and alerting triggered appropriately
- All recovery procedures documented and tested

### Issues Found

| Issue # | Description | Severity | Resolution |
|---------|-------------|----------|------------|
| 1 | | High/Medium/Low | |
| 2 | | High/Medium/Low | |
| 3 | | High/Medium/Low | |

### Overall Assessment

**Result**: [ ] Pass  [ ] Fail  [ ] Pass with Conditions

### Recommendations

[List any recommendations for improvement]

---

## Appendix: Sample Test Scripts

### Python Failover Test Script

```python
#!/usr/bin/env python3
"""
Redis Sentinel Failover Test Script
"""

import time
import redis
from redis.sentinel import Sentinel

def test_failover():
    sentinel = Sentinel([
        ('192.168.1.101', 26379),
        ('192.168.1.102', 26379),
        ('192.168.1.103', 26379)
    ], socket_timeout=0.5)

    master = sentinel.master_for('mymaster', socket_timeout=0.5)

    # Get current master
    master_addr = sentinel.discover_master('mymaster')
    print(f"Current master: {master_addr}")

    # Write test data
    counter = 0
    errors = 0

    print("Starting write loop (Ctrl+C to stop)...")
    try:
        while True:
            try:
                master.incr('failover:test:counter')
                counter += 1
                if counter % 100 == 0:
                    print(f"Writes: {counter}, Errors: {errors}")
            except redis.exceptions.ConnectionError as e:
                errors += 1
                print(f"Connection error (will retry): {e}")
                time.sleep(0.5)
                master = sentinel.master_for('mymaster', socket_timeout=0.5)
            except redis.exceptions.ReadOnlyError:
                print("Read-only error - failover in progress")
                time.sleep(1)
                master = sentinel.master_for('mymaster', socket_timeout=0.5)

            time.sleep(0.1)
    except KeyboardInterrupt:
        print(f"\nFinal stats: Writes={counter}, Errors={errors}")
        final_value = master.get('failover:test:counter')
        print(f"Final counter value: {final_value}")

if __name__ == '__main__':
    test_failover()
```

### Bash Monitoring Script

```bash
#!/bin/bash
# Redis Cluster Health Monitor

MASTER_HOST="192.168.1.101"
REPLICA1_HOST="192.168.1.102"
REPLICA2_HOST="192.168.1.103"
SENTINEL_PORT=26379
REDIS_PORT=6379

echo "=== Redis Cluster Health Check ==="
echo "Timestamp: $(date)"
echo ""

# Check Sentinel
echo "=== Sentinel Status ==="
redis-cli -h $MASTER_HOST -p $SENTINEL_PORT sentinel master mymaster 2>/dev/null | grep -E "ip|port|flags|num-slaves"
echo ""

# Get current master
CURRENT_MASTER=$(redis-cli -h $MASTER_HOST -p $SENTINEL_PORT sentinel get-master-addr-by-name mymaster 2>/dev/null | head -1)
echo "Current Master: $CURRENT_MASTER"
echo ""

# Check replication
echo "=== Replication Status ==="
redis-cli -h $CURRENT_MASTER -p $REDIS_PORT info replication 2>/dev/null | grep -E "role|connected_slaves|slave|offset|lag"
echo ""

# Check memory
echo "=== Memory Usage ==="
for host in $MASTER_HOST $REPLICA1_HOST $REPLICA2_HOST; do
    MEM=$(redis-cli -h $host -p $REDIS_PORT info memory 2>/dev/null | grep "used_memory_human" | cut -d: -f2)
    echo "$host: $MEM"
done
echo ""

# Check connectivity
echo "=== Connectivity ==="
for host in $MASTER_HOST $REPLICA1_HOST $REPLICA2_HOST; do
    PING=$(redis-cli -h $host -p $REDIS_PORT ping 2>/dev/null)
    echo "$host Redis: $PING"
    PING=$(redis-cli -h $host -p $SENTINEL_PORT ping 2>/dev/null)
    echo "$host Sentinel: $PING"
done
```

---

*Document Version: 1.0 | Last Updated: January 2026*
