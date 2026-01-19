# RabbitMQ Cluster Failover

## Scenarios, Test Cases, and Procedures

### Three-Node Cluster Configuration with Quorum Queues

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Test Environment Setup](#2-test-environment-setup)
3. [Prerequisites and Assumptions](#3-prerequisites-and-assumptions)
4. [Cluster Architecture Overview](#4-cluster-architecture-overview)
5. [Failover Scenarios Overview](#5-failover-scenarios-overview)
6. [Test Case 1: Leader Node Failure](#6-test-case-1-leader-node-failure)
7. [Test Case 2: Follower Node Failure](#7-test-case-2-follower-node-failure)
8. [Test Case 3: Network Partition (Split-Brain)](#8-test-case-3-network-partition-split-brain)
9. [Test Case 4: Graceful Node Shutdown](#9-test-case-4-graceful-node-shutdown)
10. [Test Case 5: Multiple Node Failure](#10-test-case-5-multiple-node-failure)
11. [Test Case 6: Disk Space Exhaustion](#11-test-case-6-disk-space-exhaustion)
12. [Test Case 7: Memory Pressure](#12-test-case-7-memory-pressure)
13. [Test Case 8: Rolling Restart](#13-test-case-8-rolling-restart)
14. [Test Case 9: Quorum Queue Replication Validation](#14-test-case-9-quorum-queue-replication-validation)
15. [Test Case 10: Client Connection Failover](#15-test-case-10-client-connection-failover)
16. [Monitoring and Validation Commands](#16-monitoring-and-validation-commands)
17. [Common Issues and Troubleshooting](#17-common-issues-and-troubleshooting)
18. [Recovery Procedures](#18-recovery-procedures)
19. [Best Practices for Production](#19-best-practices-for-production)
20. [Test Results Template](#20-test-results-template)
21. [Appendix: Sample Test Scripts](#21-appendix-sample-test-scripts)

---

## 1. Introduction

This document provides comprehensive failover scenarios and test cases for a three-node RabbitMQ 4.x cluster deployment using Quorum Queues. It includes detailed step-by-step procedures to validate high availability, automatic failover, Raft-based replication, and disaster recovery capabilities.

**Purpose of Failover Testing:**

- Validate automatic failover mechanisms work as expected
- Ensure zero or minimal message loss during node failures
- Test cluster behavior under various failure conditions
- Verify client reconnection and message delivery continuity
- Identify potential weaknesses in the cluster configuration
- Document recovery procedures for production incidents
- Build confidence in the high availability setup

---

## 2. Test Environment Setup

### Required Infrastructure

| Component | Specification | Purpose |
|-----------|---------------|---------|
| 3 RHEL 8 Servers | 4 CPU, 8GB RAM each | RabbitMQ cluster nodes |
| Test Client Machine | 2 CPU, 4GB RAM | Producer/consumer testing |
| Network Access | All nodes can communicate | Cluster formation |
| Monitoring Tools | RabbitMQ Management UI | Cluster status monitoring |

### Cluster Node Details

| Hostname | IP Address | Role | Node Name |
|----------|------------|------|-----------|
| rabbitmq-node1 | 192.168.1.101 | Leader/Follower | rabbit@rabbitmq-node1 |
| rabbitmq-node2 | 192.168.1.102 | Leader/Follower | rabbit@rabbitmq-node2 |
| rabbitmq-node3 | 192.168.1.103 | Leader/Follower | rabbit@rabbitmq-node3 |

### Software Versions

```
RabbitMQ Version: 4.1.x
Erlang Version: 26.x
Operating System: RHEL 8.x
Queue Type: Quorum Queues (Raft-based)
```

---

## 3. Prerequisites and Assumptions

### Prerequisites

- RabbitMQ cluster is fully operational with 3 nodes
- All nodes are properly clustered and synchronized
- Quorum queues are configured for high availability
- RabbitMQ Management plugin is enabled on all nodes
- Administrative access to all cluster nodes
- Monitoring tools are configured and accessible
- Test queues and exchanges are created
- Test client applications are ready

### Initial Cluster Verification

```bash
# Run on any node to verify cluster status
sudo rabbitmqctl cluster_status

# Expected output:
# - All 3 nodes listed under "Running Nodes"
# - All 3 nodes listed under "Disk Nodes"

# Verify quorum queue members
sudo rabbitmqctl list_queues name type leader members online

# Check node health
sudo rabbitmq-diagnostics check_running
sudo rabbitmq-diagnostics check_local_alarms
```

---

## 4. Cluster Architecture Overview

### Three-Node Cluster with Quorum Queues

In RabbitMQ 4.x, Quorum Queues replace classic mirrored queues. They use the Raft consensus algorithm for replication, providing stronger consistency guarantees.

### Key Concepts

| Term | Description |
|------|-------------|
| **Queue Leader** | The node that handles all read/write operations for the queue |
| **Queue Follower** | Replicas that maintain synchronized copies via Raft |
| **Quorum** | Majority of nodes required for operations (2 out of 3) |
| **Raft Consensus** | Algorithm ensuring strong consistency across replicas |
| **Failover** | Automatic election of a new leader when current leader fails |
| **Partition Handling** | Strategy for dealing with network splits (autoheal/pause_minority) |

### Quorum Queue Declaration

```bash
# Declare quorum queue using rabbitmqadmin
sudo rabbitmqadmin declare queue name=my-queue durable=true \
    arguments='{"x-queue-type":"quorum"}'

# Verify quorum queue created
sudo rabbitmqctl list_queues name type leader members online

# Expected output:
# my-queue   quorum   rabbit@rabbitmq-node1   [rabbit@rabbitmq-node1, rabbit@rabbitmq-node2, rabbit@rabbitmq-node3]
```

### Quorum Requirements

| Cluster Size | Quorum | Tolerated Failures |
|--------------|--------|-------------------|
| 3 nodes | 2 | 1 node |
| 5 nodes | 3 | 2 nodes |
| 7 nodes | 4 | 3 nodes |

---

## 5. Failover Scenarios Overview

### Test Scenarios Summary

| Scenario | Type | Impact Level | Expected Recovery Time |
|----------|------|--------------|------------------------|
| Leader Node Failure | Hard Failure | Medium | < 30 seconds |
| Follower Node Failure | Hard Failure | Low | Immediate (no impact) |
| Network Partition | Split-Brain | High | 1-2 minutes |
| Graceful Shutdown | Planned | Low | Immediate (no impact) |
| Multiple Node Failure | Catastrophic | Critical | Manual intervention |
| Disk Space Exhaustion | Resource | High | After disk cleanup |
| Memory Pressure | Resource | Medium | < 1 minute |
| Rolling Restart | Maintenance | None | N/A |
| Replication Validation | Validation | None | N/A |
| Client Failover | Application | Low | < 10 seconds |

---

## 6. Test Case 1: Leader Node Failure

### Objective

Simulate a catastrophic failure of the node hosting the queue leader and verify automatic failover to a follower node with minimal message loss and downtime.

**Severity:** HIGH

**Expected Outcome:** Automatic failover within 30 seconds

### Pre-Test Setup

```bash
# 1. Create test quorum queue
sudo rabbitmqadmin declare queue name=test-failover-queue durable=true \
    arguments='{"x-queue-type":"quorum"}'

# 2. Verify queue leader location
sudo rabbitmqctl list_queues name type leader members online

# 3. Publish test messages
for i in {1..1000}; do
    sudo rabbitmqadmin publish exchange=amq.default \
        routing_key=test-failover-queue payload="message-$i"
done

# 4. Verify message count
sudo rabbitmqctl list_queues name messages
```

### Test Steps

#### Step 1: Record Initial State

```bash
sudo rabbitmqctl cluster_status
sudo rabbitmqctl list_queues name type leader members online messages
# Note the queue leader node
```

#### Step 2: Identify Leader Node

```bash
sudo rabbitmqctl list_queues name leader | grep test-failover-queue
# Example output: test-failover-queue    rabbit@rabbitmq-node1
```

#### Step 3: Simulate Hard Failure (Stop Leader Node)

```bash
# On the leader node:
sudo rabbitmqctl stop_app

# OR force kill Erlang process for harder failure simulation:
sudo killall -9 beam.smp
```

#### Step 4: Monitor Failover (On Another Node)

```bash
watch -n 1 "sudo rabbitmqctl list_queues name leader online"
# Observe leader change
# Watch for new leader election
```

#### Step 5: Verify Queue Availability

```bash
sudo rabbitmqctl list_queues name type leader messages
# Queue should show new leader on surviving node
```

#### Step 6: Test Write Operations

```bash
# Publish new message to verify writes work
sudo rabbitmqadmin publish exchange=amq.default \
    routing_key=test-failover-queue payload="after-failover"

# Verify message count increased
sudo rabbitmqctl list_queues name messages
```

#### Step 7: Check Client Connections

```bash
sudo rabbitmqctl list_connections name state peer_host
# Clients should reconnect to available nodes
```

#### Step 8: Check Management UI

```
http://192.168.1.102:15672
# Verify cluster shows 2 nodes running
# Check queue is accessible and processing messages
```

### Expected Results

- Leader node is marked as down in cluster status
- One of the follower nodes is elected as new leader automatically
- Queue remains available and continues processing messages
- Clients reconnect to available nodes within 10-15 seconds
- Message loss is minimal or zero (only uncommitted messages)
- No manual intervention required
- Cluster operates with 2 nodes until failed node is restored

### Post-Test Recovery

```bash
# On the failed node:
sudo rabbitmqctl start_app

# Verify node rejoins cluster
sudo rabbitmqctl cluster_status

# Check queue members
sudo rabbitmqctl list_queues name members online
# All 3 members should be online
```

> **WARNING:** Uncommitted messages that were not replicated may be lost during hard failure. Use publisher confirms for critical messages.

---

## 7. Test Case 2: Follower Node Failure

### Objective

Verify that failure of a follower node has minimal impact on cluster operations and message processing continues normally.

**Severity:** LOW

**Expected Outcome:** No impact on service availability

### Test Steps

#### Step 1: Identify Follower Node

```bash
sudo rabbitmqctl list_queues name leader members
# Choose a node from members that is NOT the leader
```

#### Step 2: Stop Follower Node

```bash
# On the follower node:
sudo rabbitmqctl stop_app
```

#### Step 3: Verify Cluster Status

```bash
# On leader or remaining follower:
sudo rabbitmqctl cluster_status
# Follower node should be marked as down
```

#### Step 4: Verify Queue Operations

```bash
sudo rabbitmqctl list_queues name leader online messages
# Queue should continue normal operations with 2 members
# Quorum (2 of 3) is maintained
```

#### Step 5: Test Message Flow

```bash
# Publish messages
sudo rabbitmqadmin publish exchange=amq.default \
    routing_key=test-failover-queue payload="test-message"

# Consume messages
sudo rabbitmqadmin get queue=test-failover-queue ackmode=ack_requeue_false
```

#### Step 6: Restart Failed Node

```bash
# On the failed follower node:
sudo rabbitmqctl start_app
```

#### Step 7: Verify Resynchronization

```bash
sudo rabbitmqctl list_queues name members online
# All 3 members should be online and synchronized
```

### Expected Results

- No impact on message processing
- Queue leader continues operating normally
- Clients connected to other nodes are unaffected
- Clients connected to failed follower reconnect to other nodes
- No message loss occurs
- Failed node rejoins cluster cleanly after restart
- Queue resynchronizes automatically with the rejoined node

> **NOTE:** Follower failure is the least impactful failure scenario. The cluster can lose up to n-1 followers (where n is total nodes) and still maintain full functionality as long as quorum is maintained.

---

## 8. Test Case 3: Network Partition (Split-Brain)

### Objective

Test cluster behavior when network connectivity is lost between nodes, creating a partition scenario. Verify partition handling and quorum-based availability.

**Severity:** CRITICAL

**Expected Outcome:** Partition detected, minority side becomes unavailable

> **WARNING:** Network partitions can lead to split-brain scenarios. Quorum queues prevent data inconsistency by only allowing operations on the majority partition.

### Prerequisites

```bash
# Verify partition handling mode
sudo rabbitmqctl eval "application:get_env(rabbit, cluster_partition_handling)."

# Recommended: autoheal or pause_minority
# In /etc/rabbitmq/rabbitmq.conf:
# cluster_partition_handling = autoheal
```

### Test Steps

#### Step 1: Record Pre-Partition State

```bash
sudo rabbitmqctl cluster_status
sudo rabbitmqctl list_queues name leader members online messages
# All 3 nodes should be running
```

#### Step 2: Create Network Partition

```bash
# Method: Using iptables
# On node1, block traffic from node2 and node3:
sudo iptables -A INPUT -s 192.168.1.102 -j DROP
sudo iptables -A INPUT -s 192.168.1.103 -j DROP
sudo iptables -A OUTPUT -d 192.168.1.102 -j DROP
sudo iptables -A OUTPUT -d 192.168.1.103 -j DROP

# This creates partition: [node1] | [node2, node3]
```

#### Step 3: Detect Partition

```bash
# On node1 (minority - isolated):
sudo rabbitmqctl cluster_status
# Should show node2 and node3 as partitioned

# On node2 or node3 (majority):
sudo rabbitmqctl cluster_status
# Should show node1 as partitioned
```

#### Step 4: Observe Quorum Behavior

```bash
# On minority side (node1):
sudo rabbitmqctl list_queues name leader online
# Queue becomes unavailable - no quorum (1 of 3)

# On majority side (node2 or node3):
sudo rabbitmqctl list_queues name leader online
# Queue remains available - quorum maintained (2 of 3)
# New leader elected if needed
```

#### Step 5: Test Writes on Both Sides

```bash
# On minority side (node1) - should FAIL:
sudo rabbitmqadmin publish exchange=amq.default \
    routing_key=test-failover-queue payload="minority-write"
# Expected: Error - no quorum

# On majority side - should SUCCEED:
sudo rabbitmqadmin publish exchange=amq.default \
    routing_key=test-failover-queue payload="majority-write"
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
# Wait 30-60 seconds for healing
sudo rabbitmqctl cluster_status
# All nodes should show as running again

sudo rabbitmqctl list_queues name leader members online
# All members should be online
```

#### Step 8: Verify Message Integrity

```bash
sudo rabbitmqctl list_queues name messages
# Verify messages written to majority side are preserved
# Messages attempted on minority side were rejected (no loss)
```

### Expected Results

- Partition is detected within seconds on all nodes
- Minority partition loses quorum and becomes unavailable
- Majority partition continues operating normally
- Writes only succeed on majority side
- After healing, all nodes are running and synchronized
- No data corruption occurs
- No message loss (minority writes are rejected, not lost)

> **WARNING:** Always use odd number of nodes (3, 5, 7) for proper quorum. Never use "ignore" mode as it can cause data inconsistency.

---

## 9. Test Case 4: Graceful Node Shutdown

### Objective

Verify that planned maintenance shutdown of a node allows proper leader transfer and connection draining without message loss.

**Severity:** LOW

**Expected Outcome:** Zero message loss, graceful connection migration

### Test Steps

#### Step 1: Prepare for Shutdown

```bash
# Verify cluster health before shutdown
sudo rabbitmqctl cluster_status
sudo rabbitmqctl list_queues name leader members online messages
```

#### Step 2: Check if Node is Leader

```bash
sudo rabbitmqctl list_queues name leader
# If shutting down a leader node, transfer leadership first
```

#### Step 3: Transfer Leadership (If Needed)

```bash
# Transfer leadership before shutdown (recommended)
sudo rabbitmqctl eval 'rabbit_quorum_queue:transfer_leadership(<<"test-failover-queue">>, <<"rabbit@rabbitmq-node2">>).'

# Verify transfer
sudo rabbitmqctl list_queues name leader
```

#### Step 4: Gracefully Stop RabbitMQ

```bash
# Proper shutdown sequence
sudo rabbitmqctl stop_app
```

#### Step 5: Verify Cluster Adapts

```bash
# On remaining nodes:
sudo rabbitmqctl cluster_status
# Node should be marked as down gracefully

sudo rabbitmqctl list_queues name leader online
# Queue should have 2 online members, quorum maintained
```

#### Step 6: Verify Queue Availability

```bash
sudo rabbitmqctl list_queues name messages
# All queues should be available on remaining nodes

# Test publish
sudo rabbitmqadmin publish exchange=amq.default \
    routing_key=test-failover-queue payload="after-shutdown"
```

#### Step 7: Restart Node

```bash
sudo rabbitmqctl start_app
# Node should rejoin cluster cleanly
```

#### Step 8: Verify Full Recovery

```bash
sudo rabbitmqctl cluster_status
sudo rabbitmqctl list_queues name members online
# All 3 members should be online
```

### Expected Results

- No message loss occurs
- Connections drain gracefully or reconnect to other nodes
- Leadership transfers smoothly if specified
- Cluster continues operating normally with remaining nodes
- Node rejoins cluster cleanly after restart
- No alarms or errors are triggered

> **NOTE:** Graceful shutdown with leadership transfer is the preferred method for maintenance. Always use this approach for planned maintenance activities.

---

## 10. Test Case 5: Multiple Node Failure

### Objective

Test cluster behavior when multiple nodes fail simultaneously or in quick succession. This is a catastrophic scenario requiring manual intervention.

**Severity:** CRITICAL

**Expected Outcome:** Cluster unavailable, manual recovery required

> **DANGER:** This test will make the cluster completely unavailable. Only perform in test/staging environment. DO NOT run in production.

### Scenario A: Two Nodes Fail (Quorum Lost)

```bash
# Stop two nodes simultaneously
# On node2:
sudo rabbitmqctl stop_app

# On node3:
sudo rabbitmqctl stop_app

# On node1 (remaining node):
sudo rabbitmqctl cluster_status
# Cluster loses quorum

sudo rabbitmqctl list_queues name online
# Queue shows only 1 online member - no quorum
```

### Test Write Operation (Should Fail)

```bash
sudo rabbitmqadmin publish exchange=amq.default \
    routing_key=test-failover-queue payload="no-quorum-write"
# Expected: Error - queue unavailable, no quorum
```

### Recovery Procedure

```bash
# Start nodes in sequence:
# 1. Start one node to restore quorum
# On node2:
sudo rabbitmqctl start_app

# 2. Verify quorum restored
sudo rabbitmqctl list_queues name online
# Should show 2 online members

# 3. Start remaining node
# On node3:
sudo rabbitmqctl start_app

# 4. Verify all nodes are running
sudo rabbitmqctl cluster_status
sudo rabbitmqctl list_queues name members online
```

### Scenario B: All Nodes Fail

```bash
# If all nodes fail, perform these steps:

# 1. Start nodes - they should auto-recover
sudo rabbitmqctl start_app  # on each node

# 2. If node fails to start, force boot:
sudo rabbitmqctl force_boot
sudo rabbitmqctl start_app

# 3. Verify cluster status and queue integrity
sudo rabbitmqctl cluster_status
sudo rabbitmqctl list_queues name messages
```

> **WARNING:** Using force_boot may result in data loss. Only use as last resort after consulting documentation and ensuring you understand the implications.

---

## 11. Test Case 6: Disk Space Exhaustion

### Objective

Verify RabbitMQ's behavior when disk space is exhausted and test the alarm system and recovery procedures.

**Severity:** HIGH

**Expected Outcome:** Disk alarm triggers, node blocks publishers

### Test Steps

#### Step 1: Check Current Disk Limits

```bash
sudo rabbitmqctl eval "rabbit_disk_monitor:get_disk_free_limit()."
# Or check in rabbitmq.conf:
# disk_free_limit.absolute = 50GB
```

#### Step 2: Simulate Disk Exhaustion

```bash
# Create large file to fill disk space
sudo fallocate -l 40G /var/lib/rabbitmq/large-file
# Adjust size to trigger disk alarm based on your limit
```

#### Step 3: Wait for Disk Alarm

```bash
# Monitor alarms:
watch -n 2 "sudo rabbitmqctl list_alarms"
# Should show disk alarm: {resource_limit,disk,<node>}

sudo rabbitmq-diagnostics check_local_alarms
```

#### Step 4: Verify Publisher Blocking

```bash
# Try to publish messages
sudo rabbitmqadmin publish exchange=amq.default \
    routing_key=test-failover-queue payload="blocked"
# Publishers should be blocked
```

#### Step 5: Check Management UI

```
# Web UI should show red warning banner
# Node status shows disk alarm active
```

#### Step 6: Free Up Disk Space

```bash
sudo rm /var/lib/rabbitmq/large-file
```

#### Step 7: Verify Alarm Clears

```bash
sudo rabbitmqctl list_alarms
# Should return empty or no disk alarms
```

#### Step 8: Resume Publishing

```bash
# Publishers should automatically resume
sudo rabbitmqadmin publish exchange=amq.default \
    routing_key=test-failover-queue payload="after-alarm"
# Should succeed
```

### Expected Results

- Disk alarm triggers when free space drops below threshold
- Publishers are blocked from sending messages
- Consumers continue to drain queues
- Alarm clears automatically when disk space is freed
- Publishers resume automatically after alarm clears
- No message loss occurs (messages are rejected, not lost)

---

## 12. Test Case 7: Memory Pressure

### Objective

Test RabbitMQ behavior under high memory usage and verify memory alarm and flow control mechanisms.

**Severity:** HIGH

**Expected Outcome:** Memory alarm triggers, publishers throttled

### Test Steps

#### Step 1: Check Memory Configuration

```bash
sudo rabbitmqctl eval "vm_memory_monitor:get_memory_limit()."
# Or check rabbitmq.conf:
# vm_memory_high_watermark.relative = 0.4
```

#### Step 2: Simulate High Memory Usage

```bash
# Temporarily lower the memory threshold to trigger alarm
sudo rabbitmqctl eval "vm_memory_monitor:set_vm_memory_high_watermark(0.1)."
```

#### Step 3: Detect Memory Alarm

```bash
sudo rabbitmqctl list_alarms
# Should show: {resource_limit,memory,<node>}

sudo rabbitmq-diagnostics check_local_alarms
```

#### Step 4: Verify Publisher Throttling

```bash
# Publishers should be blocked or slowed
sudo rabbitmqadmin publish exchange=amq.default \
    routing_key=test-failover-queue payload="memory-test"
# May be blocked
```

#### Step 5: Clear Memory Alarm

```bash
# Restore normal threshold
sudo rabbitmqctl eval "vm_memory_monitor:set_vm_memory_high_watermark(0.4)."
```

#### Step 6: Verify Alarm Clears

```bash
sudo rabbitmqctl list_alarms
# Memory alarm should clear
```

### Expected Results

- Memory alarm triggers at configured watermark
- Publishers are throttled or blocked
- Consumers continue processing messages
- Alarm clears when memory drops below watermark
- Publishers resume normal operation
- Node remains stable and responsive

> **NOTE:** Memory alarms are a sign of insufficient resources or unbalanced producer/consumer rates. Consider adding more consumers or increasing memory allocation.

---

## 13. Test Case 8: Rolling Restart

### Objective

Perform a rolling restart of all cluster nodes with zero downtime for upgrades or configuration changes.

**Severity:** LOW

**Expected Outcome:** Zero downtime, continuous message processing

### Test Steps

#### Step 1: Verify Initial Cluster State

```bash
sudo rabbitmqctl cluster_status
sudo rabbitmqctl list_queues name leader members online messages
```

#### Step 2: Identify Queue Leaders

```bash
sudo rabbitmqctl list_queues name leader
# Note which node is leader for each queue
```

#### Step 3: Restart Follower Node 1

```bash
# Choose a follower node first
# On that node:
sudo rabbitmqctl stop_app
sudo rabbitmqctl start_app

# Verify node rejoined
sudo rabbitmqctl list_queues name members online
```

#### Step 4: Restart Follower Node 2

```bash
# On second follower:
sudo rabbitmqctl stop_app
sudo rabbitmqctl start_app

# Verify node rejoined
sudo rabbitmqctl list_queues name members online
```

#### Step 5: Transfer Leadership from Leader Node

```bash
# Before restarting leader, transfer leadership
sudo rabbitmqctl eval 'rabbit_quorum_queue:transfer_leadership(<<"test-failover-queue">>, <<"rabbit@rabbitmq-node2">>).'

# Verify transfer
sudo rabbitmqctl list_queues name leader
```

#### Step 6: Restart Original Leader Node

```bash
# On the original leader (now follower):
sudo rabbitmqctl stop_app
sudo rabbitmqctl start_app
```

#### Step 7: Verify Complete Cluster

```bash
sudo rabbitmqctl cluster_status
sudo rabbitmqctl list_queues name leader members online
# All nodes should be running, all members online
```

#### Step 8: Verify Message Flow

```bash
# Check message counts
sudo rabbitmqctl list_queues name messages

# Test publish/consume
sudo rabbitmqadmin publish exchange=amq.default \
    routing_key=test-failover-queue payload="rolling-restart-test"
```

### Best Practices for Rolling Restart

- Always restart follower nodes before leader nodes
- Transfer leadership before restarting leader
- Wait for each node to fully rejoin before restarting the next
- Verify queue members between restarts
- Monitor client connections and reconnection patterns
- Restart one node at a time (never multiple nodes simultaneously)
- Schedule during low-traffic periods when possible

---

## 14. Test Case 9: Quorum Queue Replication Validation

### Objective

Verify that quorum queue replication is working correctly and queues are synchronized across all nodes.

**Severity:** MEDIUM

**Expected Outcome:** All queues properly replicated and synchronized

### Test Steps

#### Step 1: Create Test Quorum Queues

```bash
sudo rabbitmqadmin declare queue name=replication-test-1 durable=true \
    arguments='{"x-queue-type":"quorum"}'
sudo rabbitmqadmin declare queue name=replication-test-2 durable=true \
    arguments='{"x-queue-type":"quorum"}'
sudo rabbitmqadmin declare queue name=replication-test-3 durable=true \
    arguments='{"x-queue-type":"quorum"}'
```

#### Step 2: Verify Queue Distribution

```bash
sudo rabbitmqctl list_queues name type leader members online
# Each queue should show 3 members (all nodes)
```

#### Step 3: Publish Messages to Test Queues

```bash
for i in {1..1000}; do
    sudo rabbitmqadmin publish exchange=amq.default \
        routing_key=replication-test-1 payload="test $i"
done
```

#### Step 4: Verify Message Counts

```bash
sudo rabbitmqctl list_queues name messages
# All queues should report correct counts
```

#### Step 5: Stop Leader and Verify Messages

```bash
# Identify leader
sudo rabbitmqctl list_queues name leader | grep replication-test-1

# Stop the leader
sudo rabbitmqctl stop_app  # on leader node

# Verify messages on new leader
sudo rabbitmqctl list_queues name messages
# Count should match
```

#### Step 6: Restart Node and Verify Sync

```bash
sudo rabbitmqctl start_app

# Verify all members online
sudo rabbitmqctl list_queues name members online
```

#### Step 7: Consume and Verify Message Integrity

```bash
# Consume all messages
for i in {1..1000}; do
    sudo rabbitmqadmin get queue=replication-test-1 ackmode=ack_requeue_false
done

# Verify queue empty
sudo rabbitmqctl list_queues name messages | grep replication-test-1
```

### Expected Results

- All queues have 3 members (replicated to all nodes)
- Leaders distributed across nodes
- Message counts match on all nodes
- Messages preserved after leader failure
- No duplicate or lost messages

---

## 15. Test Case 10: Client Connection Failover

### Objective

Verify that client applications can reconnect automatically when their connected node fails and continue processing messages.

**Severity:** MEDIUM

**Expected Outcome:** Clients reconnect within 10 seconds

### Prerequisites

Client application must be configured with multiple RabbitMQ node addresses for automatic failover.

```python
# Example client configuration (Python):
connection_params = [
    pika.ConnectionParameters(host="192.168.1.101"),
    pika.ConnectionParameters(host="192.168.1.102"),
    pika.ConnectionParameters(host="192.168.1.103")
]
```

### Test Steps

#### Step 1: Start Test Client

```bash
# Run producer and consumer with multi-host config
python3 producer.py --hosts 192.168.1.101,192.168.1.102,192.168.1.103
```

#### Step 2: Verify Initial Connection

```bash
# Check which node client is connected to
sudo rabbitmqctl list_connections name peer_host peer_port state node
```

#### Step 3: Stop Connected Node

```bash
# Identify which node client is using
# Stop that specific node
sudo rabbitmqctl stop_app
```

#### Step 4: Monitor Client Reconnection

```bash
# Watch client logs for reconnection attempts
# Should see connection errors followed by successful reconnect

# On remaining nodes:
watch -n 1 "sudo rabbitmqctl list_connections name peer_host state"
# Client should appear connected to different node
```

#### Step 5: Verify Message Flow Continues

```bash
sudo rabbitmqctl list_queues name messages consumers
# Message count should continue increasing
```

#### Step 6: Test Multiple Failovers

```bash
# Stop second node
sudo rabbitmqctl stop_app  # on another node

# Verify client connects to third node
sudo rabbitmqctl list_connections name peer_host
```

#### Step 7: Restore All Nodes

```bash
# Start all nodes
sudo rabbitmqctl start_app  # on each stopped node

# Verify cluster health
sudo rabbitmqctl cluster_status
```

### Expected Results

- Client detects connection failure within 1-2 seconds
- Client attempts reconnection to alternative nodes
- Successful reconnection within 5-10 seconds
- Message processing resumes automatically
- No messages are lost (with proper acknowledgments)
- Client can handle multiple consecutive failovers

> **NOTE:** Clients must implement retry logic and configure multiple broker addresses. Use client library features for automatic failover.

---

## 16. Monitoring and Validation Commands

### Essential Monitoring Commands

#### Cluster Status

```bash
sudo rabbitmqctl cluster_status
```

#### Node Health

```bash
sudo rabbitmq-diagnostics check_running
sudo rabbitmq-diagnostics check_local_alarms
sudo rabbitmq-diagnostics check_port_connectivity
sudo rabbitmq-diagnostics check_if_node_is_quorum_critical
```

#### Queue Information

```bash
sudo rabbitmqctl list_queues name type leader members online messages consumers memory
```

#### Connection Status

```bash
sudo rabbitmqctl list_connections name state user peer_host node
```

#### Channel Information

```bash
sudo rabbitmqctl list_channels connection number consumer_count
```

#### Memory Usage

```bash
sudo rabbitmqctl status | grep -A 10 memory
```

#### Disk Status

```bash
sudo rabbitmqctl status | grep -A 5 disk
df -h /var/lib/rabbitmq
```

#### Active Alarms

```bash
sudo rabbitmqctl list_alarms
```

### Management UI Monitoring

| Section | Information |
|---------|-------------|
| Overview | Cluster-wide message rates, connections, channels |
| Nodes | Individual node status, memory, disk, uptime |
| Queues | Queue details, message rates, consumers, memory |
| Connections | Active client connections and their channels |
| Exchanges | Exchange types, bindings, message rates |

---

## 17. Common Issues and Troubleshooting

| Issue | Symptoms | Resolution |
|-------|----------|------------|
| Node won't join cluster | Error: inconsistent_cluster | Reset node: `rabbitmqctl reset`, then rejoin |
| Queue has no leader | Queue unavailable | Check if quorum exists; start more nodes |
| Split-brain not healing | Partition persists | Check partition_handling mode, manually restart nodes |
| Memory alarm stuck | Alarm persists after cleanup | Restart node or adjust vm_memory_high_watermark |
| Slow failover | Takes > 60 seconds | Check network latency, reduce net_ticktime |
| Message loss on failover | Messages disappear | Implement publisher confirms and consumer acks |
| Connections not rebalancing | All clients on one node | Restart clients or use load balancer |
| Quorum queue won't elect leader | No leader shown | Ensure majority of members are online |

### Diagnostic Steps

1. Check RabbitMQ logs: `tail -f /var/log/rabbitmq/rabbit@hostname.log`
2. Check system logs: `journalctl -u rabbitmq-server -f`
3. Verify network connectivity: `ping` and `telnet` to ports 5672, 25672, 4369
4. Check Erlang cookie: `cat /var/lib/rabbitmq/.erlang.cookie`
5. Verify hostname resolution: `cat /etc/hosts`
6. Check resource usage: `top`, `df -h`, `free -h`
7. Review cluster events in Management UI

---

## 18. Recovery Procedures

### Scenario: Node Won't Start After Crash

```bash
# 1. Check logs for errors
sudo journalctl -u rabbitmq-server -n 100

# 2. Verify Erlang cookie permissions
ls -la /var/lib/rabbitmq/.erlang.cookie
sudo chmod 400 /var/lib/rabbitmq/.erlang.cookie
sudo chown rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie

# 3. Try starting
sudo rabbitmqctl start_app

# 4. If cluster start fails, force boot:
sudo rabbitmqctl force_boot
sudo rabbitmqctl start_app

# 5. If still fails, reset and rejoin cluster
sudo rabbitmqctl stop_app
sudo rabbitmqctl reset
sudo rabbitmqctl join_cluster rabbit@rabbitmq-node1
sudo rabbitmqctl start_app
```

### Scenario: Persistent Network Partition

```bash
# 1. Identify which nodes are in each partition
sudo rabbitmqctl cluster_status

# 2. Choose the partition with majority/most recent data

# 3. Restart nodes in the minority partition
# On minority nodes:
sudo rabbitmqctl stop_app
sudo rabbitmqctl start_app

# 4. If partition persists, force healing:
# On all nodes in minority partition:
sudo rabbitmqctl stop_app
sudo rabbitmqctl forget_cluster_node rabbit@minority-node
sudo rabbitmqctl reset
sudo rabbitmqctl join_cluster rabbit@majority-node
sudo rabbitmqctl start_app
```

### Scenario: Quorum Queue Member Issues

```bash
# 1. Check queue members
sudo rabbitmqctl list_queues name members online

# 2. If a member is permanently gone, delete it
sudo rabbitmqctl delete_member <queue_name> rabbit@failed-node

# 3. Add new member if needed
sudo rabbitmqctl add_member <queue_name> rabbit@new-node

# 4. Verify members
sudo rabbitmqctl list_queues name members online
```

### Scenario: All Nodes Down (Complete Cluster Failure)

```bash
# 1. Start nodes - they should auto-recover
sudo rabbitmqctl start_app  # on each node

# 2. If waiting for Mnesia tables, force boot on one node:
sudo rabbitmqctl force_boot
sudo rabbitmqctl start_app

# 3. Wait for first node to fully start
sudo rabbitmqctl status

# 4. Start other nodes one by one
sudo rabbitmqctl start_app

# 5. Verify cluster reformation
sudo rabbitmqctl cluster_status

# 6. Check queue integrity
sudo rabbitmqctl list_queues name messages
```

---

## 19. Best Practices for Production

### Cluster Configuration

- Use odd number of nodes (3, 5, 7) for proper quorum
- Set `cluster_partition_handling` to `autoheal` or `pause_minority`
- Use quorum queues for all critical queues
- Configure appropriate quorum queue delivery limits
- Use durable queues for persistent messaging

### Monitoring

- Implement comprehensive monitoring (Prometheus + Grafana)
- Set up alerts for disk/memory alarms
- Monitor queue depth and consumer lag
- Track connection and channel counts
- Log cluster events and partition occurrences
- Monitor message rates and throughput

### Client Applications

- Configure clients with all cluster node addresses
- Implement automatic reconnection logic
- Use publisher confirms for critical messages
- Always acknowledge messages after processing
- Set appropriate prefetch counts
- Handle connection failures gracefully

### Resource Management

- Set appropriate memory and disk limits
- Configure `vm_memory_high_watermark` (40% default)
- Set `disk_free_limit` to adequate value
- Monitor and rotate logs regularly
- Plan for 2x peak load capacity
- Use SSD for persistence files

### Disaster Recovery

- Regular backup of queue definitions
- Document recovery procedures
- Test failover scenarios quarterly
- Maintain separate staging environment for testing
- Keep configuration in version control
- Document node startup order

### Maintenance

- Use rolling restarts for updates
- Transfer leadership before leader maintenance
- Schedule maintenance during low-traffic periods
- Test configuration changes in staging first
- Keep RabbitMQ and Erlang versions up to date
- Plan upgrade paths carefully

---

## 20. Test Results Template

### Test Execution Record

| Field | Value |
|-------|-------|
| Test Date | |
| Tester Name | |
| Environment | Test / Staging / Production |
| RabbitMQ Version | |
| Cluster Configuration | 3 nodes |
| Queue Type | Quorum Queues |

### Test Case Results

| Test Case | Status | Duration | Message Loss | Notes |
|-----------|--------|----------|--------------|-------|
| TC1: Leader Node Failure | Pass/Fail | | | |
| TC2: Follower Node Failure | Pass/Fail | | | |
| TC3: Network Partition | Pass/Fail | | | |
| TC4: Graceful Shutdown | Pass/Fail | | | |
| TC5: Multiple Node Failure | Pass/Fail | | | |
| TC6: Disk Space Exhaustion | Pass/Fail | | | |
| TC7: Memory Pressure | Pass/Fail | | | |
| TC8: Rolling Restart | Pass/Fail | | | |
| TC9: Replication Validation | Pass/Fail | | | |
| TC10: Client Failover | Pass/Fail | | | |

### Success Criteria

- All automated failovers complete within expected time (< 30 seconds)
- Message loss is zero or within acceptable limits (< 0.1%)
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

**Result:** [ ] Pass  [ ] Fail  [ ] Pass with Conditions

**Recommendations:**

[List any recommendations for improvement]

---

## 21. Appendix: Sample Test Scripts

### Sample Producer Script (producer.py)

```python
#!/usr/bin/env python3
import pika
import time
import sys

def main():
    # Multi-host failover configuration
    hosts = ["192.168.1.101", "192.168.1.102", "192.168.1.103"]
    credentials = pika.PlainCredentials("admin", "password")

    params = [
        pika.ConnectionParameters(
            host=h, credentials=credentials,
            heartbeat=30, blocked_connection_timeout=300
        ) for h in hosts
    ]

    connection = pika.BlockingConnection(params)
    channel = connection.channel()

    # Declare quorum queue
    channel.queue_declare(
        queue="test-failover-queue",
        durable=True,
        arguments={"x-queue-type": "quorum"}
    )

    # Enable publisher confirms
    channel.confirm_delivery()

    message_count = 0
    try:
        while True:
            message = f"Message {message_count}"
            channel.basic_publish(
                exchange="",
                routing_key="test-failover-queue",
                body=message,
                properties=pika.BasicProperties(delivery_mode=2)
            )
            print(f"Sent: {message}")
            message_count += 1
            time.sleep(0.1)
    except KeyboardInterrupt:
        connection.close()

if __name__ == "__main__":
    main()
```

### Sample Consumer Script (consumer.py)

```python
#!/usr/bin/env python3
import pika
import time

def callback(ch, method, properties, body):
    print(f"Received: {body.decode()}")
    # Simulate processing
    time.sleep(0.05)
    # Acknowledge message
    ch.basic_ack(delivery_tag=method.delivery_tag)

def main():
    hosts = ["192.168.1.101", "192.168.1.102", "192.168.1.103"]
    credentials = pika.PlainCredentials("admin", "password")

    params = [
        pika.ConnectionParameters(
            host=h, credentials=credentials, heartbeat=30
        ) for h in hosts
    ]

    connection = pika.BlockingConnection(params)
    channel = connection.channel()

    # Set prefetch count
    channel.basic_qos(prefetch_count=10)

    channel.basic_consume(
        queue="test-failover-queue",
        on_message_callback=callback
    )

    print("Waiting for messages...")
    try:
        channel.start_consuming()
    except KeyboardInterrupt:
        channel.stop_consuming()
        connection.close()

if __name__ == "__main__":
    main()
```

---

## Document Information

| Field | Value |
|-------|-------|
| Document Version | 2.0 |
| Last Updated | January 2026 |
| Document Type | Test Procedures and Scenarios |
| Target Audience | DevOps Engineers, SREs, System Administrators |
| RabbitMQ Version | 4.1.x |
| Queue Type | Quorum Queues |
