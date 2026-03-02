---
name: pc-fgw-authentication-troubleshooting
description: Troubleshoot PC and FGW authentication issues, certificate registration failures, and MC service chain tasks. Use when FGW is down for systematic log analysis.
---

# nc2_pc/authentication

This skill provides a systematic troubleshooting workflow for NC2 authentication issues across PC, AHV, and CVM components.

## PC (Prism Central) Troubleshooting

### Step 1: Verify Flow Gateway Status
Check the flow gateway status using Atlas CLI:
```bash
atlas_cli flow_gateway.list
atlas_cli flow_gateway.get
```

### Step 2: If Flow Gateway is Down

When flow gateway is down, verify the following:

#### Check DNS Status
Ensure DNS resolution is working properly.

#### Check NTP Status
Verify NTP synchronization:

```bash
allssh "sudo chronyc -n sources -v"
```

#### Verify API Keys Connectivity
Ensure the system can reach the Nutanix API keys endpoint:

```
https://apikeys.nutanix.com
```

### Step 3: Review Log Files

#### Prism Proxy Access Log
Check for trust-related v4 API calls:

```bash
allssh "sudo cat /home/apache/ikat_access_logs/prism_proxy_access_log.out | grep -i v4 | grep -i trust | tail -n5"
```

#### Atlas Log
Review `atlas.out` for errors and status messages.

#### JWT Expiration Errors
Check for expired JWT token issues:

```bash
allssh "grep 'Error due to expired JWT' ~/adonis/logs/prism-service.*"
```

---

## AHV Troubleshooting

### Check OVS Bridge Configuration

Verify the Atlas bridge exists:

```bash
ovs-vsctl show | grep brAtlas
```

### Check OVN Connection Status

```bash
ovn-appctl connection-status
```

### Review Log Files

| Log File | Description |
|----------|-------------|
| `/var/log/ahv-host-agent.log` | AHV host agent logs |
| `cloudnet/cloudnet.log` | Cloud network logs |

---

## CVM Troubleshooting

### Review Log Files

| Log File | Description |
|----------|-------------|
| `cluster-agent.log` | Cluster agent logs |
| `infra-gateway.log` | Infrastructure gateway logs |

---

## Troubleshooting Flow Summary

```
1. PC: Check flow_gateway status (atlas_cli)
   │
   ├─► If DOWN:
   │   ├─► Verify DNS
   │   ├─► Verify NTP (chronyc)
   │   └─► Verify connectivity to https://apikeys.nutanix.com
   │
   └─► Review logs:
       ├─► prism_proxy_access_log.out
       ├─► atlas.out
       └─► prism-service.* (JWT errors)

2. AHV: Check OVS/OVN status
   │
   ├─► ovs-vsctl show | grep brAtlas
   ├─► ovn-appctl connection-status
   │
   └─► Review logs:
       ├─► /var/log/ahv-host-agent.log
       └─► cloudnet/cloudnet.log

3. CVM: Review logs
   │
   ├─► cluster-agent.log
   └─► infra-gateway.log
```