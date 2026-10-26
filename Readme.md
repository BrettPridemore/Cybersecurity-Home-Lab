# Lab 01: Initial Setup (Isolated Network, IDS Sensor and SIEM)
**Status:** Complete through SIEM dashboard. Agent forwarding and attack testing continue in Lab02.

**Build dates:** September 2026

**Last updated:** October 16, 2026

---
## 1. Summary
This lab builds the foundation of a home Security Operations Senter (SOC): an isolated attack/target network, a dedicated sensor to monitor traffic, and a SIEM that collects and displays everything.
By the end of Lab01:
- Kali (Attack machine) and Metasploitable (Target machine) talk to each other on isolated network.
- IDS Sensor passively captures traffic and analyzes it with Suricata (alerting) and Zeek (connection logging)
- Wazuh (SIEM) runs on Docker on separate machine with dashboard accessible to view logs and alerts.

---

## 2. Objectives
| # | Objective | Result |
|---|-----------|--------|
| 1 | Build an isolated virtual network so intentionally vulnerable machines cannot reach my home network | Done |
| 2 | Deploy an attacker VM and a vulnerable target VM on that network | Done |
| 3 | Deploy a sensor VM that passively captures all lab traffic | Done |
| 4 | Install an IDS (Suricata) and network monitor (Zeek) on the sensor | Done |
| 5 | Verify both tools capture live traffic | Done |
| 6 | Stand up a SIEM (Wazuh) on a separate machine | Done |
| 7 | Forward sensor logs to the SIEM | Lab 02 |
| 8 | Run attacks and map detections to MITRE ATT&CK | Lab 02 |

--- 

## 3. Architecture
```
┌─────────────┐        ┌──────────────────┐        ┌──────────────────────┐
│    Kali     │        │  Metasploitable  │        │   Sensor VM (ids)    │
│ 10.10.10.10 │◄──────►│   10.10.10.20    │◄──────►│ ens33: sniff, no IP  │
└─────────────┘        └──────────────────┘        │ ens34: mgmt (bridged)│
       │                        │                  └──────────┬───────────┘
       └────────────────────────┴──► VMnet2 (host-only,       │
                                      no DHCP, 10.10.10.0/24) │ home LAN
                                                              │
                                                   ┌──────────▼───────────┐
                                                   │ SIEM host (MacBook)  │
                                                   │ Wazuh in Docker      │
                                                   └──────────────────────┘
```
### Virtual networks (VMware Workstation)
| Network | Type | Purpose |
|---------|------|---------|
| VMnet2 | Host-only, DHCP disabled, `10.10.10.0/24`, host adapter disconnected | Isolated lab segment |
| VMnet0 | Bridged (automatic) | Sensor management NIC only |
| VMnet1, VMnet8 | Host-only (defaults) | Unused |

### Machine configuration
| Machine | Adapter(s) | Address |
|---------|-----------|---------|
| Kali | VMnet2 only | `10.10.10.10/24` (static) |
| Metasploitable | VMnet2 only (NAT adapter removed) | `10.10.10.20/24` (static) |
| Sensor (Ubuntu Server 24.04) | `ens33` on VMnet2, `ens34` on VMnet0 | `ens33`: none, `ens34`: DHCP |
| SIEM host | Home LAN | Wazuh via Docker Compose |

---

## 4. Tools and Frameworks
| Tool | Role | Why it is here |
|------|------|----------------|
| VMware Workstation | Hypervisor and virtual networking | Runs all three VMs and builds the isolated VMnet2 segment |
| Kali Linux | Attacker | Standard offensive security toolkit (nmap, Metasploit) |
| Metasploitable | Target | Deliberately vulnerable machine to attack safely |
| Ubuntu Server 24.04 LTS | Sensor OS | Lightweight, no GUI needed |
| Suricata 8.0.3 | Signature-based IDS | Raises alerts when traffic matches known attack patterns |
| Emerging Threats Open | Ruleset (about 52,800 rules loaded) | Free signatures for scans, exploits, and malware traffic |
| Zeek | Network security monitor | Logs every connection, DNS query, and file transfer as metadata |
| ethtool / systemd | Interface tuning | Promiscuous mode and offload settings that survive reboots |
| Wazuh 4.9.0 (manager, indexer, dashboard) | SIEM | Collects, correlates, and visualizes logs |
| Docker Desktop | Container runtime | Runs the Wazuh stack on the MacBook |
| nmap, Metasploit | Attack tools (Lab 02) | Generate recon and exploit traffic to test detection |
| MITRE ATT&CK | Framework (Lab 02) | Map each attack to a technique ID (for example T1046 Network Service Discovery) |

### Security principles applied
- **Network segmentation:** vulnerable machines live on a network with no outside route.
- **Least privilege / reduced attack surface:** Metasploitable's NAT adapter was removed from the VM hardware entirely, not just disconnected.
- **Defense in depth:** signature-based detection plus metadata logging.
- **Separation of duties:** the sensor only observes, and the SIEM is a separate system.

---

## 5. Procedure
### 5.1 Build the isolated network
1. In **Edit → Virtual Network Editor**, configure VMnet2 as host-only with DHCP off and subnet `10.10.10.0/24`.
2. Uncheck "Connect a host virtual adapter to this network."
3. Attach Kali and Metasploitable to VMnet2 only.
### 5.2 Static IP on Metasploitable
Metasploitable uses the classic `/etc/network/interfaces` file. Removing its NAT adapter and pointing the remaining adapter at VMnet2 (as a Custom network, not the generic host-only option), then:
```
auto eth0
iface eth0 inet static
    address 10.10.10.20
    netmask 255.255.255.0
```
Then `/etc/init.d/networking restart`.
### 5.3 Build the sensor VM
- Ubuntu Server 24.04, 4 vCPU, 8 GB RAM, OpenSSH server installed.
- Adapter 1: VMnet2 (sniffing). Adapter 2: Bridged to VMnet0 (management).
- After boot, `ip a` showed `ens33` (VMnet2, no IPv4) and `ens34` (home LAN, DHCP).
### 5.4 Prepare the sniffing interface
```bash
sudo ip link set ens33 up promisc on
sudo ip addr flush dev ens33
sudo ethtool -K ens33 gro off lro off tso off gso off
```
Offloads are disabled because otherwise the NIC hands the capture tools merged packets, which breaks signature matching. Made persistent with `/etc/systemd/system/sensor-iface.service` (oneshot unit running the same three commands, enabled at boot).
### 5.5 Suricata
```bash
sudo apt install -y suricata
sudo suricata-update
```
In `/etc/suricata/suricata.yaml`: `HOME_NET: "[10.10.10.0/24]"`, `af-packet` interface set to `ens33`, `eve-log` enabled. Then `sudo systemctl enable --now suricata`.
### 5.6 Zeek
Installed from the official Zeek repository (not in Ubuntu's default repos), then:
- `/opt/zeek/etc/node.cfg`: `interface=ens33`
- `/opt/zeek/share/zeek/site/local.zeek`: added `@load policy/tuning/json-logs`
- Deployed with `sudo /opt/zeek/bin/zeekctl deploy`
### 5.7 Wazuh SIEM (Docker on MacBook)
```bash
git clone https://github.com/wazuh/wazuh-docker.git -b v4.9.0
cd wazuh-docker/single-node
docker compose -f generate-indexer-certs.yml run --rm generator
# workaround for the cert bug, see Section 7:
cd config/wazuh_indexer_ssl_certs
cp root-ca.pem root-ca-manager.pem && cp root-ca.key root-ca-manager.key
cd ../..
docker compose up -d
```
Dashboard reached at `https://localhost` (self-signed certificate warning is expected), logging in with the credentials defined in the compose file.

---

## Notes and documentation

<img width="1919" height="1031" alt="image" src="https://github.com/user-attachments/assets/73efd887-ab06-46b4-87c6-c86daaa8d8ce" />

- Created two VMs on VMWare and was able to ping each other and get a response.

