---
name: nc2_cvm/cluster_deployment
description: Troubleshooting and workflow for NC2 cluster deployment failures. Use when new cluster creation fails on AWS or Azure.
---

# nc2_cvm/cluster_deployment

This skill provides a structured guide to known issues and troubleshooting steps for NC2 cluster deployment failures, categorized by cloud provider and component.

---

## Immediate Action Plan for Failed Deployments

When a deployment fails, follow these steps first:

1. **Validate Quotas** — Check Azure Subscription or AWS Service Quotas for the specific instance family.
2. **Test "Echo42"** — Run the connectivity test from a jump host or VM in the same subnet (see [Essential Connectivity Test](#essential-connectivity-test)).
3. **Check Cloud Events** — Review Azure Activity Log or AWS CloudTrail for "Access Denied" or "ResourceQuotaExceeded" events during the failure timestamp.
4. **Review cluster-agent.log** — This is the single best source of truth for why the Nutanix software considers the deployment failed.

---

## Common Deployment Failure Reasons

Across both AWS and Azure, these are the most frequent blockers:

| Category | Description |
|----------|-------------|
| **Insufficient Capacity** | The selected region/AZ does not have enough bare-metal hosts (e.g., i3.metal, AN36P) to satisfy the request or rack awareness requirements. |
| **Quota/Limits Exceeded** | Cloud account vCPU or specific service quotas (e.g., NAT Gateways, Elastic IPs) are hit. |
| **Permissions** | The service principal (Azure) or CloudFormation role (AWS) lacks required permissions (e.g., `iam:PassRole`, `RunInstances`). |
| **Network Reachability** | Nodes cannot reach the NC2 Orchestrator (MCM) or Cloud APIs (S3, EC2, Azure ARM) due to firewall/proxy restrictions. |

---

## NC2 on Azure: Known Issues & Troubleshooting

### 1. Flow Gateway (FGW) Deployment Failures

FGW deployment often fails or hangs in the "Provisioning" or "Configuring" state.

#### Known Issues

| Issue | Description |
|-------|-------------|
| **Quota Mismatch** | FGW requires specific VM sizes (e.g., D4_v4 or D32_v4). If subscription quota for this family is exhausted, deployment fails with `AZURE_ALLOCATION_FAILED`. |
| **DNS/Internet Connectivity** | If the FGW external subnet cannot resolve public DNS or reach MCM, the agent fails to download required RPMs. |
| **Trust Setup Failure** | If the Adonis service on Prism Central (PC) is crashing or NTP is out of sync between PC and FGW, the certificate exchange fails. |

#### Troubleshooting Steps

1. **Check Splunk/MCM Logs** — Look for errors like `AZURE_DENIED_BY_POLICY` or timeouts.

2. **Verify Connectivity** — Spin up a test Linux VM in the **FGW external subnet** and run:

```bash
curl -v https://gateway-external-api.cloud.nutanix.com/echo42
nslookup google.com
```

3. **Inspect Logs**

| Location | Purpose |
|----------|---------|
| FGW VM: `/var/log/fgwagent/fgwagent.log` | Agent errors |
| FGW VM: `/var/log/cloud-init.log` | Bootstrap issues |
| PC VM: `/home/nutanix/data/logs/atlas.out` | Network controller logs |

### 2. Prism Central (PC) Deployment Failure

**Symptom:** PC deployment fails with "Failed to enable microservices infrastructure".

**Root Cause:** PC cannot pull Docker images because the NAT Gateway is missing, misconfigured, or detached from the PC subnet.

**Fix:**

1. Verify a NAT Gateway is associated with the PC subnet in the Azure Portal.
2. **Toggle NAT GW** — Sometimes detaching and re-attaching the NAT Gateway to the subnet resolves backend association glitches.
3. **Check Outbound Rules** — Ensure traffic to `*.docker.io`, `*.production.cloudflare.docker.com`, and `*.nutanix.github.io` is allowed.

### 3. Specific Error Codes (Azure)

| Error Code | Symptom | Workaround |
|------------|---------|------------|
| **ENG-742935** | VM creation fails with `INVALID_ARGUMENT: Controller request timed out` | Run on PC: `atlas_cli network_controller.recover <nc_uuid> redeploy_service_list=policydb,hermes` |
| **ENG-776010** | Virtual switch vs0 inconsistent MTU state | Update VS MTU: `acli net.update_virtual_switch vs0 mtu=<value>` |

---

## NC2 on AWS: Known Issues & Troubleshooting

### 1. Cluster Creation Failures

| Issue | Fix |
|-------|-----|
| **Unsupported Instance Type** | The selected instance (e.g., z1d.metal) is not available in the chosen Availability Zone. Choose a different AZ or instance type. |
| **IAM Role Issues** | Error: `Not authorized to perform RunInstances` or `iam:PassRole`. Update the CloudFormation stack to ensure `Nutanix-Clusters-High-Nc2-Cluster-Role` has the latest permissions. |
| **VPC Endpoint Limits** | Deployment fails if the account has reached the limit for Interface VPC Endpoints (required for private S3/EC2 access). |

### 2. Node/Host Provisioning Failures

**Symptom:** Nodes fail to join the cluster within 40 minutes (booting timeout).

**Checks:**

1. **Subnet Access** — Does the subnet allow outbound access to `downloads.cloud.nutanix.com`?
2. **Log Analysis** — Check `/var/log/clusters-agents-upgrader.log` on the AHV host for download timeouts (exit status 35).
3. **API Reachability** — If using a private proxy or VPC endpoints, ensure nodes can resolve and reach `ec2.<region>.amazonaws.com`.

---

## General Troubleshooting Guide

### Log Locations & Commands

| Component | Log File / Command | Purpose |
|-----------|-------------------|---------|
| CVM | `~/data/logs/cluster-agent.log` | Primary log for cluster creation, PC deployment status, and MCM communication |
| CVM | `~/data/logs/infra-gateway.log` | Logs for infra-gateway service (proxy between cluster and MCM) |
| AHV Host | `/var/log/clusters-agents-upgrader.log` | Node provisioning, RPM download failures, boot timeouts |
| AHV Host | `/var/log/host-agent.log` | Host agent status communicating with MCM |
| PC (NC2) | `~/data/logs/atlas.out` | Flow Virtual Networking (FVN), FGW, subnet programming issues |

### Essential Connectivity Test

Must return 200 OK from CVM, PC, or FGW to confirm MCM reachability:

```bash
curl -v https://gateway-external-api.cloud.nutanix.com/echo42
```

### Verify Cluster Services

```bash
genesis status
```

Verify `cluster_agent`, `infra_gateway`, and `genesis` services are up.

---

## Agent Architecture: Host Agent, Cluster Agent & Infra Gateway

Understanding the three key agents and their interaction is essential for troubleshooting deployment failures.

### Summary: Which Agent to Check

| Scenario | Check This Agent |
|----------|------------------|
| Cluster operations fail (creation, expansion) | **Cluster Agent** |
| Node-specific tasks fail (booting, imaging) | **Host Agent** |
| Everything is disconnected in portal | **Infra Gateway** |

### 1. Host Agent (host-agent / ahv-host-agent)

**Location:** AHV Host (root)

**Role:**
- Sends "Ping" heartbeats to MCM to report node status (Provisioning, Booting, Installing, Running)
- Executes instructions from MCM (e.g., upgrading AHV, configuring networking)
- Responsible for node-level operations, reporting node health to MCM, and applying configurations (like OVS/OVN networking)

**Log Location:**
- `/var/log/host-agent.log` (Primary log)
- `/var/log/ahv-host-agent.log` (Legacy/Wrapper logs)

**Status Command:**

```bash
systemctl status ahv-host-agent
```

**Troubleshooting:**

| Issue | Action |
|-------|--------|
| **Check Connectivity** | Agent must reach MCM. If service is running but node shows "Disconnected" in portal, check `curl` connectivity to the gateway URL found in the logs. |
| **Installation Stalls** | If node is stuck in "Installing", check this log for failures applying specific configuration (usually networking or OVS bridge setup). |
| **Restart** | If agent is stuck, restart safely: `systemctl restart ahv-host-agent` |

### 2. Cluster Agent (cluster-agent)

**Location:** CVM (nutanix user)

**Role:**
- Manages cluster creation, expansion, and shrinkage
- Orchestrates Prism Central deployment during initial cluster build
- Reports cluster health and capacity to MCM
- Follows leader-follower model (one CVM is the leader)

**Log Location:**
- `~/data/logs/cluster-agent.log` — **Note:** Only the leader CVM generates active logs. Non-leaders will have empty or stale logs.

**Status Command:**

```bash
# Check status on all CVMs
allssh "systemctl status cluster-agent-service"

# Find the leader (Run on any CVM)
allssh "ps -aux | grep cluster-agent | grep -v grep"
# Look for the process running with the '--log' flag, or check the logs for "I am the leader".
```

**Troubleshooting:**

| Issue | Action |
|-------|--------|
| **PC Deployment Failed** | Detailed error (including Python tracebacks from deployment script) will be in the Cluster Agent leader's log. |
| **Service Crash** | Check for "Out of Memory" (OOM) errors or Genesis restarting it frequently. |
| **Leader Split** | Ensure only one CVM thinks it is the leader. Multiple leaders can cause race conditions in reporting to MCM. |

### 3. Infra Gateway (infra-gateway)

**Location:** CVM (nutanix user)

**Role:**
- Acts as the secure communication proxy (gateway) between the Cluster Agent and external MCM service
- Cluster Agent does **not** talk to MCM directly; it sends messages to Infra Gateway, which forwards them to the cloud
- Handles the secure tunnel (HTTPS/TLS) to `gateway-external-api.cloud.nutanix.com`
- Authenticates the cluster with MCM using certificates

**Log Location:**
- `~/data/logs/infra-gateway.log`

**Status Command:**

```bash
systemctl status infra-gateway-service
```

**Troubleshooting:**

| Issue | Action |
|-------|--------|
| **Connectivity Errors** | First place to check if dashboard says "Cluster Disconnected". Look for "connection timed out" or "handshake failed" in the log. |
| **Certificate Issues** | "401 Unauthorized" or certificate errors indicate cluster's client certificate may be expired or invalid. |
| **Verification** | Run Echo42 test from CVM to verify the path used by Infra Gateway is open: `curl -v https://gateway-external-api.cloud.nutanix.com/echo42` |

### Agent Interaction Flow

```
1. MCM sends a command (e.g., "Create Cluster")
2. Host Agent (AHV) reports "Node is Up"
3. Cluster Agent (CVM Leader) picks up the task to "Configure Zeus/Cluster"
4. Cluster Agent sends the status update to Infra Gateway (Local CVM)
5. Infra Gateway pushes the update securely over the internet to MCM
```

---

## Troubleshooting Flow Summary

```
Deployment Failed
│
├─► Immediate: Validate quotas, test Echo42, check cloud events
│
├─► Azure:
│   ├─► FGW stuck? → Check quota, DNS, trust setup
│   │   └─► Logs: fgwagent.log, cloud-init.log, atlas.out
│   ├─► PC microservices fail? → NAT Gateway, outbound rules
│   └─► ENG-742935 / ENG-776010? → Apply workaround
│
├─► AWS:
│   ├─► Instance/IAM/VPC limits? → Fix permissions or quotas
│   └─► Node boot timeout? → Check subnet, clusters-agents-upgrader.log
│
├─► Agent-level: Cluster ops fail? → Cluster Agent. Node tasks fail? → Host Agent. Disconnected? → Infra Gateway
└─► Always: Review cluster-agent.log (source of truth)
```
