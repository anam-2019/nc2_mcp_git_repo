---
name: nc2_cvm/cluster_expansion
description: Troubleshooting and workflow for adding nodes to an NC2 cluster (expansion). Use when expansion tasks fail or time out.
---

# nc2_cvm/cluster_expansion

This skill provides the workflow for troubleshooting NC2 cluster expansion failures, focusing on node discovery and imaging.

---

## Quick Start: High-Level Workflow

Cluster expansion in NC2 is orchestrated by the MCM (Multi-Cloud Manager) console but executed by agents running on the cluster. Troubleshooting typically involves verifying the workflow state in the NC2 Portal first, then investigating the Cluster Agent and Genesis logs on the Prism Element (PE) cluster.

### Key Log Locations

| Component | Purpose |
|-----------|---------|
| Cluster Agent | `/var/log/cluster-agent.log` — Tracks communication between MCM and the cluster |
| Genesis | `/home/nutanix/data/logs/genesis.out` — Tracks the actual expansion orchestration on the CVMs |
| Configuration | `/etc/nutanix/config/zeus_config_printer` — Verifies node status in Zookeeper |

---

## Step 1: Determine the Expansion State (CVM Level)

If the NC2 Portal shows the expansion is stuck or failed, you must identify the backend state from the Genesis Master CVM.

### Identify the Genesis Master

```bash
convert_cluster_status | grep master
```

Log into the returned IP address to perform subsequent checks.

### Check Expansion Status

Run the following command to see the current state of the expansion state machine:

```bash
expand_cluster_status
```

Alternatively, query Zookeeper directly for granular status:

```bash
zkcat /appliance/logical/genesis/expand_cluster_status
```

### Identify Failures in Logs

Search genesis.out for the kFailed state to find the exact timestamp and error message:

```bash
grep state_machine.py ~/data/logs/genesis.out | grep kFailed
```

---

## Step 2: Analyze Common Failure Scenarios

### Scenario A: Memory Configuration Mismatch

**Symptom:** Expansion fails after CVM memory was manually reconfigured (e.g., increased from 48GB to 64GB) on the base cluster.

**Error Log (genesis.out):** You may see logs indicating `Updating config {'cvm_mem_mb': ...}` followed by timeouts or failures to reach the new node.

**Root Cause:** The new node is provisioned with default memory settings, while existing nodes have custom configurations. The cluster attempts to update the new node's config during expansion but may fail if the node is not yet fully reachable or stable.

**Reference:** Known issues exist where expansion fails if the target node has lower memory than the base cluster (e.g., ENG-839925).

### Scenario B: Genesis Installation Failure

**Symptom:** Node add gets stuck in "joining" state.

**Error Log (genesis.out):**

```
ERROR expand_cluster.py:3721 10.x.x.x: Failed to install new genesis to 10.x.x.x
ERROR state_machine.py:358 Current state: software_change handler returned error: kFailed
```

**Troubleshooting:** This indicates the existing cluster failed to push/install the Genesis service onto the new bare-metal node. Check network connectivity between the Genesis Master and the new node IP.

### Scenario C: Hypervisor/AOS Version Mismatch

**Symptom:** Expansion is stuck; the new node is provisioned but cannot join.

**Error Log:**

```
ERROR kvm_upgrade_helper.py:283 AHV version '10.0.1.1' is different from the cluster intended hypervisor version '10.3'
ERROR expand_cluster.py:1383 Node cannot volunteer for cluster expansion.
```

**Root Cause:** If a previous AHV upgrade failed partially (e.g., one host failed to upgrade), the cluster might provision a new node with the old AHV version, while the expansion workflow expects the new version (or vice versa).

**Resolution:** Ensure the "Cluster Leader" (infra leader) is residing on a host with the correct/intended version so it instructs MCM to provision the correct image.

---

## Step 3: Verify Connectivity & Agents

If the node is provisioned in Azure but the Nutanix cluster cannot see it, check the Cluster Agent and Flow Gateway (FGW) communication.

### Check Cluster Agent Health

The cluster agent runs on a single host and uses "ping-pongs" to talk to MCM.

```bash
systemctl status cluster-agent-service | grep running
tail -f /var/log/cluster-agent.log
```

### Test Connectivity to Gateway

Ensure the cluster can reach the external Nutanix gateway APIs:

```bash
curl -v https://gateway-external-api.cloud.nutanix.com/echo42
```

### Check Azure Networking (FGW/VTEP)

If the expansion involves networking components or VTEPs, verify that the Flow Gateway (FGW) is healthy and reachable. A single Prism Central (PC) manages a single FGW deployment; ensure no conflicting FGW subnets were created manually.

---

## Step 4: Useful Commands Cheat Sheet

| Objective | Command / Action |
|-----------|------------------|
| Check Cluster Health | `cluster status` &#124; `egrep -v "UP&#124;^$"` |
| Find Master CVM | `convert_cluster_status` &#124; `grep master` |
| Watch Genesis Events | `tail -f ~/data/logs/genesis.out` |
| Check Node List in Config | `zeus_config_printer` &#124; `grep node_list -A5` |
| Check Cloud Agent Logs | `grep -i "error" /var/log/cluster-agent.log` |
| Remote Access (Azure) | Use Teleport for SSH access (Direct SSH is restricted) |
