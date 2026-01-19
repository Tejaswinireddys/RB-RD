# RabbitMQ Cluster Failover

## Test Cases and Scenarios

### Three-Node Cluster with Quorum Queues

---

## Document Information

| Field | Value |
|-------|-------|
| Document Version | 2.0 |
| Last Updated | January 2026 |
| RabbitMQ Version | 4.1.x |
| Erlang Version | 26.x |
| Operating System | Red Hat Enterprise Linux 8.x |
| Queue Type | Quorum Queues |

---

## Table of Contents

1. [Quorum Queues Overview](#1-quorum-queues-overview)
2. [Test Environment Setup](#2-test-environment-setup)
3. [Prerequisites and Initial Verification](#3-prerequisites-and-initial-verification)
4. [Test Case 1: Leader Node Failure](#4-test-case-1-leader-node-failure)
5. [Test Case 2: Follower Node Failure](#5-test-case-2-follower-node-failure)
6. [Test Case 3: Network Partition](#6-test-case-3-network-partition)
7. [Test Case 4: Graceful Node Shutdown](#7-test-case-4-graceful-node-shutdown)
8. [Test Case 5: Quorum Loss](#8-test-case-5-quorum-loss)
9. [Test Case 6: Disk Alarm](#9-test-case-6-disk-alarm)
10. [Test Case 7: Memory Alarm](#10-test-case-7-memory-alarm)
11. [Test Case 8: Rolling Restart](#11-test-case-8-rolling-restart)
12. [Test Case 9: Quorum Queue Replication Validation](#12-test-case-9-quorum-queue-replication-validation)
13. [Test Case 10: Client Connection Failover](#13-test-case-10-client-connection-failover)
14. [Monitoring Commands Reference](#14-monitoring-commands-reference)
15. [Recovery Procedures](#15-recovery-procedures)
16. [Test Results Template](#16-test-results-template)

---

## 1. Quorum Queues Overview

### What are Quorum Queues?

Quorum queues are the recommended queue type for high availability in RabbitMQ 4.x. They replace classic mirrored queues and HA policies.

| Feature | Quorum Queues | Classic Mirrored (Deprecated) |
|---------|---------------|-------------------------------|
| Replication | Raft consensus protocol | Async mirroring with policies |
| Data Safety | Strong consistency | Eventually consistent |
| Configuration | Queue argument at declaration | HA policies |
| Failover | Automatic leader election | Mirror promotion |
| Performance | Optimized for safety | Higher throughput, less safe |

### Key Concepts

| Term | Description |
|------|-------------|
| **Leader** | Node that handles all reads and writes for the queue |
| **Follower** | Nodes that replicate data from leader |
| **Quorum** | Majority of nodes required for operations (2 of 3) |
| **Raft** | Consensus algorithm used for replication |
| **Replication Factor** | Number of nodes queue is replicated to (default: cluster size) |

### Quorum Queue Declaration

```bash
# Declare quorum queue using rabbitmqadmin
rabbitmqadmin declare queue name=my-quorum-queue durable=true arguments='{"x-queue-type":"quorum"}'

# Or via rabbitmqctl
rabbitmqctl eval 'rabbit_amqqueue:declare(
    rabbit_misc:r(<<"/">>, queue, <<"my-quorum-queue">>),
    true, false,
    [{<<"x-queue-type">>, longstr, <<"quorum">>}],
    none, <<"guest">>
).'
```

---

## 2. Test Environment Setup

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

### Quorum Requirements

| Cluster Size | Quorum | Tolerated Failures |
|--------------|--------|-------------------|
| 3 nodes | 2 | 1 node |
| 5 nodes | 3 | 2 nodes |
| 7 nodes | 4 | 3 nodes |

---

## 3. Prerequisites and Initial Verification

### Verify Cluster Status

```bash
# Check cluster status
sudo rabbitmqctl cluster_status

# Expected: All 3 nodes in running nodes list
```

### Verify Node Health

```bash
# Check if RabbitMQ is running
sudo rabbitmqctl status

# Run health checks
sudo rabbitmq-diagnostics check_running
sudo rabbitmq-diagnostics check_local_alarms
sudo rabbitmq-diagnostics check_port_connectivity
```

### Create Test Quorum Queue

```bash
# Enable management plugin
sudo rabbitmq-plugins enable rabbitmq_management

# Create quorum queue
sudo rabbitmqadmin declare queue name=test-quorum-queue durable=true \
    arguments='{"x-queue-type":"quorum"}'
```

### Verify Quorum Queue Created

```bash
# List queues with type
sudo rabbitmqctl list_queues name type leader replicas

# Expected output:
# test-quorum-queue   quorum   rabbit@rabbitmq-node1   [rabbit@rabbitmq-node1, rabbit@rabbitmq-node2, rabbit@rabbitmq-node3]
```

### Verify Queue Members

```bash
# Get detailed quorum queue info
sudo rabbitmqctl list_queues name type leader members online

# Check quorum queue status
sudo rabbitmq-diagnostics check_if_node_is_quorum_critical
```

---

## 4. Test Case 1: Leader Node Failure

### Objective
Simulate failure of the quorum queue leader node and verify automatic leader election.

| Attribute | Value |
|-----------|-------|
| Severity | HIGH |
| Expected Recovery | < 30 seconds |
| Expected Data Loss | None (committed messages) |

### Pre-Test Setup

```bash
# Identify queue leader
sudo rabbitmqctl list_queues name type leader

# Publish test messages
for i in {1..1000}; do
    sudo rabbitmqadmin publish exchange=amq.default routing_key=test-quorum-queue payload="message-$i"
done

# Verify message count
sudo rabbitmqctl list_queues name messages
```

### Test Steps

#### Step 1: Record Initial State

```bash
# Record cluster and queue state
sudo rabbitmqctl cluster_status
sudo rabbitmqctl list_queues name type leader members online messages
```

#### Step 2: Identify Queue Leader

```bash
# Find leader node
sudo rabbitmqctl list_queues name leader

# Example output: test-quorum-queue    rabbit@rabbitmq-node1
```

#### Step 3: Stop RabbitMQ on Leader Node

```bash
# On the leader node (e.g., rabbitmq-node1)
sudo rabbitmqctl stop_app
```

#### Step 4: Monitor Leader Election

```bash
# On another node, watch for new leader
watch -n 1 "sudo rabbitmqctl list_queues name leader online"

# New leader should be elected from followers
```

#### Step 5: Verify Queue Availability

```bash
# Check queue is available with new leader
sudo rabbitmqctl list_queues name type leader messages

# Publish new message
sudo rabbitmqadmin publish exchange=amq.default routing_key=test-quorum-queue payload="after-failover"

# Verify message count
sudo rabbitmqctl list_queues name messages
```

#### Step 6: Verify Quorum Maintained

```bash
# Check online members (should be 2)
sudo rabbitmqctl list_queues name members online

# Quorum (2 of 3) is maintained
```

### Post-Test Recovery

```bash
# On the stopped node
sudo rabbitmqctl start_app

# Verify node rejoined
sudo rabbitmqctl cluster_status

# Verify queue has all members online
sudo rabbitmqctl list_queues name members online
```

### Expected Results

- [ ] New leader elected automatically from followers
- [ ] Queue remains available during failover
- [ ] All committed messages preserved
- [ ] New writes succeed on new leader
- [ ] Stopped node rejoins as follower
- [ ] All members back online after recovery

---

## 5. Test Case 2: Follower Node Failure

### Objective
Verify that failure of a follower node has minimal impact on queue operations.

| Attribute | Value |
|-----------|-------|
| Severity | LOW |
| Expected Recovery | Immediate |
| Expected Data Loss | None |

### Test Steps

#### Step 1: Identify Follower Node

```bash
# List queue members
sudo rabbitmqctl list_queues name leader members

# Choose a node that is NOT the leader
```

#### Step 2: Stop RabbitMQ on Follower

```bash
# On a follower node (not the leader)
sudo rabbitmqctl stop_app
```

#### Step 3: Verify Queue Operations Continue

```bash
# Queue should continue with 2 members (quorum maintained)
sudo rabbitmqctl list_queues name leader online messages

# Publish messages
sudo rabbitmqadmin publish exchange=amq.default routing_key=test-quorum-queue payload="test-message"

# Consume messages
sudo rabbitmqadmin get queue=test-quorum-queue ackmode=ack_requeue_false
```

#### Step 4: Verify Quorum Status

```bash
# Check online members
sudo rabbitmqctl list_queues name members online

# Should show 2 online members
```

#### Step 5: Restart Follower

```bash
# On the stopped follower
sudo rabbitmqctl start_app

# Verify node rejoined
sudo rabbitmqctl list_queues name members online

# All 3 members should be online
```

### Expected Results

- [ ] No impact on queue operations
- [ ] Leader continues handling requests
- [ ] Quorum maintained with 2 nodes
- [ ] Follower rejoins and syncs automatically
- [ ] All members online after recovery

---

## 6. Test Case 3: Network Partition

### Objective
Test cluster behavior during network partition with quorum queues.

| Attribute | Value |
|-----------|-------|
| Severity | CRITICAL |
| Expected Recovery | 1-2 minutes |
| Partition Handling | Quorum-based |

### Prerequisites

```bash
# Check partition handling
sudo rabbitmqctl eval "application:get_env(rabbit, cluster_partition_handling)."
```

### Test Steps

#### Step 1: Record Pre-Partition State

```bash
sudo rabbitmqctl cluster_status
sudo rabbitmqctl list_queues name leader members online messages
```

#### Step 2: Create Network Partition

```bash
# On node1, isolate from node2 and node3
sudo iptables -A INPUT -s 192.168.1.102 -j DROP
sudo iptables -A INPUT -s 192.168.1.103 -j DROP
sudo iptables -A OUTPUT -d 192.168.1.102 -j DROP
sudo iptables -A OUTPUT -d 192.168.1.103 -j DROP

# Creates: [node1] | [node2, node3]
```

#### Step 3: Observe Quorum Behavior

```bash
# On isolated node (node1)
sudo rabbitmqctl list_queues name leader online

# If node1 was leader:
# - It loses quorum (only 1 of 3 nodes)
# - Queue becomes unavailable on this node

# On majority side (node2 or node3)
sudo rabbitmqctl list_queues name leader online

# - Quorum maintained (2 of 3 nodes)
# - New leader elected if needed
# - Queue remains available
```

#### Step 4: Test Writes on Both Sides

```bash
# On isolated node (minority)
sudo rabbitmqadmin publish exchange=amq.default routing_key=test-quorum-queue payload="minority-write"
# Should FAIL - no quorum

# On majority side
sudo rabbitmqadmin publish exchange=amq.default routing_key=test-quorum-queue payload="majority-write"
# Should SUCCEED - quorum maintained
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

sudo rabbitmqctl cluster_status
sudo rabbitmqctl list_queues name leader members online
```

### Expected Results

- [ ] Minority side loses quorum and becomes unavailable
- [ ] Majority side maintains quorum and availability
- [ ] Writes only succeed on majority side
- [ ] Cluster heals after network restored
- [ ] No data corruption or loss

---

## 7. Test Case 4: Graceful Node Shutdown

### Objective
Verify proper leader transfer during planned maintenance.

| Attribute | Value |
|-----------|-------|
| Severity | LOW |
| Expected Recovery | Immediate |
| Expected Data Loss | None |

### Test Steps

#### Step 1: Check Current Leader

```bash
sudo rabbitmqctl list_queues name leader
```

#### Step 2: Transfer Leadership (If Shutting Down Leader)

```bash
# Transfer leadership before shutdown (optional but recommended)
sudo rabbitmqctl eval 'rabbit_quorum_queue:transfer_leadership(<<"test-quorum-queue">>, <<"rabbit@rabbitmq-node2">>).'

# Verify new leader
sudo rabbitmqctl list_queues name leader
```

#### Step 3: Stop RabbitMQ Application

```bash
# On the node being shut down
sudo rabbitmqctl stop_app
```

#### Step 4: Verify Queue Available

```bash
# On other nodes
sudo rabbitmqctl list_queues name leader online messages
```

#### Step 5: Restart Node

```bash
sudo rabbitmqctl start_app

# Verify all members online
sudo rabbitmqctl list_queues name members online
```

### Expected Results

- [ ] Leadership transferred smoothly (if specified)
- [ ] No message loss
- [ ] Queue remains available
- [ ] Node rejoins as follower

---

## 8. Test Case 5: Quorum Loss

### Objective
Test behavior when quorum is lost (majority of nodes unavailable).

| Attribute | Value |
|-----------|-------|
| Severity | CRITICAL |
| Recovery | Manual intervention |

> **Warning**: This will make the queue unavailable. Test environment only.

### Test Steps

#### Step 1: Stop Two Nodes

```bash
# Stop node2
sudo rabbitmqctl stop_app  # on rabbitmq-node2

# Stop node3
sudo rabbitmqctl stop_app  # on rabbitmq-node3
```

#### Step 2: Observe Queue Unavailability

```bash
# On remaining node (node1)
sudo rabbitmqctl list_queues name leader online

# Queue should show only 1 online member
# Operations will fail due to no quorum
```

#### Step 3: Attempt Write (Should Fail)

```bash
sudo rabbitmqadmin publish exchange=amq.default routing_key=test-quorum-queue payload="no-quorum-write"
# Expected: Error - queue unavailable
```

#### Step 4: Recovery

```bash
# Start nodes to restore quorum
sudo rabbitmqctl start_app  # on node2
sudo rabbitmqctl start_app  # on node3

# Verify quorum restored
sudo rabbitmqctl list_queues name members online
```

### Expected Results

- [ ] Queue unavailable without quorum
- [ ] Writes fail cleanly with error
- [ ] Reads may work for already-fetched messages
- [ ] Full recovery after quorum restored
- [ ] No data corruption

---

## 9. Test Case 6: Disk Alarm

### Objective
Verify behavior when disk space is low.

| Attribute | Value |
|-----------|-------|
| Severity | HIGH |
| Impact | Publishers blocked |

### Test Steps

#### Step 1: Check Disk Status

```bash
sudo rabbitmqctl eval "rabbit_disk_monitor:get_disk_free_limit()."
```

#### Step 2: Trigger Disk Alarm

```bash
# Set high threshold to trigger alarm
sudo rabbitmqctl eval "rabbit_disk_monitor:set_disk_free_limit({mem_relative, 5.0})."
```

#### Step 3: Verify Alarm

```bash
sudo rabbitmqctl list_alarms
# Should show disk alarm

sudo rabbitmq-diagnostics check_local_alarms
```

#### Step 4: Test Publisher Blocking

```bash
# Publish attempt will be blocked
sudo rabbitmqadmin publish exchange=amq.default routing_key=test-quorum-queue payload="blocked"
```

#### Step 5: Clear Alarm

```bash
# Restore normal threshold
sudo rabbitmqctl eval "rabbit_disk_monitor:set_disk_free_limit({mem_relative, 1.0})."

# Verify alarm cleared
sudo rabbitmqctl list_alarms
```

### Expected Results

- [ ] Disk alarm triggers
- [ ] Publishers blocked
- [ ] Consumers continue working
- [ ] Alarm clears when resolved

---

## 10. Test Case 7: Memory Alarm

### Objective
Test behavior under memory pressure.

| Attribute | Value |
|-----------|-------|
| Severity | HIGH |
| Impact | Publishers blocked |

### Test Steps

#### Step 1: Check Memory Configuration

```bash
sudo rabbitmqctl eval "vm_memory_monitor:get_memory_limit()."
```

#### Step 2: Trigger Memory Alarm

```bash
# Lower threshold to trigger alarm
sudo rabbitmqctl eval "vm_memory_monitor:set_vm_memory_high_watermark(0.1)."
```

#### Step 3: Verify Alarm

```bash
sudo rabbitmqctl list_alarms
sudo rabbitmq-diagnostics check_local_alarms
```

#### Step 4: Clear Alarm

```bash
# Restore normal threshold
sudo rabbitmqctl eval "vm_memory_monitor:set_vm_memory_high_watermark(0.4)."

sudo rabbitmqctl list_alarms
```

### Expected Results

- [ ] Memory alarm triggers
- [ ] Publishers blocked/throttled
- [ ] Consumers continue
- [ ] Alarm clears when resolved

---

## 11. Test Case 8: Rolling Restart

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
sudo rabbitmqctl list_queues name leader members online
```

#### Step 2: Restart Followers First

```bash
# Identify followers (not leader)
sudo rabbitmqctl list_queues name leader

# Restart follower 1
sudo rabbitmqctl stop_app  # on follower node
sudo rabbitmqctl start_app

# Wait for sync
sudo rabbitmqctl list_queues name members online

# Restart follower 2
sudo rabbitmqctl stop_app  # on other follower
sudo rabbitmqctl start_app
```

#### Step 3: Transfer Leadership Then Restart Leader

```bash
# Transfer leadership away from node being restarted
sudo rabbitmqctl eval 'rabbit_quorum_queue:transfer_leadership(<<"test-quorum-queue">>, <<"rabbit@rabbitmq-node2">>).'

# Verify transfer
sudo rabbitmqctl list_queues name leader

# Restart old leader
sudo rabbitmqctl stop_app
sudo rabbitmqctl start_app
```

#### Step 4: Verify Complete Cluster

```bash
sudo rabbitmqctl cluster_status
sudo rabbitmqctl list_queues name leader members online
```

### Expected Results

- [ ] Zero downtime maintained
- [ ] Quorum always preserved
- [ ] No message loss
- [ ] All members online after restart

---

## 12. Test Case 9: Quorum Queue Replication Validation

### Objective
Verify data is correctly replicated across all queue members.

| Attribute | Value |
|-----------|-------|
| Severity | MEDIUM |
| Purpose | Configuration validation |

### Test Steps

#### Step 1: Create Test Queue

```bash
sudo rabbitmqadmin declare queue name=replication-test durable=true \
    arguments='{"x-queue-type":"quorum"}'
```

#### Step 2: Verify Members

```bash
sudo rabbitmqctl list_queues name type members online

# Should show all 3 nodes as members
```

#### Step 3: Publish Messages

```bash
for i in {1..1000}; do
    sudo rabbitmqadmin publish exchange=amq.default routing_key=replication-test payload="message-$i"
done
```

#### Step 4: Verify Replication

```bash
# Check message count
sudo rabbitmqctl list_queues name messages

# Stop leader and verify messages on new leader
sudo rabbitmqctl list_queues name leader
sudo rabbitmqctl stop_app  # on leader

# On another node
sudo rabbitmqctl list_queues name messages
# Count should match
```

#### Step 5: Consume and Verify

```bash
# Consume all messages
for i in {1..1000}; do
    sudo rabbitmqadmin get queue=replication-test ackmode=ack_requeue_false
done

# Verify queue empty
sudo rabbitmqctl list_queues name messages
```

### Expected Results

- [ ] Queue replicated to all members
- [ ] Messages preserved after leader failure
- [ ] No duplicate or lost messages

---

## 13. Test Case 10: Client Connection Failover

### Objective
Verify clients reconnect during node failures.

| Attribute | Value |
|-----------|-------|
| Severity | MEDIUM |
| Expected Reconnect | < 10 seconds |

### Test Steps

#### Step 1: Check Connections

```bash
sudo rabbitmqctl list_connections name peer_host node
```

#### Step 2: Stop Connected Node

```bash
# Identify and stop node with client connection
sudo rabbitmqctl stop_app  # on connected node
```

#### Step 3: Monitor Reconnection

```bash
# On other nodes
watch -n 1 "sudo rabbitmqctl list_connections name peer_host state"
```

#### Step 4: Verify Operations Continue

```bash
sudo rabbitmqctl list_queues name messages consumers
```

### Expected Results

- [ ] Client reconnects automatically
- [ ] Operations continue on new node
- [ ] No manual intervention required

---

## 14. Monitoring Commands Reference

### Quorum Queue Commands

```bash
# List quorum queues with details
sudo rabbitmqctl list_queues name type leader members online messages

# Check if node is quorum critical
sudo rabbitmq-diagnostics check_if_node_is_quorum_critical

# Get quorum queue info
sudo rabbitmqctl list_queues name type arguments
```

### Cluster Health

```bash
# Cluster status
sudo rabbitmqctl cluster_status

# Node health
sudo rabbitmq-diagnostics check_running
sudo rabbitmq-diagnostics check_local_alarms
sudo rabbitmq-diagnostics status
```

### Queue Information

```bash
# List all queues
sudo rabbitmqctl list_queues name type messages consumers memory

# Quorum queue specific
sudo rabbitmqctl list_queues name leader members online
```

### Connection Information

```bash
sudo rabbitmqctl list_connections name state user peer_host node
sudo rabbitmqctl list_channels connection consumer_count
```

### Resource Monitoring

```bash
sudo rabbitmqctl list_alarms
sudo rabbitmqctl status | grep -A 20 memory
```

---

## 15. Recovery Procedures

### Node Won't Start

```bash
# Check status
sudo rabbitmqctl status

# Force boot if cluster issue
sudo rabbitmqctl force_boot
sudo rabbitmqctl start_app

# Reset and rejoin if needed
sudo rabbitmqctl stop_app
sudo rabbitmqctl reset
sudo rabbitmqctl join_cluster rabbit@rabbitmq-node1
sudo rabbitmqctl start_app
```

### Remove Failed Node

```bash
sudo rabbitmqctl forget_cluster_node rabbit@failed-node
```

### Quorum Queue Recovery

```bash
# Delete member from quorum queue (if node permanently gone)
sudo rabbitmqctl delete_member <queue_name> <node_name>

# Add member to quorum queue
sudo rabbitmqctl add_member <queue_name> <node_name>

# Example:
sudo rabbitmqctl delete_member test-quorum-queue rabbit@failed-node
sudo rabbitmqctl add_member test-quorum-queue rabbit@new-node
```

### Force Leader Election

```bash
# Transfer leadership to specific node
sudo rabbitmqctl eval 'rabbit_quorum_queue:transfer_leadership(<<"queue-name">>, <<"rabbit@target-node">>).'
```

---

## 16. Test Results Template

### Test Execution Record

| Field | Value |
|-------|-------|
| Test Date | |
| Tester Name | |
| Environment | |
| RabbitMQ Version | |
| Queue Type | Quorum |
| Cluster Size | 3 nodes |

### Test Results

| Test Case | Status | Failover Time | Data Loss | Notes |
|-----------|--------|---------------|-----------|-------|
| TC1: Leader Node Failure | | | | |
| TC2: Follower Node Failure | | | | |
| TC3: Network Partition | | | | |
| TC4: Graceful Shutdown | | | | |
| TC5: Quorum Loss | | | | |
| TC6: Disk Alarm | | | | |
| TC7: Memory Alarm | | | | |
| TC8: Rolling Restart | | | | |
| TC9: Replication Validation | | | | |
| TC10: Client Failover | | | | |

### Success Criteria

- [ ] Leader election completes within 30 seconds
- [ ] No data loss for committed messages
- [ ] Cluster recovers automatically
- [ ] Quorum maintained with 1 node failure
- [ ] Clients reconnect within 10 seconds

---

## Quick Reference: Key Commands

| Action | Command |
|--------|---------|
| Cluster status | `sudo rabbitmqctl cluster_status` |
| Stop app | `sudo rabbitmqctl stop_app` |
| Start app | `sudo rabbitmqctl start_app` |
| List queues (quorum) | `sudo rabbitmqctl list_queues name type leader members online` |
| List alarms | `sudo rabbitmqctl list_alarms` |
| Health check | `sudo rabbitmq-diagnostics check_running` |
| Quorum critical check | `sudo rabbitmq-diagnostics check_if_node_is_quorum_critical` |
| Transfer leadership | `sudo rabbitmqctl eval 'rabbit_quorum_queue:transfer_leadership(...).'` |
| Add member | `sudo rabbitmqctl add_member <queue> <node>` |
| Delete member | `sudo rabbitmqctl delete_member <queue> <node>` |
| Force boot | `sudo rabbitmqctl force_boot` |
| Forget node | `sudo rabbitmqctl forget_cluster_node rabbit@node` |

---

*Document Version: 2.0 | Last Updated: January 2026*
