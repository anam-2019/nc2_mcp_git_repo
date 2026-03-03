---
name: nc_ahv/cloudnet
description: Troubleshooting CloudNet (CNC) service on NC2 AHV hosts. Use when overlay networking, ENI, or OVS/OVN issues occur.
---

# nc_ahv/cloudnet

This skill covers the CloudNet service (Cloud Network Controller / CNC), a core component used across NC2 on AWS, NC2 on Azure, and NC2 on GCP.

---

## Overview

- **Installation:** AHV (Bare Metal) hosts in all NC2 deployments
- **Role:** Abstraction layer that allows the standard Nutanix AHV networking stack to function on top of different public cloud networking constructs (AWS VPC, Azure VNet, etc.)

### Quick Status Check

| Check | Command / Location |
|-------|---------------------|
| Service Status | `systemctl status cloudnetd` (or `cloudnet`) |
| Logs | `/var/log/cloudnet/cloudnet.log` (on AHV host) |

---

## What Cloudnet Does

From a deployment perspective, cloudnet acts as the bridge between the AHV/OVS (Open vSwitch) networking stack and the Public Cloud's "underlay" network.

| Function | Description |
|----------|-------------|
| **Underlay Network Configuration** | During node provisioning (cluster creation or expansion), cloudnet initializes networking interfaces on the bare-metal host. Ensures OVS configuration aligns with cloud provider requirements (bonds/bridges for the specific bare-metal instance type). |
| **Cloud API / Metadata Interaction** | Communicates with the cloud provider's instance metadata service and APIs to discover network topology, IP configurations, and environment-specific details. |
| **IPAM Integration** | For Native Networking (UVMs get IPs directly from VPC/VNet), cloudnet manages secondary IP addresses and ENIs (or Azure equivalents) attached to the host, ensuring UVM traffic is correctly routed. |
| **Health & Connectivity Checks** | Monitors underlay network connectivity. If cloudnet fails or crashes, the node may fail to join the cluster or lose network connectivity. |

---

## PC + PE + AHV Service Chain

| Layer | Component | Role | Key Command / Log |
|-------|-----------|------|-------------------|
| **PC** | Atlas | Source of truth for logical network (VPCs, overlay subnets, VPNs). Manages global IPAM for overlay. | `atlas_cli cloud_config.get` |
| **PE (CVM)** | Network Service | Local agent for Atlas. Receives logical configs from PC and instructs hosts (e.g., PlugNic RPC). Reserves physical IPs back to Atlas. | `~/data/logs/network_service.out` — investigate if entity creation fails |
| **AHV** | Cloudnet (CNC) | Leaderless service on every host. Talks to cloud APIs (manage ENIs/IPs) and programs OVS to steer traffic. | `cloudnet.log` — ENI creation and programming |

---

## AHV Troubleshooting Commands

### Service Status

```bash
systemctl status cloudnetd
# or
systemctl status cloudnet
```

### OVN Connectivity

```bash
ovn-appctl connection-status
```

Check basic connectivity between ovn-controller and ovsdb.

```bash
sudo systemctl status ovn-controller
sudo systemctl restart ovn-controller
```

### OVS Bridge Check

```bash
ovs-vsctl show | grep brAtlas
```

### Other Services

```bash
systemctl restart fgwagent
```

### Log Locations

| Log | Purpose |
|-----|---------|
| `/var/log/cloudnet/cloudnet.log` | CloudNet agent, ENI creation |
| `/var/log/ovn/ovn-controller.log` | OVN controller; check for "connection dropped" |

```bash
tail -f /var/log/cloudnet/cloudnet.log
tail -f /var/log/ovn/ovn-controller.log
```

### Connectivity Tests

```bash
netstat -anlp | grep cloudnet
nc -zv x.x.x.x <cloudnet port no>
```

### Packet Capture (Traffic Debugging)

```bash
tcpdump -i eth2 -nne host x.x.x.x -Q out
tcpdump -i eth2 -nne host x.x.x.x -Q in
```

Packet from PC to VPN gateway FIP: verify traffic exiting eth2 (br0.uvms) and entering eth1 (attached to brOverlay0).

---

## Troubleshooting Flow Summary

```
CloudNet / Overlay Issue
│
├─► Service down? → systemctl status cloudnetd
│
├─► OVN connectivity? → ovn-appctl connection-status
│   └─► "connection dropped" in ovn-controller.log? → restart ovn-controller
│
├─► OVS bridge missing? → ovs-vsctl show | grep brAtlas
│
├─► Entity creation fails? → Check network_service.out (PE/CVM)
│
└─► Traffic flow issue? → tcpdump on relevant interfaces
```
