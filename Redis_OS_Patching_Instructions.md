# Redis Cluster OS Patching Instructions

## Rolling Patch Procedure for Three-Node Sentinel Cluster

---

## Document Information

| Field | Value |
|-------|-------|
| Document Version | 1.0 |
| Last Updated | January 2026 |
| Redis Version | 8.x |
| Cluster Size | 3 Nodes (1 Master + 2 Replicas) |
| HA Method | Redis Sentinel |

---

## Critical Rules

> **IMPORTANT: READ BEFORE PROCEEDING**

| Rule | Description |
|------|-------------|
| **ONE NODE AT A TIME** | Never patch more than one node simultaneously |
| **REPLICAS FIRST** | Always patch replica nodes before master |
| **VERIFY BEFORE PROCEED** | Must complete ALL verification steps before patching next node |
| **MAINTAIN QUORUM** | Sentinel needs 2 of 3 nodes for failover capability |
| **NO PARALLEL PATCHING** | Do NOT schedule two nodes for patching at the same time |
| **WAIT FOR SYNC** | Ensure replication is synchronized before proceeding |

---

## Cluster Node Details

| Hostname | IP Address | Redis Role | Sentinel |
|----------|------------|------------|----------|
| redis-node1 | 192.168.1.101 | Master | Sentinel 1 |
| redis-node2 | 192.168.1.102 | Replica | Sentinel 2 |
| redis-node3 | 192.168.1.103 | Replica | Sentinel 3 |

---

## Pre-Patching Checklist

Before starting the patching process, verify the cluster is healthy.

### Run on Any Node

```bash
# 1. Check current master
redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster

# 2. Check replication status (run on master)
redis-cli -p 6379 INFO replication
# Expected: connected_slaves:2, all slaves state=online

# 3. Check Sentinel quorum
redis-cli -p 26379 SENTINEL ckquorum mymaster
# Expected: OK 3 usable Sentinels

# 4. Check for any issues
redis-cli -p 6379 INFO persistence
```

### Pre-Patching Verification Checklist

| Check | Expected Result | Actual | Pass/Fail |
|-------|-----------------|--------|-----------|
| Master identified | IP returned | | |
| 2 replicas connected | connected_slaves:2 | | |
| All replicas online | state=online | | |
| Sentinel quorum OK | OK 3 usable Sentinels | | |
| Replication lag = 0 | lag=0 | | |

> **DO NOT PROCEED** if any check fails. Contact application team.

---

## Patching Sequence

### IMPORTANT: Patch Order

```
┌─────────────────────────────────────────────────────────────────┐
│                    PATCHING SEQUENCE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. PATCH REPLICA 1  ──►  VERIFY  ──►  CONFIRM  ──►            │
│                                                    │            │
│  2. PATCH REPLICA 2  ──►  VERIFY  ──►  CONFIRM  ──►            │
│                                                    │            │
│  3. FAILOVER MASTER  ──►  PATCH OLD MASTER  ──►  COMPLETE      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

NOTE: Always patch REPLICAS first, then MASTER last!
```

---

## STEP 1: Identify Current Master

Before patching, identify which node is the current master.

```bash
# Run on any node
redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster

# Example output: 192.168.1.101 6379
# This means redis-node1 is currently the master
```

**Record Current Master:** ____________________

---

## STEP 2: Patch Replica 1

> **NOTE:** Start with a REPLICA node, NOT the master.

Assuming redis-node2 is a replica:

### 2.1 Pre-Patch Verification

```bash
# Verify node is a replica
redis-cli -h 192.168.1.102 -p 6379 INFO replication | grep role
# Expected: role:slave

# Check replication is synced
redis-cli -h 192.168.1.102 -p 6379 INFO replication | grep master_link_status
# Expected: master_link_status:up
```

### 2.2 Stop Redis on Replica 1

```bash
# Run on redis-node2
redis-cli -p 6379 SHUTDOWN SAVE
```

### 2.3 Verify Cluster Still Operational

```bash
# Run on redis-node1 (master)
redis-cli -p 6379 INFO replication
# Expected: connected_slaves:1 (reduced from 2)

# Verify Sentinel still has quorum
redis-cli -p 26379 SENTINEL ckquorum mymaster
# Expected: OK 2 usable Sentinels (still enough for quorum)
```

### 2.4 Perform OS Patching

```bash
# Run on redis-node2
# Perform your standard OS patching procedures
# Reboot if required
```

### 2.5 Start Redis After Patching

```bash
# Run on redis-node2 (after reboot if applicable)
# Start Redis server
redis-server /etc/redis/redis.conf

# Start Sentinel
redis-sentinel /etc/redis/sentinel.conf
```

### 2.6 Post-Patch Verification for Replica 1

```bash
# Run on redis-node2
redis-cli -p 6379 INFO replication
# Expected: role:slave, master_link_status:up

# Run on master (redis-node1)
redis-cli -p 6379 INFO replication
# Expected: connected_slaves:2

# Check Sentinel
redis-cli -p 26379 SENTINEL ckquorum mymaster
# Expected: OK 3 usable Sentinels
```

**VERIFICATION CHECKPOINT 1**

| Check | Command | Expected Result | Actual | Pass/Fail |
|-------|---------|-----------------|--------|-----------|
| Replica 1 rejoined | `redis-cli -p 6379 INFO replication` | role:slave | | |
| Master link up | `INFO replication \| grep master_link_status` | up | | |
| 2 replicas connected | `INFO replication` on master | connected_slaves:2 | | |
| Sentinel quorum | `SENTINEL ckquorum mymaster` | OK 3 usable | | |
| Replication lag | `INFO replication \| grep lag` | lag=0 | | |

> **STOP!** Do NOT proceed to Replica 2 until ALL checks pass.
>
> **Wait minimum 5 minutes** after node rejoins before proceeding.

---

## STEP 3: Patch Replica 2

> **PREREQUISITE:** Step 2 must be 100% complete and verified.

Assuming redis-node3 is the second replica:

### 3.1 Pre-Patch Verification

```bash
# Verify 2 replicas connected to master
redis-cli -h 192.168.1.101 -p 6379 INFO replication
# Expected: connected_slaves:2

# Verify node is a replica
redis-cli -h 192.168.1.103 -p 6379 INFO replication | grep role
# Expected: role:slave
```

### 3.2 Stop Redis on Replica 2

```bash
# Run on redis-node3
redis-cli -p 6379 SHUTDOWN SAVE
```

### 3.3 Verify Cluster Still Operational

```bash
# Run on redis-node1 (master)
redis-cli -p 6379 INFO replication
# Expected: connected_slaves:1

# Verify Sentinel still has quorum
redis-cli -p 26379 SENTINEL ckquorum mymaster
# Expected: OK 2 usable Sentinels
```

### 3.4 Perform OS Patching

```bash
# Run on redis-node3
# Perform your standard OS patching procedures
# Reboot if required
```

### 3.5 Start Redis After Patching

```bash
# Run on redis-node3 (after reboot if applicable)
# Start Redis server
redis-server /etc/redis/redis.conf

# Start Sentinel
redis-sentinel /etc/redis/sentinel.conf
```

### 3.6 Post-Patch Verification for Replica 2

```bash
# Run on redis-node3
redis-cli -p 6379 INFO replication
# Expected: role:slave, master_link_status:up

# Run on master
redis-cli -p 6379 INFO replication
# Expected: connected_slaves:2
```

**VERIFICATION CHECKPOINT 2**

| Check | Command | Expected Result | Actual | Pass/Fail |
|-------|---------|-----------------|--------|-----------|
| Replica 2 rejoined | `redis-cli -p 6379 INFO replication` | role:slave | | |
| Master link up | `INFO replication \| grep master_link_status` | up | | |
| 2 replicas connected | `INFO replication` on master | connected_slaves:2 | | |
| Sentinel quorum | `SENTINEL ckquorum mymaster` | OK 3 usable | | |
| Replication lag | `INFO replication \| grep lag` | lag=0 | | |

> **STOP!** Do NOT proceed to Master until ALL checks pass.
>
> **Wait minimum 5 minutes** after node rejoins before proceeding.

---

## STEP 4: Patch Master Node

> **PREREQUISITE:** Steps 2 and 3 must be 100% complete and verified.

### 4.1 Pre-Patch Verification

```bash
# Verify all nodes healthy
redis-cli -h 192.168.1.101 -p 6379 INFO replication
# Expected: role:master, connected_slaves:2

redis-cli -p 26379 SENTINEL ckquorum mymaster
# Expected: OK 3 usable Sentinels
```

### 4.2 Trigger Manual Failover

```bash
# Trigger failover BEFORE shutting down master
redis-cli -p 26379 SENTINEL failover mymaster

# Wait 10 seconds for failover to complete
sleep 10

# Verify new master elected
redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster
# Should return different IP (e.g., 192.168.1.102 or 192.168.1.103)
```

### 4.3 Verify Old Master is Now Replica

```bash
# Run on redis-node1 (old master)
redis-cli -p 6379 INFO replication
# Expected: role:slave (now a replica)
```

### 4.4 Stop Redis on Old Master

```bash
# Run on redis-node1
redis-cli -p 6379 SHUTDOWN SAVE
```

### 4.5 Verify Cluster Still Operational

```bash
# Run on new master
redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster
# Returns new master IP

redis-cli -h <new-master-ip> -p 6379 INFO replication
# Expected: role:master, connected_slaves:1

redis-cli -p 26379 SENTINEL ckquorum mymaster
# Expected: OK 2 usable Sentinels
```

### 4.6 Perform OS Patching

```bash
# Run on redis-node1
# Perform your standard OS patching procedures
# Reboot if required
```

### 4.7 Start Redis After Patching

```bash
# Run on redis-node1 (after reboot if applicable)
# Start Redis server
redis-server /etc/redis/redis.conf

# Start Sentinel
redis-sentinel /etc/redis/sentinel.conf
```

### 4.8 Post-Patch Verification for Old Master

```bash
# Run on redis-node1
redis-cli -p 6379 INFO replication
# Expected: role:slave (will rejoin as replica)

# Verify on new master
redis-cli -h <new-master-ip> -p 6379 INFO replication
# Expected: connected_slaves:2
```

**VERIFICATION CHECKPOINT 3**

| Check | Command | Expected Result | Actual | Pass/Fail |
|-------|---------|-----------------|--------|-----------|
| Old master rejoined as replica | `INFO replication` | role:slave | | |
| Master link up | `INFO replication \| grep master_link_status` | up | | |
| 2 replicas connected to new master | `INFO replication` on master | connected_slaves:2 | | |
| Sentinel quorum | `SENTINEL ckquorum mymaster` | OK 3 usable | | |
| Replication lag | `INFO replication \| grep lag` | lag=0 | | |

---

## Final Verification

After all three nodes are patched, perform final cluster health check.

### Run These Commands

```bash
# 1. Identify current master
redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster

# 2. Check replication on master
redis-cli -h <master-ip> -p 6379 INFO replication
# Expected: connected_slaves:2, both replicas online with lag=0

# 3. Check Sentinel quorum
redis-cli -p 26379 SENTINEL ckquorum mymaster
# Expected: OK 3 usable Sentinels

# 4. Test write operation
redis-cli -h <master-ip> -p 6379 SET patching:complete "$(date)"
redis-cli -h <master-ip> -p 6379 GET patching:complete
```

### Final Verification Checklist

| Check | Expected Result | Actual | Pass/Fail |
|-------|-----------------|--------|-----------|
| Master identified | IP returned | | |
| 2 replicas connected | connected_slaves:2 | | |
| All replicas synced | lag=0 for all | | |
| Sentinel quorum | OK 3 usable Sentinels | | |
| Write test successful | Value returned | | |

---

## Troubleshooting

### Redis Won't Start After Patching

```bash
# Check logs
sudo tail -100 /var/log/redis/redis.log

# Check if Redis is running
ps aux | grep redis-server

# Check RDB/AOF file integrity
redis-check-rdb /var/lib/redis/dump.rdb
redis-check-aof /var/lib/redis/appendonly.aof
```

### Replica Won't Sync with Master

```bash
# Check replication status
redis-cli -p 6379 INFO replication

# Force reconnect to master
redis-cli -p 6379 REPLICAOF <master-ip> 6379
```

### Sentinel Not Detecting Nodes

```bash
# Reset Sentinel state
redis-cli -p 26379 SENTINEL RESET mymaster

# Restart Sentinel if needed
redis-sentinel /etc/redis/sentinel.conf
```

---

## Emergency Contacts

| Role | Contact |
|------|---------|
| Application Team | [Contact Info] |
| DBA Team | [Contact Info] |
| On-Call Engineer | [Contact Info] |

---

## Quick Reference Commands

| Action | Command |
|--------|---------|
| Get master address | `redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster` |
| Check replication | `redis-cli -p 6379 INFO replication` |
| Check Sentinel quorum | `redis-cli -p 26379 SENTINEL ckquorum mymaster` |
| Shutdown with save | `redis-cli -p 6379 SHUTDOWN SAVE` |
| Trigger failover | `redis-cli -p 26379 SENTINEL failover mymaster` |
| Start Redis | `redis-server /etc/redis/redis.conf` |
| Start Sentinel | `redis-sentinel /etc/redis/sentinel.conf` |

---

## Summary Checklist

| Step | Action | Verified |
|------|--------|----------|
| 1 | Pre-patching cluster health verified | [ ] |
| 2 | Current master identified | [ ] |
| 3 | **Replica 1** stopped | [ ] |
| 4 | Cluster operational with 1 replica | [ ] |
| 5 | Replica 1 patched and rebooted | [ ] |
| 6 | Replica 1 started and rejoined | [ ] |
| 7 | **Checkpoint 1: 2 replicas connected** | [ ] |
| 8 | **Wait 5 minutes** | [ ] |
| 9 | **Replica 2** stopped | [ ] |
| 10 | Cluster operational with 1 replica | [ ] |
| 11 | Replica 2 patched and rebooted | [ ] |
| 12 | Replica 2 started and rejoined | [ ] |
| 13 | **Checkpoint 2: 2 replicas connected** | [ ] |
| 14 | **Wait 5 minutes** | [ ] |
| 15 | **Manual failover triggered** | [ ] |
| 16 | New master confirmed | [ ] |
| 17 | Old master (now replica) stopped | [ ] |
| 18 | Cluster operational | [ ] |
| 19 | Old master patched and rebooted | [ ] |
| 20 | Old master started and rejoined as replica | [ ] |
| 21 | **Checkpoint 3: All nodes healthy** | [ ] |
| 22 | Final verification completed | [ ] |

---

## Important Reminders

1. **NEVER patch two nodes at the same time**
2. **ALWAYS patch REPLICAS before MASTER**
3. **TRIGGER FAILOVER before patching master**
4. **VERIFY replication is synced before proceeding to next node**
5. **WAIT minimum 5 minutes between nodes**
6. **STOP and contact application team if any verification fails**

---

*Document Version: 1.0 | Last Updated: January 2026*
