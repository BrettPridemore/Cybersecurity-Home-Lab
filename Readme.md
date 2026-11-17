# Lab 01: Initial Setup (Isolated Network, IDS Sensor, and SIEM)

**Status:** ✅ Complete
**Build dates:** September – November 2026
**Last updated:** November 16, 2026

---

## 1. Summary
This lab builds the foundation of my home Security Operations Center (SOC). It has three parts:

- **An isolated lab network** where an attacker VM (Kali) and a vulnerable target VM (Metasploitable) can talk to each other but can't reach my home network.
- **A sensor VM** that listens to all lab traffic and analyzes it with Suricata (alerts) and Zeek (connection logs).
- **A SIEM (Wazuh)** running in Docker on my MacBook that collects the sensor's logs and shows them on a dashboard.

By the end of this lab, traffic between Kali and Metasploitable shows up in the Wazuh dashboard as both Suricata alerts and Zeek connection records.

---

## 2. Objectives
| # | Objective | Result |
|---|-----------|--------|
| 1 | Build an isolated virtual network for the vulnerable machines | ✅ Done |
| 2 | Deploy an attacker VM (Kali) and a target VM (Metasploitable) | ✅ Done |
| 3 | Deploy a sensor VM that passively captures all lab traffic | ✅ Done |
| 4 | Install Suricata (IDS) and Zeek (network monitor) on the sensor | ✅ Done |
| 5 | Verify both tools capture live traffic | ✅ Done |
| 6 | Run Wazuh SIEM on a separate machine | ✅ Done |
| 7 | Forward sensor logs to Wazuh with the Wazuh agent | ✅ Done |
| 8 | Get Suricata alerts into the Wazuh dashboard | ✅ Done |
| 9 | Write a custom Wazuh rule so Zeek logs show up in the dashboard | ✅ Done |

---

## 3. Environment

### Architecture
```
   Isolated lab network: VMnet2 (10.10.10.0/24, no DHCP, no internet)
 ┌─────────────┐          ┌──────────────────┐
 │    Kali     │◄────────►│  Metasploitable  │
 │ 10.10.10.10 │          │   10.10.10.20    │
 └─────────────┘          └──────────────────┘
          │  sensor listens to everything on VMnet2
          ▼
 ┌────────────────────────────┐
 │  Sensor VM (Ubuntu Server) │
 │  ens33: listen only, no IP │
 │  ens34: management (NAT)   │
 │  Suricata + Zeek + agent   │
 └─────────────┬──────────────┘
               │  Wazuh agent sends logs (TCP 1514/1515)
               ▼
 ┌────────────────────────────┐
 │  MacBook (home Wi-Fi)      │
 │  Wazuh SIEM in Docker      │
 │  Reserved IP via DHCP      │
 └────────────────────────────┘
```
The VMs run in VMware Workstation on a wired Windows desktop. The Wazuh SIEM runs on a separate MacBook so the monitoring system isn't on the same machine as the lab.

<!-- TODO: Add screenshots later  VMware Virtual Network Editor showing VMnet2 (host-only, DHCP off, no host adapter) and VMnet8 (NAT) -->

### Machines
| Machine | Network adapter(s) | IP address |
|---------|-------------------|------------|
| Kali | VMnet2 only | `10.10.10.10` (static) |
| Metasploitable | VMnet2 only (NAT adapter removed) | `10.10.10.20` (static) |
| Sensor (Ubuntu Server 24.04, 2 vCPU, 6 GB RAM) | `ens33` on VMnet2, `ens34` on VMnet8 (NAT) | `ens33`: none, `ens34`: DHCP from VMware NAT |
| MacBook (SIEM) | Home Wi-Fi | Reserved IP on the home router |

<!-- TODO: Add screenshots later Sensor VM settings showing Network Adapter on VMnet2 and Network Adapter 2 on NAT> -->

---

## 4. Tools Used
| Tool | What it does in this lab |
|------|--------------------------|
| VMware Workstation | Runs the VMs and creates the isolated network |
| Kali Linux | Attacker machine |
| Metasploitable | Intentionally vulnerable target |
| Ubuntu Server | Sensor operating system |
| Suricata + Emerging Threats Open rules | IDS that raises alerts when traffic matches known attack patterns (about 52,800 rules) |
| Zeek | Logs every connection on the network (who talked to who, when, and how much) |
| Wazuh server | SIEM that collects, stores, and displays the logs |
| Wazuh agent | Runs on the sensor and ships Suricata and Zeek logs to Wazuh |

### Security principles applied
- **Network segmentation:** the vulnerable machines are on a network with no route out.
- **Reduced attack surface:** Metasploitable's NAT adapter was removed completely, not just disconnected.
- **Defense in depth:** Suricata (signature alerts) and Zeek (connection logs) cover different things.
- **Separation of duties:** the sensor only watches, and the SIEM lives on a different machine.

---

## 5. Procedure

### 5.1 Build the isolated network
In VMware's **Virtual Network Editor**, I set up VMnet2 as host-only with DHCP off, subnet `10.10.10.0/24`, and **no host virtual adapter**, so my Windows desktop isn't part of the lab network. Kali and Metasploitable are attached to VMnet2 only, with static IPs.

Metasploitable's static IP (in `/etc/network/interfaces`):
```
auto eth0
iface eth0 inet static
    address 10.10.10.20
    netmask 255.255.255.0
```

<!-- TODO: Add screenshots later  Kali successfully pinging Metasploitable (`ping 10.10.10.20`)-->

### 5.2 Build the sensor VM
Ubuntu Server 24.04 with two network adapters:
- **ens33 (VMnet2):** the listening interface. It has no IP address so nothing on the lab network can talk to it.
- **ens34 (NAT):** the management interface, used for updates and for sending logs to Wazuh.

I set the listening interface to promiscuous mode (so it sees all traffic, not just traffic addressed to it) and turned off NIC offloading (which merges packets and breaks IDS signature matching):
```bash
sudo ip link set ens33 up promisc on
sudo ip addr flush dev ens33
sudo ethtool -K ens33 gro off lro off tso off gso off
```

### 5.3 Install Suricata
```bash
sudo apt install -y suricata
sudo suricata-update
```
In `/etc/suricata/suricata.yaml`, I set `HOME_NET` to `10.10.10.0/24`, set the capture interface to `ens33`, and made sure `eve-log` (JSON output) was on. Then started it with `sudo systemctl enable --now suricata`.

### 5.4 Install Zeek
Installed from the official Zeek repository and set the interface to `ens33` in `/opt/zeek/etc/node.cfg`.:

```echo 'deb https://download.opensuse.org/repositories/security:/zeek/xUbuntu_24.04/ /' | sudo tee /etc/apt/sources.list.d/security:zeek.list
curl -fsSL https://download.opensuse.org/repositories/security:zeek/xUbuntu_24.04/Release.key | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/security_zeek.gpg > /dev/null
sudo apt update
sudo apt install zeek-8.0
```
<!-- TODO: Add screenshots later  Zeek `conn.log` showing a Kali → Metasploitable connection -->

### 5.5 Run Wazuh on the MacBook
```bash
git clone https://github.com/wazuh/wazuh-docker.git -b v4.9.0
cd wazuh-docker/single-node
docker compose -f generate-indexer-certs.yml run --rm generator
docker compose up -d
```
The dashboard is at `https://localhost` on the Mac.

<!-- TODO: Add screenshots later  Wazuh dashboard home page after logging in-->

### 5.6 Connect the sensor to Wazuh
First I confirmed the sensor could reach the Wazuh ports on the Mac:
```bash
nc -zv -w 3 <MAC_IP> 1514    # agent events
nc -zv -w 3 <MAC_IP> 1515    # agent enrollment
```
Then installed the Wazuh agent on the sensor:
```bash
sudo WAZUH_MANAGER="<MAC_IP>" apt install -y wazuh-agent=4.9.0-1
sudo apt-mark hold wazuh-agent
```
In `/var/ossec/etc/ossec.conf`, I added the Suricata and Zeek logs so the agent reads them:
```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>

<localfile>
  <log_format>json</log_format>
  <location>/opt/zeek/logs/current/*.log</location>
</localfile>
```
Then started the agent with `sudo systemctl enable --now wazuh-agent`.

<!-- TODO: Add screenshots later  `sudo systemctl status wazuh-agent` showing **active (running)**-->

### 5.7 Add a custom rule for Zeek
Wazuh only saves events that match a rule. It has built-in rules for Suricata but none for Zeek, so Zeek logs were reaching Wazuh and being thrown away. I wrote a custom rule that matches any Zeek record (every Zeek record has a `uid` and connection IPs):

```xml
<group name="zeek,">
  <rule id="100200" level="3">
    <decoded_as>json</decoded_as>
    <field name="uid">\.+</field>
    <field name="id_orig_h">\.+</field>
    <description>Zeek: $(id_orig_h) -> $(id_resp_h):$(id_resp_p)</description>
  </rule>
</group>
```
- **level 3** is the lowest level that still gets saved to the dashboard.
- **id 100200** is in the custom range (100000+) so it doesn't clash with Wazuh's own rules.

Loaded it into the manager container and gave it the right owner:
```bash
docker compose cp local_rules.xml wazuh.manager:/var/ossec/etc/rules/local_rules.xml
docker compose exec wazuh.manager chown wazuh:wazuh /var/ossec/etc/rules/local_rules.xml
docker compose exec wazuh.manager chmod 660 /var/ossec/etc/rules/local_rules.xml
docker compose restart wazuh.manager
```
Then tested it with `wazuh-logtest` by pasting a sample Zeek line and confirming it matched rule 100200.

<!-- TODO: Add screenshots later  `wazuh-logtest` output showing Phase 3 matched rule 100200 -->

### 5.8 Test the full pipeline
From Kali, I pinged and scanned Metasploitable. Then in Wazuh **Threat Hunting**, with the sensor agent selected, I searched:
- `rule.groups:suricata` for Suricata alerts
- `rule.groups:zeek` for Zeek connection records

---

## 6. Results / Findings
- The sensor's agent enrolled with Wazuh and shows as **Active**.
- An Nmap scan produced **9 Suricata alerts** in Wazuh, a mix of Emerging Threats (ET) signatures and Suricata's built-in protocol anomaly alerts.
- A ping from Kali showed up in Wazuh as a Zeek record: `Zeek: 10.10.10.10 -> 10.10.10.20`.

<!-- TODO: Add screenshots later  Wazuh **Agents** page showing the sensor as Active -->
<!-- TODO: Add screenshots later  Threat Hunting filtered to `rule.groups:suricata` showing the alerts -->
<!-- TODO: Add screenshots later  One expanded Suricata alert showing the `data.alert.signature` field -->
<!-- TODO: Add screenshots later Threat Hunting filtered to `rule.groups:zeek` showing the Kali → Metasploitable record -->

---

## 7. Troubleshooting
| Problem | Cause | Fix |
|---------|-------|-----|
| Wazuh indexer wouldn't start after generating certificates | Known issue in the v4.9.0 cert generator: the manager expects `root-ca-manager` files that aren't created | Copied `root-ca.pem` / `root-ca.key` to `root-ca-manager.pem` / `root-ca-manager.key` in `config/wazuh_indexer_ssl_certs` |
| Ports 1514/1515 still failed after the network was fixed | Docker Desktop wasn't running, so Wazuh wasn't running | Started Docker and set it to launch at login |
| Wazuh agent failed to start | A typo in `ossec.conf` (an extra `<`) and the manager address left as `MANAGER_IP` | Fixed both lines and restarted the agent |
| Agent got "connection refused" after I changed the rule | `wazuh-control restart` inside the container didn't bring every manager service back up | Restarted the whole container with `docker compose restart wazuh.manager` |
| Zeek alerts were created but never showed in the dashboard | Zeek's dotted field names (`id.orig_h`) clashed with Wazuh's existing `data.id` field, so the database rejected them | Set `redef Log::default_scope_sep = "_";` in Zeek and updated the rule to `id_orig_h` |

<!-- TODO: Add screenshots later  Virtual Network Editor showing VMnet8 set to Host-only (the "before")-->

---

## 8. Lessons Learned
- **Troubleshoot one layer at a time.** Checking IP address → ping → port → service step by step found each problem faster than guessing.
- **A SIEM needs a fixed address.** If the server's IP changes, every agent loses it.
- **Check that services survive a reboot.** Docker not starting broke the pipeline even though everything was configured correctly.
- **Follow the data through each stage.** For Zeek, I checked the pathway data took (Zeek log → agent → rule → alerts file → dashboard) and found the break was in the last step, not the rule.
- **Verify isolation, don't assume it.** Zeek caught the host adapter on the lab network that I thought was off.

---

## 9. Next Steps
- **Lab 02:** run recon and login attacks from Kali against Metasploitable and compare what Suricata, Zeek, and Wazuh each detect.
- Perform alert tuning for Zeek and Suricata rules.
- Create custom alerts based on common traffic.
