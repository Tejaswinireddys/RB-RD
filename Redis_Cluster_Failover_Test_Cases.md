# Redis Cluster Failover

## Test Cases and Scenarios

### Three-Node Cluster with Redis Sentinel

---

## Document Information

| Field | Value |
|-------|-------|
| Document Version | 2.0 |
| Last Updated | January 2026 |
| Redis Version | 8.x |
| Operating System | Red Hat Enterprise Linux 8.x |
| HA Method | Redis Sentinel |

---

## Table of Contents

1. [Redis Sentinel Overview](#1-redis-sentinel-overview)
2. [Test Environment Setup](#2-test-environment-setup)
3. [Prerequisites and Initial Verification](#3-prerequisites-and-initial-verification)
4. [Test Case 1: Master Node Failure](#4-test-case-1-master-node-failure)
5. [Test Case 2: Replica Node Failure](#5-test-case-2-replica-node-failure)
6. [Test Case 3: Sentinel Node Failure](#6-test-case-3-sentinel-node-failure)
7. [Test Case 4: Network Partition](#7-test-case-4-network-partition)
8. [Test Case 5: Graceful Node Shutdown](#8-test-case-5-graceful-node-shutdown)
9. [Test Case 6: Quorum Loss](#9-test-case-6-quorum-loss)
10. [Test Case 7: Disk Alarm](#10-test-case-7-disk-alarm)
11. [Test Case 8: Memory Exhaustion](#11-test-case-8-memory-exhaustion)
12. [Test Case 9: Rolling Restart](#12-test-case-9-rolling-restart)
13. [Test Case 10: Data Replication Validation](#13-test-case-10-data-replication-validation)
14. [Test Case 11: Client Connection Failover](#14-test-case-11-client-connection-failover)
15. [Test Case 12: Persistence and Recovery](#15-test-case-12-persistence-and-recovery)
16. [Monitoring Commands Reference](#16-monitoring-commands-reference)
17. [Recovery Procedures](#17-recovery-procedures)
18. [Test Results Template](#18-test-results-template)

---

## 1. Redis Sentinel Overview

### What is Redis Sentinel?

Redis Sentinel provides high availability for Redis through automatic failover, monitoring, and configuration provider services.

| Feature | Description |
|---------|-------------|
| Monitoring | Sentinel checks if master and replica instances are working |
| Notification | Sentinel can notify on failures via API |
| Automatic Failover | Promotes replica to master when master fails |
| Configuration Provider | Clients connect to Sentinel to get current master address |

### Key Concepts

| Term | Description |
|------|-------------|
| **Master** | Node that handles all writes and replicates to replicas |
| **Replica** | Nodes that maintain copies of master data |
| **Sentinel** | Monitoring process that handles automatic failover |
| **Quorum** | Minimum Sentinels required to agree on failover (majority) |
| **SDOWN** | Subjectively Down - single Sentinel thinks node is down |
| **ODOWN** | Objectively Down - quorum of Sentinels agree node is down |

### Quorum Requirements

| Cluster Size | Quorum | Tolerated Failures |
|--------------|--------|-------------------|
| 3 Sentinels | 2 | 1 Sentinel |
| 5 Sentinels | 3 | 2 Sentinels |

---

## 2. Test Environment Setup

### Cluster Node Details

| Hostname | IP Address | Redis Role | Sentinel |
|----------|------------|------------|----------|
| redis-node1 | 192.168.1.101 | Master | Sentinel 1 |
| redis-node2 | 192.168.1.102 | Replica | Sentinel 2 |
| redis-node3 | 192.168.1.103 | Replica | Sentinel 3 |

### Port Configuration

| Port | Purpose |
|------|---------|
| 6379 | Redis client connections |
| 26379 | Sentinel communication |

---

## 3. Prerequisites and Initial Verification

### Verify Redis Status

```bash
# Check Redis is running on all nodes
redis-cli -h 192.168.1.101 -p 6379 PING
redis-cli -h 192.168.1.102 -p 6379 PING
redis-cli -h 192.168.1.103 -p 6379 PING
```

### Verify Replication Status

```bash
# Check replication on master
redis-cli -h 192.168.1.101 -p 6379 INFO replication

# Expected output:
# role:master
# connected_slaves:2
# slave0:ip=192.168.1.102,port=6379,state=online,offset=xxx,lag=0
# slave1:ip=192.168.1.103,port=6379,state=online,offset=xxx,lag=0
```

### Verify Sentinel Status

```bash
# Check Sentinel is running
redis-cli -h 192.168.1.101 -p 26379 PING

# Get master information from Sentinel
redis-cli -h 192.168.1.101 -p 26379 SENTINEL master mymaster

# Get current master address
redis-cli -h 192.168.1.101 -p 26379 SENTINEL get-master-addr-by-name mymaster
```

### Verify Sentinel Quorum

```bash
# Check if quorum can be reached
redis-cli -h 192.168.1.101 -p 26379 SENTINEL ckquorum mymaster

# Expected: OK 3 usable Sentinels. Quorum and failover authorization is possible
```

### Create Test Data

```bash
# Write test data to master
redis-cli -h 192.168.1.101 -p 6379 SET test:key "test_value"
redis-cli -h 192.168.1.101 -p 6379 SET test:counter 1000

# Verify data replicated
redis-cli -h 192.168.1.102 -p 6379 GET test:key
redis-cli -h 192.168.1.103 -p 6379 GET test:key
```

---

## 4. Test Case 1: Master Node Failure

### Objective
Simulate failure of the Redis master node and verify automatic failover via Sentinel.

| Attribute | Value |
|-----------|-------|
| Severity | HIGH |
| Expected Recovery | 10-30 seconds |
| Expected Data Loss | Minimal (uncommitted only) |

### Pre-Test Setup

```bash
# Identify current master
redis-cli -h 192.168.1.101 -p 26379 SENTINEL get-master-addr-by-name mymaster

# Record message count
redis-cli -h 192.168.1.101 -p 6379 DBSIZE

# Write test data
for i in {1..1000}; do
    redis-cli -h 192.168.1.101 -p 6379 SET test:msg:$i "message-$i"
done
```

### Test Steps

#### Step 1: Record Initial State

```bash
# Record cluster state
redis-cli -h 192.168.1.101 -p 6379 INFO replication
redis-cli -h 192.168.1.101 -p 26379 SENTINEL master mymaster
```

#### Step 2: Identify Current Master

```bash
# Get master address
redis-cli -h 192.168.1.101 -p 26379 SENTINEL get-master-addr-by-name mymaster

# Example output: 192.168.1.101 6379
```

#### Step 3: Shutdown Master

```bash
# On the master node (192.168.1.101)
redis-cli -h 192.168.1.101 -p 6379 SHUTDOWN NOSAVE

# Or use DEBUG SLEEP to simulate hang
redis-cli -h 192.168.1.101 -p 6379 DEBUG SLEEP 60
```

#### Step 4: Monitor Failover

```bash
# On another node, watch for new master
watch -n 1 "redis-cli -h 192.168.1.102 -p 26379 SENTINEL get-master-addr-by-name mymaster"

# Check Sentinel logs
tail -f /var/log/redis/sentinel.log

# Key events:
# +sdown master mymaster 192.168.1.101 6379
# +odown master mymaster 192.168.1.101 6379 #quorum 2/2
# +switch-master mymaster 192.168.1.101 6379 192.168.1.102 6379
```

#### Step 5: Verify New Master

```bash
# Check new master address
redis-cli -h 192.168.1.102 -p 26379 SENTINEL get-master-addr-by-name mymaster

# Verify new master accepts writes
redis-cli -h 192.168.1.102 -p 6379 SET test:failover "success"
redis-cli -h 192.168.1.102 -p 6379 GET test:failover
```

#### Step 6: Verify Data Integrity

```bash
# Check data count
redis-cli -h 192.168.1.102 -p 6379 DBSIZE

# Verify test data
redis-cli -h 192.168.1.102 -p 6379 GET test:key
```

### Post-Test Recovery

```bash
# Start old master (will rejoin as replica)
redis-server /etc/redis/redis.conf

# Verify it rejoined as replica
redis-cli -h 192.168.1.101 -p 6379 INFO replication
# Expected: role:slave

# Verify Sentinel sees all nodes
redis-cli -h 192.168.1.102 -p 26379 SENTINEL replicas mymaster
```

### Expected Results

- [ ] Sentinel detects master failure (SDOWN then ODOWN)
- [ ] New master elected from replicas
- [ ] Data preserved on new master
- [ ] Old master rejoins as replica
- [ ] Clients can write to new master

---

## 5. Test Case 2: Replica Node Failure

### Objective
Verify that failure of a replica node has minimal impact on operations.

| Attribute | Value |
|-----------|-------|
| Severity | LOW |
| Expected Recovery | Immediate |
| Expected Data Loss | None |

### Test Steps

#### Step 1: Identify Replica Node

```bash
# Check replication status
redis-cli -h 192.168.1.101 -p 6379 INFO replication

# Choose a replica (e.g., 192.168.1.102)
```

#### Step 2: Shutdown Replica

```bash
# On the replica node
redis-cli -h 192.168.1.102 -p 6379 SHUTDOWN NOSAVE
```

#### Step 3: Verify Master Operations Continue

```bash
# Write to master - should succeed
redis-cli -h 192.168.1.101 -p 6379 SET test:replica:failure "test"
redis-cli -h 192.168.1.101 -p 6379 GET test:replica:failure

# Check master sees reduced replicas
redis-cli -h 192.168.1.101 -p 6379 INFO replication
# connected_slaves:1
```

#### Step 4: Verify Sentinel Status

```bash
# Sentinel will mark replica as down
redis-cli -h 192.168.1.101 -p 26379 SENTINEL replicas mymaster

# Replica should show s_down flag
```

#### Step 5: Restart Replica

```bash
# On the replica node
redis-server /etc/redis/redis.conf

# Verify replica rejoined
redis-cli -h 192.168.1.102 -p 6379 INFO replication
# Expected: role:slave, master_link_status:up

# Verify data synced
redis-cli -h 192.168.1.102 -p 6379 GET test:replica:failure
```

### Expected Results

- [ ] No impact on master operations
- [ ] Master continues accepting writes
- [ ] Replica rejoins and syncs automatically
- [ ] No data loss

---

## 6. Test Case 3: Sentinel Node Failure

### Objective
Verify that failure of a Sentinel node does not impact Redis operations.

| Attribute | Value |
|-----------|-------|
| Severity | LOW |
| Expected Recovery | Immediate |
| Expected Data Loss | None |

### Test Steps

#### Step 1: Check Sentinel Status

```bash
# Verify all Sentinels running
redis-cli -h 192.168.1.101 -p 26379 SENTINEL sentinels mymaster

# Should show 2 other Sentinels
```

#### Step 2: Stop One Sentinel

```bash
# On redis-node2
redis-cli -h 192.168.1.102 -p 26379 SHUTDOWN
```

#### Step 3: Verify Remaining Sentinels

```bash
# Check Sentinel count
redis-cli -h 192.168.1.101 -p 26379 SENTINEL sentinels mymaster

# Should show 1 other Sentinel
```

#### Step 4: Verify Quorum Still Possible

```bash
# Check quorum
redis-cli -h 192.168.1.101 -p 26379 SENTINEL ckquorum mymaster

# With 2 Sentinels remaining, quorum (2) is still achievable
```

#### Step 5: Verify Redis Operations

```bash
# Redis should work normally
redis-cli -h 192.168.1.101 -p 6379 SET test:sentinel "working"
redis-cli -h 192.168.1.101 -p 6379 GET test:sentinel
```

#### Step 6: Restart Sentinel

```bash
# On redis-node2
redis-sentinel /etc/redis/sentinel.conf

# Verify Sentinel rejoined
redis-cli -h 192.168.1.101 -p 26379 SENTINEL sentinels mymaster
```

### Expected Results

- [ ] No impact on Redis operations
- [ ] Remaining Sentinels maintain quorum
- [ ] Failover still possible with 2 Sentinels
- [ ] Sentinel rejoins automatically

---

## 7. Test Case 4: Network Partition

### Objective
Test cluster behavior during network partition.

| Attribute | Value |
|-----------|-------|
| Severity | CRITICAL |
| Expected Recovery | 30-60 seconds |
| Partition Handling | Quorum-based |

### Prerequisites

```bash
# Configure min-replicas for safety
redis-cli -h 192.168.1.101 -p 6379 CONFIG SET min-replicas-to-write 1
redis-cli -h 192.168.1.101 -p 6379 CONFIG SET min-replicas-max-lag 10
```

### Test Steps

#### Step 1: Record Pre-Partition State

```bash
redis-cli -h 192.168.1.101 -p 6379 INFO replication
redis-cli -h 192.168.1.101 -p 26379 SENTINEL master mymaster
```

#### Step 2: Create Network Partition

```bash
# On redis-node1 (master), isolate from other nodes
sudo iptables -A INPUT -s 192.168.1.102 -j DROP
sudo iptables -A INPUT -s 192.168.1.103 -j DROP
sudo iptables -A OUTPUT -d 192.168.1.102 -j DROP
sudo iptables -A OUTPUT -d 192.168.1.103 -j DROP

# Creates: [node1] | [node2, node3]
```

#### Step 3: Observe Partition Behavior

```bash
# On isolated master (node1)
redis-cli -h 192.168.1.101 -p 6379 INFO replication
# connected_slaves:0

# On majority side (node2)
redis-cli -h 192.168.1.102 -p 26379 SENTINEL get-master-addr-by-name mymaster
# Should elect new master
```

#### Step 4: Test Writes on Both Sides

```bash
# On isolated old master (should FAIL if min-replicas-to-write=1)
redis-cli -h 192.168.1.101 -p 6379 SET partition:test "minority"
# Error: NOREPLICAS

# On new master (majority side)
redis-cli -h 192.168.1.102 -p 6379 SET partition:test "majority"
# Should succeed
```

#### Step 5: Restore Network

```bash
# On node1
sudo iptables -F
```

#### Step 6: Verify Cluster Heals

```bash
# Wait for cluster to reform
sleep 30

# Old master should become replica
redis-cli -h 192.168.1.101 -p 6379 INFO replication
# Expected: role:slave

# Check all Sentinels agree on master
redis-cli -h 192.168.1.101 -p 26379 SENTINEL get-master-addr-by-name mymaster
redis-cli -h 192.168.1.102 -p 26379 SENTINEL get-master-addr-by-name mymaster
redis-cli -h 192.168.1.103 -p 26379 SENTINEL get-master-addr-by-name mymaster
```

### Expected Results

- [ ] Isolated master stops accepting writes (with min-replicas)
- [ ] Majority side elects new master
- [ ] Old master becomes replica after partition heals
- [ ] No data corruption

---

## 8. Test Case 5: Graceful Node Shutdown

### Objective
Verify proper failover during planned maintenance.

| Attribute | Value |
|-----------|-------|
| Severity | LOW |
| Expected Recovery | 10-15 seconds |
| Expected Data Loss | None |

### Test Steps

#### Step 1: Check Current Master

```bash
redis-cli -h 192.168.1.101 -p 26379 SENTINEL get-master-addr-by-name mymaster
```

#### Step 2: Trigger Manual Failover (If Shutting Down Master)

```bash
# Request Sentinel to perform failover
redis-cli -h 192.168.1.101 -p 26379 SENTINEL failover mymaster

# Verify new master elected
redis-cli -h 192.168.1.101 -p 26379 SENTINEL get-master-addr-by-name mymaster
```

#### Step 3: Graceful Shutdown

```bash
# On the node being shut down
redis-cli -h 192.168.1.101 -p 6379 SHUTDOWN SAVE
```

#### Step 4: Verify Cluster Status

```bash
# Check new master is operating
redis-cli -h 192.168.1.102 -p 6379 INFO replication

# Verify writes work
redis-cli -h 192.168.1.102 -p 6379 SET test:graceful "shutdown"
```

#### Step 5: Restart Node

```bash
# Start Redis
redis-server /etc/redis/redis.conf

# Node will join as replica
redis-cli -h 192.168.1.101 -p 6379 INFO replication
```

### Expected Results

- [ ] Manual failover completes smoothly
- [ ] No data loss with SHUTDOWN SAVE
- [ ] Node rejoins as replica

---

## 9. Test Case 6: Quorum Loss

### Objective
Test behavior when Sentinel quorum is lost.

| Attribute | Value |
|-----------|-------|
| Severity | CRITICAL |
| Recovery | Manual intervention |

> **Warning**: This will prevent automatic failover. Test environment only.

### Test Steps

#### Step 1: Stop Two Sentinels

```bash
# Stop Sentinel on node2
redis-cli -h 192.168.1.102 -p 26379 SHUTDOWN

# Stop Sentinel on node3
redis-cli -h 192.168.1.103 -p 26379 SHUTDOWN
```

#### Step 2: Verify Quorum Lost

```bash
# Check quorum
redis-cli -h 192.168.1.101 -p 26379 SENTINEL ckquorum mymaster

# Expected: NOQUORUM (only 1 Sentinel, need 2)
```

#### Step 3: Test Failover (Should Fail)

```bash
# Try to trigger failover
redis-cli -h 192.168.1.101 -p 26379 SENTINEL failover mymaster

# Will fail due to no quorum
```

#### Step 4: Verify Redis Still Works

```bash
# Redis operations continue (no failover capability)
redis-cli -h 192.168.1.101 -p 6379 SET test:noquorum "working"
```

#### Step 5: Recovery

```bash
# Start Sentinels
redis-sentinel /etc/redis/sentinel.conf  # on node2
redis-sentinel /etc/redis/sentinel.conf  # on node3

# Verify quorum restored
redis-cli -h 192.168.1.101 -p 26379 SENTINEL ckquorum mymaster
```

### Expected Results

- [ ] Automatic failover disabled without quorum
- [ ] Redis operations continue
- [ ] Quorum restored after Sentinels start

---

## 10. Test Case 7: Disk Alarm

### Objective
Verify behavior when disk space is exhausted.

| Attribute | Value |
|-----------|-------|
| Severity | HIGH |
| Impact | Persistence fails |

### Test Steps

#### Step 1: Check Persistence Config

```bash
redis-cli -h 192.168.1.101 -p 6379 CONFIG GET save
redis-cli -h 192.168.1.101 -p 6379 CONFIG GET appendonly
```

#### Step 2: Simulate Disk Full

```bash
# Create large file to fill disk
sudo fallocate -l 40G /var/lib/redis/large-file
```

#### Step 3: Trigger Save

```bash
# Trigger background save
redis-cli -h 192.168.1.101 -p 6379 BGSAVE

# Check save status
redis-cli -h 192.168.1.101 -p 6379 LASTSAVE
redis-cli -h 192.168.1.101 -p 6379 INFO persistence
# rdb_last_bgsave_status:err
```

#### Step 4: Verify Write Behavior

```bash
# If stop-writes-on-bgsave-error is yes (default)
redis-cli -h 192.168.1.101 -p 6379 SET test:disk "test"
# May return: MISCONF error
```

#### Step 5: Clear Disk

```bash
# Remove test file
sudo rm /var/lib/redis/large-file

# Retry save
redis-cli -h 192.168.1.101 -p 6379 BGSAVE

# Verify success
redis-cli -h 192.168.1.101 -p 6379 INFO persistence
# rdb_last_bgsave_status:ok
```

### Expected Results

- [ ] Save operations fail when disk full
- [ ] Writes blocked if stop-writes-on-bgsave-error enabled
- [ ] Operations resume after disk freed

---

## 11. Test Case 8: Memory Exhaustion

### Objective
Test behavior under memory pressure.

| Attribute | Value |
|-----------|-------|
| Severity | HIGH |
| Impact | Depends on eviction policy |

### Test Steps

#### Step 1: Check Memory Configuration

```bash
redis-cli -h 192.168.1.101 -p 6379 CONFIG GET maxmemory
redis-cli -h 192.168.1.101 -p 6379 CONFIG GET maxmemory-policy
```

#### Step 2: Set Memory Limit

```bash
# Set low limit for testing
redis-cli -h 192.168.1.101 -p 6379 CONFIG SET maxmemory 100mb
redis-cli -h 192.168.1.101 -p 6379 CONFIG SET maxmemory-policy allkeys-lru
```

#### Step 3: Fill Memory

```bash
# Generate data until limit reached
for i in {1..100000}; do
    redis-cli -h 192.168.1.101 -p 6379 SET key:$i "$(head -c 1000 /dev/urandom | base64)" 2>/dev/null
done
```

#### Step 4: Observe Behavior

```bash
# Check memory usage
redis-cli -h 192.168.1.101 -p 6379 INFO memory | grep used_memory_human

# Check eviction stats
redis-cli -h 192.168.1.101 -p 6379 INFO stats | grep evicted_keys
```

#### Step 5: Recovery

```bash
# Increase memory limit
redis-cli -h 192.168.1.101 -p 6379 CONFIG SET maxmemory 2gb

# Or flush data
redis-cli -h 192.168.1.101 -p 6379 FLUSHDB
```

### Expected Results

- [ ] Eviction policy applied when limit reached
- [ ] OOM errors returned (if noeviction policy)
- [ ] Reads continue working
- [ ] Normal operation after limit increased

---

## 12. Test Case 9: Rolling Restart

### Objective
Perform rolling restart with zero downtime.

| Attribute | Value |
|-----------|-------|
| Severity | LOW |
| Expected Downtime | None |

### Test Steps

#### Step 1: Verify Initial State

```bash
redis-cli -h 192.168.1.101 -p 6379 INFO replication
redis-cli -h 192.168.1.101 -p 26379 SENTINEL master mymaster
```

#### Step 2: Restart Replicas First

```bash
# Restart replica 1 (node3)
redis-cli -h 192.168.1.103 -p 6379 SHUTDOWN SAVE
redis-server /etc/redis/redis.conf  # on node3

# Wait for sync
redis-cli -h 192.168.1.103 -p 6379 INFO replication

# Restart replica 2 (node2)
redis-cli -h 192.168.1.102 -p 6379 SHUTDOWN SAVE
redis-server /etc/redis/redis.conf  # on node2
```

#### Step 3: Failover Then Restart Master

```bash
# Trigger failover
redis-cli -h 192.168.1.101 -p 26379 SENTINEL failover mymaster

# Wait for failover
sleep 10

# Verify new master
redis-cli -h 192.168.1.102 -p 26379 SENTINEL get-master-addr-by-name mymaster

# Restart old master (now replica)
redis-cli -h 192.168.1.101 -p 6379 SHUTDOWN SAVE
redis-server /etc/redis/redis.conf  # on node1
```

#### Step 4: Verify Complete Cluster

```bash
redis-cli -h 192.168.1.102 -p 6379 INFO replication
redis-cli -h 192.168.1.102 -p 26379 SENTINEL master mymaster
```

### Expected Results

- [ ] Zero downtime during restart
- [ ] All data preserved
- [ ] Replication working after restart

---

## 13. Test Case 10: Data Replication Validation

### Objective
Verify data is correctly replicated to all replicas.

| Attribute | Value |
|-----------|-------|
| Severity | MEDIUM |
| Purpose | Configuration validation |

### Test Steps

#### Step 1: Check Replication Status

```bash
redis-cli -h 192.168.1.101 -p 6379 INFO replication

# Verify: connected_slaves:2, all slaves state=online, lag=0
```

#### Step 2: Write Test Data

```bash
# Write various data types
redis-cli -h 192.168.1.101 -p 6379 SET string:key "test_value"
redis-cli -h 192.168.1.101 -p 6379 HSET hash:key field1 value1 field2 value2
redis-cli -h 192.168.1.101 -p 6379 LPUSH list:key item1 item2 item3
redis-cli -h 192.168.1.101 -p 6379 SADD set:key member1 member2 member3
redis-cli -h 192.168.1.101 -p 6379 ZADD zset:key 1 one 2 two 3 three
```

#### Step 3: Verify on Replicas

```bash
# On replica 1
redis-cli -h 192.168.1.102 -p 6379 GET string:key
redis-cli -h 192.168.1.102 -p 6379 HGETALL hash:key
redis-cli -h 192.168.1.102 -p 6379 LRANGE list:key 0 -1

# On replica 2
redis-cli -h 192.168.1.103 -p 6379 GET string:key
redis-cli -h 192.168.1.103 -p 6379 SMEMBERS set:key
redis-cli -h 192.168.1.103 -p 6379 ZRANGE zset:key 0 -1 WITHSCORES
```

#### Step 4: Verify Replication Offset

```bash
# Compare offsets
redis-cli -h 192.168.1.101 -p 6379 INFO replication | grep master_repl_offset
redis-cli -h 192.168.1.102 -p 6379 INFO replication | grep slave_repl_offset
redis-cli -h 192.168.1.103 -p 6379 INFO replication | grep slave_repl_offset

# Offsets should be close
```

### Expected Results

- [ ] All data types replicated correctly
- [ ] Replication offsets match
- [ ] No lag between master and replicas

---

## 14. Test Case 11: Client Connection Failover

### Objective
Verify clients reconnect during node failures.

| Attribute | Value |
|-----------|-------|
| Severity | MEDIUM |
| Expected Reconnect | < 10 seconds |

### Prerequisites

Client must use Sentinel-aware driver with multiple Sentinel addresses.

### Test Steps

#### Step 1: Check Connections

```bash
redis-cli -h 192.168.1.101 -p 6379 CLIENT LIST
```

#### Step 2: Stop Master

```bash
redis-cli -h 192.168.1.101 -p 6379 SHUTDOWN NOSAVE
```

#### Step 3: Monitor Client Reconnection

```bash
# Watch for clients on new master
watch -n 1 "redis-cli -h 192.168.1.102 -p 6379 CLIENT LIST"
```

#### Step 4: Verify Operations Continue

```bash
# Check client can write to new master
redis-cli -h 192.168.1.102 -p 6379 INFO clients
```

### Expected Results

- [ ] Client detects connection loss
- [ ] Client queries Sentinel for new master
- [ ] Client reconnects within 10 seconds
- [ ] Operations continue on new master

---

## 15. Test Case 12: Persistence and Recovery

### Objective
Verify data integrity through RDB/AOF persistence.

| Attribute | Value |
|-----------|-------|
| Severity | MEDIUM |
| Purpose | Data safety validation |

### Test Steps

#### Step 1: Check Persistence Config

```bash
redis-cli -h 192.168.1.101 -p 6379 CONFIG GET save
redis-cli -h 192.168.1.101 -p 6379 CONFIG GET appendonly
```

#### Step 2: Write Test Data

```bash
redis-cli -h 192.168.1.101 -p 6379 SET persist:test:1 "data_1"
redis-cli -h 192.168.1.101 -p 6379 SET persist:test:2 "data_2"
redis-cli -h 192.168.1.101 -p 6379 DBSIZE
```

#### Step 3: Force Persistence

```bash
# Trigger RDB save
redis-cli -h 192.168.1.101 -p 6379 BGSAVE

# Wait for completion
redis-cli -h 192.168.1.101 -p 6379 LASTSAVE

# If AOF enabled, rewrite
redis-cli -h 192.168.1.101 -p 6379 BGREWRITEAOF
```

#### Step 4: Simulate Crash

```bash
# Force kill (crash simulation)
redis-cli -h 192.168.1.101 -p 6379 DEBUG SEGFAULT

# Or
sudo kill -9 $(pidof redis-server)
```

#### Step 5: Restart and Verify

```bash
# Start Redis
redis-server /etc/redis/redis.conf

# Verify data recovered
redis-cli -h 192.168.1.101 -p 6379 DBSIZE
redis-cli -h 192.168.1.101 -p 6379 GET persist:test:1
redis-cli -h 192.168.1.101 -p 6379 GET persist:test:2
```

### Expected Results

- [ ] Data persisted to RDB/AOF
- [ ] Data fully recovered after restart
- [ ] No corruption

---

## 16. Monitoring Commands Reference

### Redis Status Commands

```bash
# Server info
redis-cli -h <host> -p 6379 INFO

# Replication status
redis-cli -h <host> -p 6379 INFO replication

# Memory usage
redis-cli -h <host> -p 6379 INFO memory

# Persistence status
redis-cli -h <host> -p 6379 INFO persistence

# Client connections
redis-cli -h <host> -p 6379 CLIENT LIST

# Database size
redis-cli -h <host> -p 6379 DBSIZE
```

### Sentinel Commands

```bash
# Master info
redis-cli -h <host> -p 26379 SENTINEL master mymaster

# Get master address
redis-cli -h <host> -p 26379 SENTINEL get-master-addr-by-name mymaster

# List replicas
redis-cli -h <host> -p 26379 SENTINEL replicas mymaster

# List Sentinels
redis-cli -h <host> -p 26379 SENTINEL sentinels mymaster

# Check quorum
redis-cli -h <host> -p 26379 SENTINEL ckquorum mymaster

# Trigger failover
redis-cli -h <host> -p 26379 SENTINEL failover mymaster
```

### Health Check Commands

```bash
# Ping test
redis-cli -h <host> -p 6379 PING

# Latency test
redis-cli -h <host> -p 6379 --latency

# Memory doctor
redis-cli -h <host> -p 6379 MEMORY DOCTOR
```

---

## 17. Recovery Procedures

### Node Won't Start

```bash
# Check logs
sudo tail -100 /var/log/redis/redis.log

# Check RDB file
redis-check-rdb /var/lib/redis/dump.rdb

# Check AOF file
redis-check-aof /var/lib/redis/appendonly.aof

# Repair AOF if needed
redis-check-aof --fix /var/lib/redis/appendonly.aof
```

### Force Replica to Master

```bash
# On the replica you want to promote
redis-cli -h <replica> -p 6379 REPLICAOF NO ONE

# Point other replicas to new master
redis-cli -h <other-replica> -p 6379 REPLICAOF <new-master-ip> 6379
```

### Reset Sentinel

```bash
# Reset Sentinel state for a master
redis-cli -h <host> -p 26379 SENTINEL RESET mymaster
```

### Remove Failed Node from Sentinel

```bash
# Sentinel will auto-remove after timeout
# Or restart Sentinels to refresh state
```

---

## 18. Test Results Template

### Test Execution Record

| Field | Value |
|-------|-------|
| Test Date | |
| Tester Name | |
| Environment | |
| Redis Version | |
| Sentinel Quorum | 2 |
| Cluster Size | 3 nodes |

### Test Results

| Test Case | Status | Failover Time | Data Loss | Notes |
|-----------|--------|---------------|-----------|-------|
| TC1: Master Node Failure | | | | |
| TC2: Replica Node Failure | | | | |
| TC3: Sentinel Node Failure | | | | |
| TC4: Network Partition | | | | |
| TC5: Graceful Shutdown | | | | |
| TC6: Quorum Loss | | | | |
| TC7: Disk Alarm | | | | |
| TC8: Memory Exhaustion | | | | |
| TC9: Rolling Restart | | | | |
| TC10: Replication Validation | | | | |
| TC11: Client Failover | | | | |
| TC12: Persistence Recovery | | | | |

### Success Criteria

- [ ] Failover completes within 30 seconds
- [ ] No data loss for acknowledged writes
- [ ] Cluster recovers automatically
- [ ] Quorum maintained with 1 node failure
- [ ] Clients reconnect within 10 seconds

---

## Quick Reference: Key Commands

| Action | Command |
|--------|---------|
| Get master | `redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster` |
| Check quorum | `redis-cli -p 26379 SENTINEL ckquorum mymaster` |
| Manual failover | `redis-cli -p 26379 SENTINEL failover mymaster` |
| Replication status | `redis-cli -p 6379 INFO replication` |
| Shutdown (save) | `redis-cli -p 6379 SHUTDOWN SAVE` |
| Shutdown (no save) | `redis-cli -p 6379 SHUTDOWN NOSAVE` |
| Make replica | `redis-cli -p 6379 REPLICAOF <ip> <port>` |
| Promote to master | `redis-cli -p 6379 REPLICAOF NO ONE` |
| Background save | `redis-cli -p 6379 BGSAVE` |
| Memory info | `redis-cli -p 6379 INFO memory` |

---

*Document Version: 2.0 | Last Updated: January 2026*
