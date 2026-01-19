# RabbitMQ Cluster OS Patching Instructions

## Rolling Patch Procedure for Three-Node Cluster

---

## Document Information

| Field | Value |
|-------|-------|
| Document Version | 1.0 |
| Last Updated | January 2026 |
| RabbitMQ Version | 4.1.x |
| Cluster Size | 3 Nodes |
| Partition Handling | pause_minority |

---

## Critical Rules

> **IMPORTANT: READ BEFORE PROCEEDING**

| Rule | Description |
|------|-------------|
| **ONE NODE AT A TIME** | Never patch more than one node simultaneously |
| **VERIFY BEFORE PROCEED** | Must complete ALL verification steps before patching next node |
| **MAINTAIN QUORUM** | Cluster needs 2 of 3 nodes running at all times |
| **NO PARALLEL PATCHING** | Do NOT schedule two nodes for patching at the same time |
| **WAIT FOR CONFIRMATION** | Get explicit confirmation from application team if unsure |

---

## Cluster Node Details

| Hostname | IP Address | Node Name |
|----------|------------|-----------|
| rabbitmq-node1 | 192.168.1.101 | rabbit@rabbitmq-node1 |
| rabbitmq-node2 | 192.168.1.102 | rabbit@rabbitmq-node2 |
| rabbitmq-node3 | 192.168.1.103 | rabbit@rabbitmq-node3 |

---

## Pre-Patching Checklist

Before starting the patching process, verify the cluster is healthy.

### Run on Any Node

```bash
# 1. Check cluster status - ALL 3 nodes must be running
sudo rabbitmqctl cluster_status

# 2. Check for any alarms - should return empty
sudo rabbitmqctl list_alarms

# 3. Check node health
sudo rabbitmq-diagnostics check_running
sudo rabbitmq-diagnostics check_local_alarms

# 4. Record queue status
sudo rabbitmqctl list_queues name messages consumers
```

### Pre-Patching Verification Checklist

| Check | Expected Result | Actual | Pass/Fail |
|-------|-----------------|--------|-----------|
| All 3 nodes running | Yes | | |
| No alarms present | Empty list | | |
| Node health check passes | OK | | |
| Queues have consumers | Yes | | |

> **DO NOT PROCEED** if any check fails. Contact application team.

---

## Patching Procedure

### Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    PATCHING SEQUENCE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PATCH NODE 1  ──►  VERIFY  ──►  CONFIRM  ──►                  │
│                                              │                  │
│  PATCH NODE 2  ──►  VERIFY  ──►  CONFIRM  ──►                  │
│                                              │                  │
│  PATCH NODE 3  ──►  VERIFY  ──►  COMPLETE                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## STEP 1: Patch Node 1 (rabbitmq-node1)

### 1.1 Pre-Patch Verification

```bash
# Run on node2 or node3
sudo rabbitmqctl cluster_status
# Confirm: All 3 nodes are running
```

### 1.2 Stop RabbitMQ on Node 1

```bash
# Run on rabbitmq-node1
sudo rabbitmqctl stop_app
```

### 1.3 Verify Cluster Still Operational

```bash
# Run on node2 or node3 - IMPORTANT!
sudo rabbitmqctl cluster_status
# Expected: 2 nodes running (node2, node3)
# Expected: 1 node down (node1)

sudo rabbitmqctl list_queues name messages
# Queues should still be accessible
```

### 1.4 Perform OS Patching

```bash
# Run on rabbitmq-node1
# Perform your standard OS patching procedures
# Reboot if required
```

### 1.5 Start RabbitMQ After Patching

```bash
# Run on rabbitmq-node1 (after reboot if applicable)
sudo rabbitmqctl start_app
```

### 1.6 Post-Patch Verification for Node 1

```bash
# Run on rabbitmq-node1
sudo rabbitmqctl cluster_status
```

**VERIFICATION CHECKPOINT 1**

| Check | Command | Expected Result | Actual | Pass/Fail |
|-------|---------|-----------------|--------|-----------|
| Node 1 rejoined cluster | `sudo rabbitmqctl cluster_status` | 3 nodes running | | |
| No alarms on node 1 | `sudo rabbitmqctl list_alarms` | Empty | | |
| Node 1 health check | `sudo rabbitmq-diagnostics check_running` | OK | | |
| Queues accessible | `sudo rabbitmqctl list_queues name messages` | Queues listed | | |

> **STOP!** Do NOT proceed to Node 2 until ALL checks pass.
>
> **Wait minimum 5 minutes** after node rejoins before proceeding.

---

## STEP 2: Patch Node 2 (rabbitmq-node2)

> **PREREQUISITE:** Step 1 must be 100% complete and verified.

### 2.1 Pre-Patch Verification

```bash
# Run on node1 or node3
sudo rabbitmqctl cluster_status
# Confirm: All 3 nodes are running
```

### 2.2 Stop RabbitMQ on Node 2

```bash
# Run on rabbitmq-node2
sudo rabbitmqctl stop_app
```

### 2.3 Verify Cluster Still Operational

```bash
# Run on node1 or node3 - IMPORTANT!
sudo rabbitmqctl cluster_status
# Expected: 2 nodes running (node1, node3)
# Expected: 1 node down (node2)

sudo rabbitmqctl list_queues name messages
# Queues should still be accessible
```

### 2.4 Perform OS Patching

```bash
# Run on rabbitmq-node2
# Perform your standard OS patching procedures
# Reboot if required
```

### 2.5 Start RabbitMQ After Patching

```bash
# Run on rabbitmq-node2 (after reboot if applicable)
sudo rabbitmqctl start_app
```

### 2.6 Post-Patch Verification for Node 2

```bash
# Run on rabbitmq-node2
sudo rabbitmqctl cluster_status
```

**VERIFICATION CHECKPOINT 2**

| Check | Command | Expected Result | Actual | Pass/Fail |
|-------|---------|-----------------|--------|-----------|
| Node 2 rejoined cluster | `sudo rabbitmqctl cluster_status` | 3 nodes running | | |
| No alarms on node 2 | `sudo rabbitmqctl list_alarms` | Empty | | |
| Node 2 health check | `sudo rabbitmq-diagnostics check_running` | OK | | |
| Queues accessible | `sudo rabbitmqctl list_queues name messages` | Queues listed | | |

> **STOP!** Do NOT proceed to Node 3 until ALL checks pass.
>
> **Wait minimum 5 minutes** after node rejoins before proceeding.

---

## STEP 3: Patch Node 3 (rabbitmq-node3)

> **PREREQUISITE:** Steps 1 and 2 must be 100% complete and verified.

### 3.1 Pre-Patch Verification

```bash
# Run on node1 or node2
sudo rabbitmqctl cluster_status
# Confirm: All 3 nodes are running
```

### 3.2 Stop RabbitMQ on Node 3

```bash
# Run on rabbitmq-node3
sudo rabbitmqctl stop_app
```

### 3.3 Verify Cluster Still Operational

```bash
# Run on node1 or node2 - IMPORTANT!
sudo rabbitmqctl cluster_status
# Expected: 2 nodes running (node1, node2)
# Expected: 1 node down (node3)

sudo rabbitmqctl list_queues name messages
# Queues should still be accessible
```

### 3.4 Perform OS Patching

```bash
# Run on rabbitmq-node3
# Perform your standard OS patching procedures
# Reboot if required
```

### 3.5 Start RabbitMQ After Patching

```bash
# Run on rabbitmq-node3 (after reboot if applicable)
sudo rabbitmqctl start_app
```

### 3.6 Post-Patch Verification for Node 3

```bash
# Run on rabbitmq-node3
sudo rabbitmqctl cluster_status
```

**VERIFICATION CHECKPOINT 3**

| Check | Command | Expected Result | Actual | Pass/Fail |
|-------|---------|-----------------|--------|-----------|
| Node 3 rejoined cluster | `sudo rabbitmqctl cluster_status` | 3 nodes running | | |
| No alarms on node 3 | `sudo rabbitmqctl list_alarms` | Empty | | |
| Node 3 health check | `sudo rabbitmq-diagnostics check_running` | OK | | |
| Queues accessible | `sudo rabbitmqctl list_queues name messages` | Queues listed | | |

---

## Final Verification

After all three nodes are patched, perform final cluster health check.

### Run on Any Node

```bash
# 1. Verify all nodes running
sudo rabbitmqctl cluster_status

# 2. Check no alarms
sudo rabbitmqctl list_alarms

# 3. Check all queues
sudo rabbitmqctl list_queues name type leader members online messages

# 4. Verify all queue members online
# Each queue should show 3 members in 'online' column
```

### Final Verification Checklist

| Check | Expected Result | Actual | Pass/Fail |
|-------|-----------------|--------|-----------|
| All 3 nodes running | Yes | | |
| No alarms on any node | Empty | | |
| All queue members online | 3 members per queue | | |
| Message counts stable | Yes | | |

---

## Troubleshooting

### Node Won't Start After Patching

```bash
# Check logs
sudo journalctl -u rabbitmq-server -n 100

# Try starting the app
sudo rabbitmqctl start_app

# If fails, check if Erlang cookie permissions changed
sudo ls -la /var/lib/rabbitmq/.erlang.cookie
# Should be: -r-------- rabbitmq rabbitmq

# Fix permissions if needed
sudo chmod 400 /var/lib/rabbitmq/.erlang.cookie
sudo chown rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie

# Retry
sudo rabbitmqctl start_app
```

### Node Won't Rejoin Cluster

```bash
# Reset and rejoin
sudo rabbitmqctl stop_app
sudo rabbitmqctl reset
sudo rabbitmqctl join_cluster rabbit@rabbitmq-node1
sudo rabbitmqctl start_app
```

### Cluster Shows Partition

```bash
# If partition detected after patching
sudo rabbitmqctl cluster_status

# Restart the partitioned node
sudo rabbitmqctl stop_app
sudo rabbitmqctl start_app
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
| Check cluster status | `sudo rabbitmqctl cluster_status` |
| Check alarms | `sudo rabbitmqctl list_alarms` |
| Stop RabbitMQ app | `sudo rabbitmqctl stop_app` |
| Start RabbitMQ app | `sudo rabbitmqctl start_app` |
| Health check | `sudo rabbitmq-diagnostics check_running` |
| List queues | `sudo rabbitmqctl list_queues name messages` |

---

## Summary Checklist

| Step | Action | Verified |
|------|--------|----------|
| 1 | Pre-patching cluster health verified | [ ] |
| 2 | Node 1 stopped | [ ] |
| 3 | Cluster operational with 2 nodes | [ ] |
| 4 | Node 1 patched and rebooted | [ ] |
| 5 | Node 1 started and rejoined cluster | [ ] |
| 6 | **Checkpoint 1: All 3 nodes running** | [ ] |
| 7 | **Wait 5 minutes** | [ ] |
| 8 | Node 2 stopped | [ ] |
| 9 | Cluster operational with 2 nodes | [ ] |
| 10 | Node 2 patched and rebooted | [ ] |
| 11 | Node 2 started and rejoined cluster | [ ] |
| 12 | **Checkpoint 2: All 3 nodes running** | [ ] |
| 13 | **Wait 5 minutes** | [ ] |
| 14 | Node 3 stopped | [ ] |
| 15 | Cluster operational with 2 nodes | [ ] |
| 16 | Node 3 patched and rebooted | [ ] |
| 17 | Node 3 started and rejoined cluster | [ ] |
| 18 | **Checkpoint 3: All 3 nodes running** | [ ] |
| 19 | Final verification completed | [ ] |

---

## Important Reminders

1. **NEVER patch two nodes at the same time**
2. **ALWAYS verify cluster health before proceeding to next node**
3. **WAIT minimum 5 minutes between nodes**
4. **STOP and contact application team if any verification fails**
5. **Document any issues encountered**

---

*Document Version: 1.0 | Last Updated: January 2026*
