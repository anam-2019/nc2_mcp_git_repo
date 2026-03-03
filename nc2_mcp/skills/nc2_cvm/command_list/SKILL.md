---
name: nc2_cvm/command_list
description: Quick reference of NC2 and Nutanix CLI commands for troubleshooting. Use when you need to find the right command for memory, disk, CVM, AHV, PC, or service checks.
---

# nc2_cvm/command_list

Quick reference of essential commands for NC2 and Nutanix cluster troubleshooting across HW, CVM, AHV, and PC.

---

## HW — Memory

```bash
ncli alert history duration=4
ncli alert history auto-resolved=true duration=1 resolved=true

allssh 'sudo cat /proc/meminfo | grep -i total'
```

| Action | Command |
|--------|---------|
| Check memory size via scheduler | `links --dump http://x.x.x.x:2030/sched` |

### Put Node into Maintenance

```bash
# Check only (no action)
acli host.enter_maintenance_mode_check x.x.x.x

# Enter maintenance
acli host.enter_maintenance_mode x.x.x.x

# Exit maintenance
acli host.exit_maintenance_mode <host_ip>
```

### Connect to Affected CVM & Power Down

```bash
ssh x.x.x.x
cvm_shutdown -P now

# Or SSH to host to power down
ssh x.x.x.x
```

### Power Off Node

```bash
shutdown now
# or
shutdown -h now
```

---

## CVM — General

```bash
cat /proc/meminfo | grep Mem
svmips | wc -l
cs | grep -v UP
nodetool -h0 ring | grep Normal | grep -c Up
```

### Zeus / Config

```bash
zeus_config_printer | grep dyn_ring_change_info -A9
ncc/panacea/tools/bin/zeus_config_printer.py -f -t node_List
```

### CVM Lifecycle

```bash
cvm_shutdown -P now
virsh list --title --all
virsh start <cvm_name>
watch -d genesis status
hostssh 'virsh list | grep -v CVM | grep -c running'
```

### Cluster / Host / VM Info

```bash
acli vm.list power_state=on | grep -v "VM UUID" | wc -l
ncli cluster get-domain-fault-tolerance-status type=node | grep Tolerance
ncli host ls | awk -v RS= '/x.x.x.x/'
acli vm.list                    # display host IP
ncli vm ls name=<vm_name>       # display VM IP
ncli host edit id=<cvm_host_id> enable-maintenance-mode=[true|false]
ncli host get-rm-status
```

### Genesis / Rolling Restart

```bash
zkcat /appliance/logical/genesis/node_shutdown_token
zkcat /appliance/logical/genesis/rolling_restart_znode
tail -F ~/data/logs/genesis.out
zeus_config_printer | egrep "dyn_ring_change | node_removal_ack"
cassandra_status_history_printer
```

---

## Disk Troubleshooting

```bash
df -h | grep -i stargate | wc -l
list_disks                      # slot and serial
lsblk                           # /dev/sdX mounting path
lsscsi
ncli disk ls | awk -v RS= '/serial_no/'
zeus_config_printer | grep -i <disk_serial_number>
zeus_config_printer | grep is_degraded
allssh "grep Marking ~/data/logs/dynamic_ring_charger.INFO*"
```

### Host / Disk RM Status

```bash
ncli host ls | awk -v RS= '/x.x.x.x/'
ncli host edit id=<#> is-degraded=false
cassandra_status_history
progress_monitor_cli --fetchall
ncli disk get-rm-status
ncli host get-rm-status
```

### Logs

```bash
tail -f data/logs/hades.out | grep <disk_serial_no>
links 127.0.0.1:2010 | grep -i master    # curator
dmesg -T | grep -i ext4                   # -T is timestamp
dmesg -T | grep -i oom
```

| Tool | Purpose |
|------|---------|
| `edit-hades -p \| less` | Prism UI reads disk info from hades |
| `hades.out` | Hades log |

### AHV / HBA

```bash
ssh root@x.x.x.x   # AHV
# /var/log/NTNX.serial.out.0 — HBA timeout
```

### Disk Config / Removal

```bash
cat /home/nutanix/data/stargate-storage/disks/Z1ZBAHZ0/disk_config.json
ncli disk rm-start id=135646378 force=true
```

---

## SW — Software / Upgrade

```bash
progress_monitor_cli --fetchall
allssh cat ~/config/upgrade* | tail -2
allssh cat ~/config/upgrade.history       # AOS
hostssh cat /etc/nutanix-release          # AHV
hostssh "uname -a"
```

### Version Checks

```bash
ncc --version
ncli cluster version
stargate --version
upgrade_status                           # PC version
ncli multicluster get-cluster-state      # PC version & IP
ncli cluster version                     # or info
lcm_upgrade_status
```

### Cluster Health / Restart

```bash
cs | grep -v UP
show_leaders
genesis stop prism ; sleep 5 ; cluster start
ncli cluster get-domain-fault-tolerance-status type=node
```

---

## ecli

```bash
ecli task.list component_list=Lazan limit=100 | egrep -i 'kQueued|kRunning' | wc -l
ecli task.list component_list=Cerebro include_completed=false
```

---

## gflag

```bash
allssh "cat ~/config/acropolis.gflags"
links http://x.x.x.x:2220/h/gflags | grep polaris
links -dump http://x.x.x.x:2070/h/gflags | grep -i magneto_xat_force_xat_table_refresh   # view setting
allssh curl http://x.x.x.x:2070/h/gflags?magneto_xat_force_xat_table_refresh=True        # apply change
allssh "source /etc/profile; genesis stop stargate && cluster start; sleep 120"
curl -s http://x.x.x.x:2060/h/gflags | grep vpn
```

---

## Alert

```bash
ncli alert history duration=4
ncli alert history auto-resolved=true duration=1 resolved=true
panacea_cli show_all_ips
panacea_cli show_node_list
```

---

## Service Leader

```bash
panacea_cli show_leaders
/home/nutanix/ncc/panacea/tools/bin/zeus_config_printer.py -t leaders_list
```

### Leader Check by Service

| Service | Command |
|---------|---------|
| Curator | `links 127.0.0.1:2010` &#124; `grep -i master` |
| Stargate | `links 127.0.0.1:2009` |
| Cerebro | `links 127.0.0.1:2020` or `cerebro_cli get_master_location 2> /dev/null` |
| Acropolis | `links 127.0.0.1:2030` &#124; `grep Master` |
| Prism | `curl localhost:2019/prism/leader` |
| LCM | `lcm_leader` |
| Genesis | `convert_cluster_status` or `ntpq -pn` |
| Zookeeper | `links http:0:9876` or `zkServer.sh` |

### Zookeeper Leaders (Script)

```bash
( ztop=/appliance/logical/leaders; for z in $(zkls $ztop | egrep -v 'vdisk|shard'); do [[ "${#z}" -gt 40 ]] && continue; leader=$(zkls $ztop/$z | grep -m1 ^n) || continue; echo "$z" $(zkcat $ztop/$z/$leader | strings); done | column -t; )
```

### Service Restart

```bash
watch -d genesis status
allssh genesis stop magneto
cluster start
genesis restart scavenger    # wait a few minutes
```

### Stargate Leader

```bash
allssh 'zgrep "W2022" ~/data/logs/stargate*INFO*'
less acropolis.out | grep -i kAcropolis | tail -3
```

---

## NTP

```bash
ntpstat
allssh ntpq -pn
systemctl status ntpd.service
netstat -tulnp | grep ":123"
nc -vvv 0.us.pool.ntp.org 443

# NC2 / Chrony
ncli cluster get-ntp-servers
ntpdate -q <ntp_server_ip>
cat /etc/chrony.conf
allssh "sudo chronyc -n sources -v"
```

---

## DNS

```bash
ncli cluster get-name-server
cat /etc/resolv.conf
nslookup x.x.x.x
allssh cat /etc/dnsmasq.conf
hostssh "cat /etc/resolv.conf"
nc -vz x.x.x.x 53
```

---

## Network

```bash
curl -v https://downloads.cloud.nutanix.com/echo42
nc -zv x.x.x.x 9440
allssh 'for i in x.x.x.x y.y.y.y ; do nc -z -w 1 -v $i 2009; done'
```

### Outbound Reachability

```bash
curl -v --silent --head https://downloads.cloud.nutanix.com/*
curl -v --silent --head https://apikeys.nutanix.com/*
curl -v https://gateway-external-api.cloud.nutanix.com/echo42
```

### tcpdump

```bash
tcpdump -i eth0 -enn icmp and host <host>
```

---

## NC2

```bash
ncli multicluster get-cluster-state
curl ifconfig.me
curl -v https://gateway-external-api.cloud.nutanix.com/echo42
```

---

## PC — Prism Central

### MSP / Cluster Health

```bash
mspctl cluster list
mspctl cls health
mspctl cls health -c addons
mspctl cluster health prism-central --verbose
mspctl controller version --verbose
mspctl task list
panacea_cli show_leaders | grep -i msp
```

### Logs

| Log | Purpose |
|-----|---------|
| `~/data/logs/msp_controller.out` | MSP controller |
| `~/data/logs/genesis.out` | Genesis |
| `journalctl -u <service-name>` | Service journal |
| `sudo kubectl logs -n <namespace> <podname> <container_name>` | Pod logs |

### Services

```bash
allssh systemctl status docker.service
allssh systemctl status kubelet-master.service
allssh systemctl status registry
```

### Kubernetes

**Namespaces:** `kube-system`, `ntnx-system`, `ntnx-base`

```bash
sudo kubectl get pods -A
sudo kubectl get pods -n <namespace> <container_name>
sudo kubectl get pods -n ntnx-base
sudo kubectl get pods -A | egrep -v 'Running|Completed'
sudo kubectl describe pod <podname> -n <namespace>
sudo kubectl get pod -A -o wide | grep mspdns
sudo kubectl get svc -A | grep iam
sudo kubectl get cm -A | grep lb
```

### Atlas (Flow Virtual Networking)

```bash
atlas_cli config.get                    # "enable_atlas_networking: True"
atlas_cli subnet.list                   # overlay subnet
atlas_cli network_controller.list       # deployed state
atlas_cli vpn_gateway.list
atlas_cli virtual_network.list
```

| Log | Purpose |
|-----|---------|
| `/home/nutanix/data/logs/atlas.out` | `grep -i deploy` |

---

## CVM — NC2 Specific

```bash
systemctl list-units --type=service | grep -i nutanix
systemctl status cluster-agent-service.service | grep running
systemctl status infra-gateway-service.service
```

| Log | Purpose |
|-----|---------|
| `/var/log/cluster-agent.log` | Cluster agent |
| `/var/log/infra-agent.log` | Infra agent |

```bash
hostssh "ovn-appctl connection-status"   # expected: connected
```

---

## AHV

### OVS / OVN

```bash
virsh list --title
ovs-vsctl show
hostssh "ovs-vsctl show | grep brAtlas"
hostssh "ifconfig | grep eth | grep mtu"
ifconfig eth0 mtu 9000 up
```

| Log | Purpose |
|-----|---------|
| `/var/log/ovn/ovn-controller.log` | OVN controller |

### Agents

```bash
systemctl status host-agent-service.service
systemctl status cluster-agent-upgrader
```

| Log | Purpose |
|-----|---------|
| `/var/log/ahv-host-agent.log` | AHV host agent |
| `/var/log/host-agent.log` | Host agent |
| `boot.log` | Boot |
| `cloud-init.log` | Cloud init |
| `journald_boot.log` | Journal boot |

### CloudNet

| Log | Purpose |
|-----|---------|
| `/var/log/cloudnet/cloudnet.log` | CloudNet agent |
