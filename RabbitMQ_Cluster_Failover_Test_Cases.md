# RabbitMQ Cluster Failover

## Test Cases and Scenarios

### Three-Node Cluster Configuration

---

## Document Information

| Field | Value |
|-------|-------|
| Document Version | 1.0 |
| Last Updated | January 2026 |
| RabbitMQ Version | 4.1.x |
| Erlang Version | 26.x |
| Operating System | Red Hat Enterprise Linux 8.x |

---

## Table of Contents

1. [Test Environment Setup](#1-test-environment-setup)
2. [Prerequisites and Initial Verification](#2-prerequisites-and-initial-verification)
3. [Test Case 1: Master Node Failure](#3-test-case-1-master-node-failure)
4. [Test Case 2: Replica Node Failure](#4-test-case-2-replica-node-failure)
5. [Test Case 3: Network Partition](#5-test-case-3-network-partition)
6. [Test Case 4: Graceful Node Shutdown](#6-test-case-4-graceful-node-shutdown)
7. [Test Case 5: Multiple Node Failure](#7-test-case-5-multiple-node-failure)
8. [Test Case 6: Disk Alarm](#8-test-case-6-disk-alarm)
9. [Test Case 7: Memory Alarm](#9-test-case-7-memory-alarm)
10. [Test Case 8: Rolling Restart](#10-test-case-8-rolling-restart)
11. [Test Case 9: Queue Mirroring Validation](#11-test-case-9-queue-mirroring-validation)
12. [Test Case 10: Client Connection Failover](#12-test-case-10-client-connection-failover)
13. [Monitoring Commands Reference](#13-monitoring-commands-reference)
14. [Recovery Procedures](#14-recovery-procedures)
15. [Test Results Template](#15-test-results-template)

---

## 1. Test Environment Setup

### Cluster Node Details

| Hostname | IP Address | Node Name |
|----------|------------|-----------|
| rabbitmq-node1 | 192.168.1.101 | rabbit@rabbitmq-node1 |
| rabbitmq-node2 | 192.168.1.102 | rabbit@rabbitmq-node2 |
| rabbitmq-node3 | 192.168.1.103 | rabbit@rabbitmq-node3 |

### Port Configuration

| Port | Purpose |
|------|---------|
| 5672 | AMQP client connections |
| 15672 | Management UI and HTTP API |
| 25672 | Inter-node communication |
| 4369 | EPMD (Erlang Port Mapper) |

---

## 2. Prerequisites and Initial Verification

### Verify Cluster Status

```bash
# Check cluster status on any node
sudo rabbitmqctl cluster_status

# Expected output should show all 3 nodes:
# Running Nodes: rabbit@rabbitmq-node1, rabbit@rabbitmq-node2, rabbit@rabbitmq-node3
```

### Verify Node Health

```bash
# Check if RabbitMQ application is running
sudo rabbitmqctl status

# Check node health
sudo rabbitmq-diagnostics check_running
sudo rabbitmq-diagnostics check_local_alarms
sudo rabbitmq-diagnostics check_port_connectivity
```

### Verify HA Policy

```bash
# List all policies
sudo rabbitmqctl list_policies

# Expected: ha-all policy with pattern ".*" and ha-mode: all
```

### Create Test Queue

```bash
# Enable management plugin if not enabled
sudo rabbitmq-plugins enable rabbitmq_management

# Create test queue using rabbitmqadmin
sudo rabbitmqadmin declare queue name=test-failover-queue durable=true

# Verify queue creation
sudo rabbitmqctl list_queues name pid slave_pids synchronised_slave_pids
```

---

## 3. Test Case 1: Master Node Failure

### Objective
Simulate failure of the node hosting the queue master and verify automatic failover.

| Attribute | Value |
|-----------|-------|
| Severity | HIGH |
| Expected Recovery | < 30 seconds |
| Expected Data Loss | Minimal (in-flight messages only) |

### Pre-Test Setup

```bash
# Identify queue master node
sudo rabbitmqctl list_queues name pid slave_pids

# Publish test messages
for i in {1..1000}; do
    sudo rabbitmqadmin publish exchange=amq.default routing_key=test-failover-queue payload="message-$i"
done

# Verify message count
sudo rabbitmqctl list_queues name messages
```

### Test Steps

#### Step 1: Record Initial State

```bash
# On any node
sudo rabbitmqctl cluster_status
sudo rabbitmqctl list_queues name messages consumers state
```

#### Step 2: Identify Queue Master

```bash
# Find which node hosts the queue master
sudo rabbitmqctl list_queues name pid

# The node name in PID indicates the master (e.g., <rabbit@rabbitmq-node1.xxx.x>)
```

#### Step 3: Stop RabbitMQ Application on Master Node

```bash
# On the queue master node (e.g., rabbitmq-node1)
sudo rabbitmqctl stop_app

# This stops the RabbitMQ application but keeps Erlang VM running
# More realistic than systemctl for testing application-level failures
```

#### Step 4: Monitor Failover

```bash
# On another node, monitor cluster status
watch -n 1 "sudo rabbitmqctl cluster_status"

# Check queue status - master should migrate
sudo rabbitmqctl list_queues name pid slave_pids state
```

#### Step 5: Verify Queue Availability

```bash
# Queue should be available on surviving nodes
sudo rabbitmqctl list_queues name messages consumers state

# Publish new message to verify writes work
sudo rabbitmqadmin publish exchange=amq.default routing_key=test-failover-queue payload="after-failover"

# Verify message count increased
sudo rabbitmqctl list_queues name messages
```

#### Step 6: Verify Client Connectivity

```bash
# Check active connections
sudo rabbitmqctl list_connections name state peer_host

# Clients should reconnect to available nodes
```

### Post-Test Recovery

```bash
# On the stopped node
sudo rabbitmqctl start_app

# Verify node rejoined cluster
sudo rabbitmqctl cluster_status

# Verify queue synchronization
sudo rabbitmqctl list_queues name synchronised_slave_pids
```

### Expected Results

- [ ] Queue master migrates to a mirror node automatically
- [ ] Queue remains available during failover
- [ ] Messages are preserved (except in-flight)
- [ ] Clients reconnect to available nodes
- [ ] Stopped node rejoins cluster after start_app
- [ ] Queue synchronizes with rejoined node

---

## 4. Test Case 2: Replica Node Failure

### Objective
Verify that failure of a replica (mirror) node has minimal impact on operations.

| Attribute | Value |
|-----------|-------|
| Severity | LOW |
| Expected Recovery | Immediate |
| Expected Data Loss | None |

### Test Steps

#### Step 1: Identify Replica Node

```bash
# List queue mirrors
sudo rabbitmqctl list_queues name pid slave_pids

# Choose a node from slave_pids (not the master)
```

#### Step 2: Stop RabbitMQ Application on Replica

```bash
# On a replica node (e.g., rabbitmq-node2)
sudo rabbitmqctl stop_app
```

#### Step 3: Verify Cluster Status

```bash
# On master or remaining replica
sudo rabbitmqctl cluster_status

# Stopped node should appear in cluster but not in running nodes
```

#### Step 4: Verify Queue Operations Continue

```bash
# Queue should continue operating normally
sudo rabbitmqctl list_queues name messages consumers state

# Publish messages
sudo rabbitmqadmin publish exchange=amq.default routing_key=test-failover-queue payload="test-message"

# Consume messages
sudo rabbitmqadmin get queue=test-failover-queue ackmode=ack_requeue_false
```

#### Step 5: Restart Failed Node

```bash
# On the stopped replica
sudo rabbitmqctl start_app

# Verify node rejoined
sudo rabbitmqctl cluster_status
```

#### Step 6: Verify Resynchronization

```bash
# Check queue synchronization status
sudo rabbitmqctl list_queues name slave_pids synchronised_slave_pids

# slave_pids and synchronised_slave_pids should match
```

### Expected Results

- [ ] No impact on queue master operations
- [ ] Messages continue to be published and consumed
- [ ] Replica rejoins cluster cleanly
- [ ] Queue resynchronizes automatically

---

## 5. Test Case 3: Network Partition

### Objective
Test cluster behavior during network partition and verify partition handling.

| Attribute | Value |
|-----------|-------|
| Severity | CRITICAL |
| Expected Recovery | 1-2 minutes |
| Partition Handling | autoheal (recommended) |

### Prerequisites

```bash
# Verify partition handling mode
sudo rabbitmqctl eval "application:get_env(rabbit, cluster_partition_handling)."

# Should return: {ok,autoheal} or {ok,pause_minority}
```

### Test Steps

#### Step 1: Record Pre-Partition State

```bash
sudo rabbitmqctl cluster_status
sudo rabbitmqctl list_queues name messages
```

#### Step 2: Create Network Partition

```bash
# On node1, block traffic from node2 and node3
sudo iptables -A INPUT -s 192.168.1.102 -j DROP
sudo iptables -A INPUT -s 192.168.1.103 -j DROP
sudo iptables -A OUTPUT -d 192.168.1.102 -j DROP
sudo iptables -A OUTPUT -d 192.168.1.103 -j DROP

# Creates partition: [node1] | [node2, node3]
```

#### Step 3: Detect Partition

```bash
# On node1 (isolated)
sudo rabbitmqctl cluster_status
# Should show node2 and node3 as partitioned

# On node2 or node3
sudo rabbitmqctl cluster_status
# Should show node1 as partitioned
```

#### Step 4: Observe Partition Handling

```bash
# Check for partition alarms
sudo rabbitmqctl list_alarms

# With autoheal: cluster will automatically heal after network restored
# With pause_minority: minority partition (node1) will pause
```

#### Step 5: Restore Network

```bash
# On node1, remove iptables rules
sudo iptables -F
```

#### Step 6: Verify Partition Healed

```bash
# Wait 30-60 seconds for autoheal
sudo rabbitmqctl cluster_status

# All nodes should be running
```

#### Step 7: Verify Queue Integrity

```bash
sudo rabbitmqctl list_queues name messages synchronised_slave_pids
```

### Expected Results

- [ ] Partition detected by all nodes
- [ ] Autoheal restarts affected nodes
- [ ] Cluster reforms after network restored
- [ ] No data corruption occurs
- [ ] Alarms clear after healing

---

## 6. Test Case 4: Graceful Node Shutdown

### Objective
Verify proper queue migration during planned maintenance.

| Attribute | Value |
|-----------|-------|
| Severity | LOW |
| Expected Recovery | Immediate |
| Expected Data Loss | None |

### Test Steps

#### Step 1: Prepare for Shutdown

```bash
# Check cluster health
sudo rabbitmqctl cluster_status
sudo rabbitmqctl list_queues name messages
```

#### Step 2: Suspend Listeners (Optional)

```bash
# Prevent new connections to the node being shut down
sudo rabbitmqctl suspend_listeners

# Existing connections continue but no new ones accepted
```

#### Step 3: Stop RabbitMQ Application

```bash
# Graceful stop
sudo rabbitmqctl stop_app

# Or full stop including Erlang VM
sudo rabbitmqctl stop
```

#### Step 4: Verify Cluster Adapts

```bash
# On remaining nodes
sudo rabbitmqctl cluster_status
sudo rabbitmqctl list_queues name messages consumers state
```

#### Step 5: Restart Node

```bash
# Start RabbitMQ
sudo rabbitmqctl start_app

# Or if fully stopped
# Start via system service, then verify:
sudo rabbitmqctl cluster_status
```

#### Step 6: Resume Listeners

```bash
# If listeners were suspended
sudo rabbitmqctl resume_listeners
```

### Expected Results

- [ ] No message loss during graceful shutdown
- [ ] Connections drain or migrate gracefully
- [ ] Queue masters migrate to other nodes
- [ ] Node rejoins cluster cleanly

---

## 7. Test Case 5: Multiple Node Failure

### Objective
Test cluster behavior when multiple nodes fail (quorum lost).

| Attribute | Value |
|-----------|-------|
| Severity | CRITICAL |
| Recovery | Manual intervention required |

> **Warning**: This test will make the cluster unavailable. Only perform in test environment.

### Scenario A: Two Nodes Fail

```bash
# Stop node2
sudo rabbitmqctl stop_app  # on rabbitmq-node2

# Stop node3
sudo rabbitmqctl stop_app  # on rabbitmq-node3

# On node1 (remaining node)
sudo rabbitmqctl cluster_status
# Cluster loses quorum
```

### Recovery Procedure

```bash
# Start nodes in sequence
# On node2
sudo rabbitmqctl start_app

# On node3
sudo rabbitmqctl start_app

# Verify recovery
sudo rabbitmqctl cluster_status
```

### Scenario B: All Nodes Down

```bash
# If all nodes stopped, start the last node that was running first

# Use force_boot if needed
sudo rabbitmqctl force_boot
sudo rabbitmqctl start_app

# Then start other nodes
```

### Expected Results

- [ ] Cluster becomes unavailable with quorum lost
- [ ] Recovery requires manual node restart
- [ ] Data preserved after recovery (if persistence enabled)

---

## 8. Test Case 6: Disk Alarm

### Objective
Verify RabbitMQ behavior when disk space is low.

| Attribute | Value |
|-----------|-------|
| Severity | HIGH |
| Impact | Publishers blocked |

### Test Steps

#### Step 1: Check Current Disk Limit

```bash
sudo rabbitmqctl eval "rabbit_disk_monitor:get_disk_free_limit()."
```

#### Step 2: Trigger Disk Alarm (Test Only)

```bash
# Temporarily set high disk limit to trigger alarm
sudo rabbitmqctl eval "rabbit_disk_monitor:set_disk_free_limit({mem_relative, 5.0})."

# Or create large file to fill disk
sudo fallocate -l 40G /var/lib/rabbitmq/large-file
```

#### Step 3: Verify Alarm Triggered

```bash
sudo rabbitmqctl list_alarms
# Should show: {resource_limit,disk,rabbit@node}

# Check node status
sudo rabbitmq-diagnostics check_local_alarms
```

#### Step 4: Verify Publisher Blocking

```bash
# Try to publish - should be blocked
sudo rabbitmqadmin publish exchange=amq.default routing_key=test-queue payload="test"
# Will hang or return error
```

#### Step 5: Clear Alarm

```bash
# Remove test file
sudo rm /var/lib/rabbitmq/large-file

# Or restore normal disk limit
sudo rabbitmqctl eval "rabbit_disk_monitor:set_disk_free_limit({mem_relative, 1.0})."
```

#### Step 6: Verify Recovery

```bash
sudo rabbitmqctl list_alarms
# Should be empty

# Publishing should work again
sudo rabbitmqadmin publish exchange=amq.default routing_key=test-queue payload="test"
```

### Expected Results

- [ ] Disk alarm triggers when limit reached
- [ ] Publishers are blocked
- [ ] Consumers continue to work
- [ ] Alarm clears when disk freed
- [ ] Publishers resume automatically

---

## 9. Test Case 7: Memory Alarm

### Objective
Test RabbitMQ behavior under memory pressure.

| Attribute | Value |
|-----------|-------|
| Severity | HIGH |
| Impact | Publishers blocked |

### Test Steps

#### Step 1: Check Memory Configuration

```bash
sudo rabbitmqctl eval "vm_memory_monitor:get_memory_limit()."
sudo rabbitmqctl eval "vm_memory_monitor:get_vm_memory_high_watermark()."
```

#### Step 2: Trigger Memory Alarm (Test Only)

```bash
# Temporarily lower memory threshold
sudo rabbitmqctl eval "vm_memory_monitor:set_vm_memory_high_watermark(0.1)."

# Or publish many large messages
for i in {1..100000}; do
    sudo rabbitmqadmin publish exchange=amq.default routing_key=test-queue payload="$(head -c 10000 /dev/urandom | base64)"
done
```

#### Step 3: Verify Alarm

```bash
sudo rabbitmqctl list_alarms
# Should show: {resource_limit,memory,rabbit@node}

sudo rabbitmqctl status | grep -A 10 memory
```

#### Step 4: Clear Alarm

```bash
# Consume messages to reduce memory
sudo rabbitmqadmin get queue=test-queue count=10000 ackmode=ack_requeue_false

# Or restore normal threshold
sudo rabbitmqctl eval "vm_memory_monitor:set_vm_memory_high_watermark(0.4)."
```

#### Step 5: Verify Recovery

```bash
sudo rabbitmqctl list_alarms
# Should be empty
```

### Expected Results

- [ ] Memory alarm triggers at watermark
- [ ] Publishers are blocked/throttled
- [ ] Consumers continue processing
- [ ] Alarm clears when memory freed
- [ ] Publishers resume automatically

---

## 10. Test Case 8: Rolling Restart

### Objective
Perform rolling restart with zero downtime.

| Attribute | Value |
|-----------|-------|
| Severity | LOW |
| Expected Downtime | None |

### Test Steps

#### Step 1: Verify Initial State

```bash
sudo rabbitmqctl cluster_status
sudo rabbitmqctl list_queues name messages
```

#### Step 2: Start Continuous Message Flow

```bash
# Keep producer and consumer running throughout test
# Monitor for any interruptions
```

#### Step 3: Restart Replica 1 (node3)

```bash
# On rabbitmq-node3
sudo rabbitmqctl stop_app
sudo rabbitmqctl start_app

# Wait for sync
sudo rabbitmqctl list_queues name synchronised_slave_pids
```

#### Step 4: Restart Replica 2 (node2)

```bash
# On rabbitmq-node2
sudo rabbitmqctl stop_app
sudo rabbitmqctl start_app

# Wait for sync
sudo rabbitmqctl list_queues name synchronised_slave_pids
```

#### Step 5: Restart Primary (node1)

```bash
# On rabbitmq-node1 (if it's queue master, queue will migrate)
sudo rabbitmqctl stop_app
sudo rabbitmqctl start_app
```

#### Step 6: Verify Complete Cluster

```bash
sudo rabbitmqctl cluster_status
sudo rabbitmqctl list_queues name synchronised_slave_pids
```

### Expected Results

- [ ] Zero message loss
- [ ] Continuous availability
- [ ] All nodes synchronized after restart
- [ ] No client disconnections (with proper reconnect logic)

---

## 11. Test Case 9: Queue Mirroring Validation

### Objective
Verify queue mirroring is working correctly.

| Attribute | Value |
|-----------|-------|
| Severity | MEDIUM |
| Purpose | Configuration validation |

### Test Steps

#### Step 1: Check HA Policy

```bash
sudo rabbitmqctl list_policies

# Verify ha-all policy exists with:
# Pattern: .*
# Definition: {"ha-mode":"all","ha-sync-mode":"automatic"}
```

#### Step 2: Create Test Queue

```bash
sudo rabbitmqadmin declare queue name=mirror-test durable=true
```

#### Step 3: Verify Mirroring

```bash
sudo rabbitmqctl list_queues name pid slave_pids

# Queue should have 2 slave_pids (mirrors on other nodes)
```

#### Step 4: Publish Messages

```bash
for i in {1..1000}; do
    sudo rabbitmqadmin publish exchange=amq.default routing_key=mirror-test payload="message-$i"
done
```

#### Step 5: Verify Synchronization

```bash
sudo rabbitmqctl list_queues name messages slave_pids synchronised_slave_pids

# synchronised_slave_pids should match slave_pids
```

#### Step 6: Test Failover

```bash
# Identify and stop master node for the queue
# Verify messages preserved on mirrors

sudo rabbitmqctl stop_app  # on master

# On another node
sudo rabbitmqctl list_queues name messages
# Message count should be preserved
```

### Expected Results

- [ ] Queue has mirrors on all nodes
- [ ] Messages replicated to all mirrors
- [ ] Synchronization maintained
- [ ] Messages preserved during failover

---

## 12. Test Case 10: Client Connection Failover

### Objective
Verify clients reconnect automatically during node failures.

| Attribute | Value |
|-----------|-------|
| Severity | MEDIUM |
| Expected Reconnect | < 10 seconds |

### Prerequisites

Client must be configured with multiple node addresses.

### Test Steps

#### Step 1: Check Current Connections

```bash
sudo rabbitmqctl list_connections name peer_host peer_port state
```

#### Step 2: Identify Client's Node

```bash
# Note which node the client is connected to
sudo rabbitmqctl list_connections name peer_host node
```

#### Step 3: Stop Connected Node

```bash
# On the node client is connected to
sudo rabbitmqctl stop_app
```

#### Step 4: Monitor Reconnection

```bash
# On remaining nodes
watch -n 1 "sudo rabbitmqctl list_connections name peer_host state"

# Client should appear on a different node
```

#### Step 5: Verify Operations Continue

```bash
# Check if messages are flowing
sudo rabbitmqctl list_queues name messages consumers
```

#### Step 6: Restart Node

```bash
sudo rabbitmqctl start_app
```

### Expected Results

- [ ] Client detects connection loss
- [ ] Client reconnects to available node
- [ ] Message flow continues
- [ ] No manual intervention required

---

## 13. Monitoring Commands Reference

### Cluster Health

```bash
# Cluster status
sudo rabbitmqctl cluster_status

# Node health checks
sudo rabbitmq-diagnostics check_running
sudo rabbitmq-diagnostics check_local_alarms
sudo rabbitmq-diagnostics check_port_connectivity
sudo rabbitmq-diagnostics check_virtual_hosts

# Comprehensive health check
sudo rabbitmq-diagnostics status
```

### Queue Information

```bash
# List all queues with details
sudo rabbitmqctl list_queues name messages consumers memory state pid slave_pids synchronised_slave_pids

# Queue details for specific queue
sudo rabbitmqctl list_queues name messages | grep test-queue
```

### Connection Information

```bash
# List connections
sudo rabbitmqctl list_connections name state user peer_host peer_port

# List channels
sudo rabbitmqctl list_channels connection number consumer_count messages_unacknowledged
```

### Resource Monitoring

```bash
# Memory usage
sudo rabbitmqctl status | grep -A 20 memory

# Disk usage
sudo rabbitmq-diagnostics check_alarms

# Active alarms
sudo rabbitmqctl list_alarms
```

### Replication Status

```bash
# Queue mirroring status
sudo rabbitmqctl list_queues name slave_pids synchronised_slave_pids

# Sync specific queue manually
sudo rabbitmqctl sync_queue <queue_name>

# Cancel sync
sudo rabbitmqctl cancel_sync_queue <queue_name>
```

---

## 14. Recovery Procedures

### Node Won't Start

```bash
# Check status
sudo rabbitmqctl status

# If cluster issue, try force boot
sudo rabbitmqctl force_boot
sudo rabbitmqctl start_app

# If still fails, reset and rejoin
sudo rabbitmqctl stop_app
sudo rabbitmqctl reset
sudo rabbitmqctl join_cluster rabbit@rabbitmq-node1
sudo rabbitmqctl start_app
```

### Remove Failed Node from Cluster

```bash
# On a running node, remove the failed node
sudo rabbitmqctl forget_cluster_node rabbit@failed-node
```

### Reset Node Completely

```bash
sudo rabbitmqctl stop_app
sudo rabbitmqctl reset
sudo rabbitmqctl start_app
```

### Force Sync Queues

```bash
# List unsynchronized queues
sudo rabbitmqctl list_queues name slave_pids synchronised_slave_pids | grep -v "^\s*$"

# Sync specific queue
sudo rabbitmqctl sync_queue queue_name

# Sync all queues
for queue in $(sudo rabbitmqctl list_queues -q name); do
    sudo rabbitmqctl sync_queue "$queue"
done
```

### Clear Alarms

```bash
# Check current alarms
sudo rabbitmqctl list_alarms

# Clear file descriptor alarm (increase limits)
# Clear disk alarm (free disk space)
# Clear memory alarm (increase memory or consume messages)

# Verify alarms cleared
sudo rabbitmq-diagnostics check_local_alarms
```

---

## 15. Test Results Template

### Test Execution Record

| Field | Value |
|-------|-------|
| Test Date | |
| Tester Name | |
| Environment | |
| RabbitMQ Version | |
| Erlang Version | |
| HA Policy | ha-all / ha-exactly:2 |

### Test Results

| Test Case | Status | Failover Time | Message Loss | Notes |
|-----------|--------|---------------|--------------|-------|
| TC1: Master Node Failure | | | | |
| TC2: Replica Node Failure | | | | |
| TC3: Network Partition | | | | |
| TC4: Graceful Shutdown | | | | |
| TC5: Multiple Node Failure | | | | |
| TC6: Disk Alarm | | | | |
| TC7: Memory Alarm | | | | |
| TC8: Rolling Restart | | | | |
| TC9: Queue Mirroring | | | | |
| TC10: Client Failover | | | | |

### Success Criteria

- [ ] All failovers complete within 30 seconds
- [ ] Message loss < 0.1% (in-flight only)
- [ ] Cluster recovers to full health
- [ ] No manual intervention for automatic scenarios
- [ ] Clients reconnect within 10 seconds

### Issues Found

| # | Description | Severity | Resolution |
|---|-------------|----------|------------|
| 1 | | | |
| 2 | | | |

### Overall Result

**[ ] PASS  [ ] FAIL  [ ] PASS WITH CONDITIONS**

---

## Quick Reference: Key Commands

| Action | Command |
|--------|---------|
| Cluster status | `sudo rabbitmqctl cluster_status` |
| Stop app | `sudo rabbitmqctl stop_app` |
| Start app | `sudo rabbitmqctl start_app` |
| List queues | `sudo rabbitmqctl list_queues name messages` |
| List connections | `sudo rabbitmqctl list_connections` |
| List alarms | `sudo rabbitmqctl list_alarms` |
| Health check | `sudo rabbitmq-diagnostics check_running` |
| Force boot | `sudo rabbitmqctl force_boot` |
| Reset node | `sudo rabbitmqctl reset` |
| Join cluster | `sudo rabbitmqctl join_cluster rabbit@node` |
| Forget node | `sudo rabbitmqctl forget_cluster_node rabbit@node` |
| Sync queue | `sudo rabbitmqctl sync_queue <name>` |
| List policies | `sudo rabbitmqctl list_policies` |

---

*Document Version: 1.0 | Last Updated: January 2026*
