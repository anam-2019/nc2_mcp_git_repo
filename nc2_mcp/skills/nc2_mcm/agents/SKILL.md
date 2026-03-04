---
name: nc2_mcm/agents
description: NC2 agent architecture and troubleshooting. Use when cluster agents, host agents, or MCM connectivity issues occur.
---

# nc2_mcm/agents

This skill covers the NC2 agent architecture across AHV, CVM, PC, and FGW. Agents run as services on AOS and AHV, similar to other native Nutanix services.

---

## Agent Summary

| Agent | Location | Role |
|-------|----------|------|
| **Upgrader-agent** (cluster-agent-upgrader) | AHV | Installs and upgrades RPMs on the node |
| **Host-agent** (ahv-host-agent / AHV gateway) | AHV | Node-level ops: CVM health, support tunnel (Azure), starting cluster-agent on CVM. Runs on each AHV host. |
| **Cluster-Agent** | CVM | Communicates with MCM; forwards requests to Infra Gateway. Manages cluster creation, expansion, node add. |
| **Infra Gateway Agent** | CVM | Receives cluster-level ops from Cluster-Agent; forwards to MCM. Tells CVMs when nodes are added. |
| **CloudNet Agent** (cloudnet / CNC) | AHV | Creates subnets and host-based network resources (e.g., br0). Runs on each AHV host. |
| **Network Service Agent** | CVM (NC2 on AWS only) | Manages network/VM creation-deletion, power on/off, migration, vNIC updates, Security Group updates for ENIs |

---

## AHV — Logs & Commands

### Primary Logs

| Log | Purpose |
|-----|---------|
| `/var/log/ahv-host-agent.log` | AHV host agent |
| `/var/log/host-agent.log` | Host agent |

### Other Logs

| Log | Purpose |
|-----|---------|
| `boot.log` | Boot sequence |
| `cloud-init.log` | Cloud init / bootstrap |
| `journald_boot.log` | Journal boot logs |
| `/var/log/clusters-agents-upgrader.log` | Upgrade agent (RPM install/upgrade) |
| `/var/log/cloudnet/cloudnet.log` | CloudNet agent |

### Configuration

```bash
cat /etc/ahv-host-agent.yaml
```

### CloudNet

```bash
systemctl status cloudnetd
tail -f /var/log/cloudnet/cloudnet.log
```

---

## CVM — Logs & Commands

| Log | Purpose |
|-----|---------|
| `~/data/logs/cluster-agent.log` | Cluster agent (leader CVM has active logs) |
| `~/data/logs/infra-gateway.log` | Infra Gateway agent |
| `~/data/logs/network_service.out` | Network Service agent (NC2 on AWS only) |

---

## PC — Logs

| Log | Purpose |
|-----|---------|
| `~/data/logs/genesis*` | Genesis logs |
| `~/data/logs/aplos*` | Aplos logs |

---

## FGW — Logs

| Log | Purpose |
|-----|---------|
| `generic-agent.log` | Generic agent |
| `fgwagent.log` | FGW agent |

---

## Service Status by Component

### At CVM — Cluster Agent Status

```bash
systemctl list-units --type=service | grep -i nutanix
systemctl status cluster-agent-service.service
systemctl status infra-gateway-service.service
```

**Note:** Only one cluster-agent will be active (leader). If the leader keeps changing, there is a problem.

### MCM Reachability

```bash
curl -vvv https://gateway-external-api.cloud.nutanix.com/echo42
```

### At AHV

```bash
systemctl list-units --type=service | grep -i nutanix
systemctl status host-agent-service.service
systemctl status cluster-agent-upgrader
systemctl status ahv-install-cvm.service
systemctl status
```

### Network Check (AHV)

```bash
ovs-vsctl show
nodetool -h 0 ring
virsh list --title
hostssh "ovs-vsctl show | grep brAtlas"
hostssh "ovn-appctl connection-status"
```

### At PC

```bash
panacea_cli show_leaders | grep -i msp
mspctl cluster health prism-central
mspctl cluster list
mspctl task list
sudo kubectl get pods -A
sudo kubectl get pods -A -o wide | grep mspdns
sudo kubectl get pods -A -o wide | grep kube-apiserver
sudo kubectl get pods -A -o wide | grep athena
```

---

## PC and FGW Service Chain

| Chain | Components |
|-------|------------|
| **OVN path** | OVN → Atlas → Adonis → fgwagent |
| **Mercury path** | Mercury → Envoy → ikat → generic agent |

---

## Background: Agent Roles

### Upgrader-agent (cluster-agent-upgrader)

Responsible for installing and upgrading RPMs on the node. Runs on AHV.

### Host-agent (ahv-host-agent / AHV gateway)

- Runs on each AHV host
- Node-level operations: ensuring CVM is up, configuring support tunnel for Azure, starting cluster-agent on CVM
- Handles host-specific operations such as installation

### Cluster Agent (on CVM)

- Communicates with MCM
- Forwards requests to the Infra Gateway Agent on behalf of the CVMs
- Manages cluster creation, expansion, and node add
- Leader-follower model; leader chosen by MCM

### Infra Gateway Agent (on CVM)

- Receives cluster-level operations from the Cluster-Agent
- Example: Cluster-Agent notifies that a node is being added → Infra Gateway tells the cluster (CVMs) to react accordingly
- Handles secure tunnel to MCM

### CloudNet Agent (on AHV host)

- Runs on each AHV host
- Creates subnets and host-based network resources such as br0

### Network Service Agent (on CVM, NC2 on AWS only)

Manages network creation/deletion, VM creation/deletion, VM power on/off, migration, VM vNIC updates, and Security Group updates for ENIs.

---

## Troubleshooting Flow Summary

```
Agent / MCM Issue
│
├─► Cluster disconnected? → Check Infra Gateway, test Echo42
│
├─► Node stuck (booting/installing)? → Host Agent logs
│
├─► Cluster ops fail? → Cluster Agent logs (leader CVM)
│
├─► Upgrade/RPM issues? → clusters-agents-upgrader.log
│
├─► Network/OVS issues? → CloudNet logs, ovn-appctl, ovs-vsctl
│
└─► PC/MSP issues? → mspctl, kubectl get pods
```
