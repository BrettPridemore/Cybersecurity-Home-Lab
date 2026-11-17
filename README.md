# Lab 02: Attack and Detect (Recon and Login Attacks vs. Suricata, Zeek, and Wazuh)

**Status:** ✅ Complete
**Build dates:** November 2026
**Last updated:** November 16, 2026

---

## 1. Summary
Using the pipeline built in Lab 01, I run common attacks from Kali against Metasploitable on the isolated lab network, then compare what each tool sees:

- **Suricata:** did it raise an alert, and which signature fired?
- **Zeek:** what connections did it record?
- **Wazuh:** what showed up in the dashboard, and how easy was it to spot?

The goal is to understand what each tool is good at, and where the gaps are.

---

## 2. Objectives
| # | Objective | Result |
|---|-----------|--------|
| 1 | Confirm the Lab 01 pipeline is healthy before testing | ✅ Done |
| 2 | Run network recon scans (Nmap) from Kali | ✅ Done |
| 3 | Run repeated SSH login attempts against Metasploitable | ✅ Done |
| 4 | Run repeated FTP login attempts against Metasploitable | ✅ Done |
| 5 | Capture and compare detections from Suricata, Zeek, and Wazuh | ✅ Done |
| 6 | Map each attack to MITRE ATT&CK | ✅ Done (T1046, T1110.001) |
| 7 | Identify detection gaps and recommend improvements | ✅ Done — carried into Lab 03 |

---

## 3. Environment
Same as Lab 01. See [Lab 01](../Lab01-Initial-Setup/) for the full architecture.

| Machine | Role | IP |
|---------|------|----|
| Kali | Attacker | `10.10.10.10` |
| Metasploitable | Target | `10.10.10.20` |
| Sensor | Suricata + Zeek + Wazuh agent | listens on VMnet2 |
| MacBook | Wazuh SIEM | Home network address |

---

## 4. Tools Used
| Tool | Role in this lab |
|------|------------------|
| Nmap | Network discovery and service scanning |
| Hydra | Repeated login attempts against SSH and FTP |
| Suricata | Signature-based alerts |
| Zeek | Connection records (`conn.log`, plus `ssh.log` and `ftp.log`) |
| Wazuh | Central view of all detections |
| MITRE ATT&CK | Framework for labeling each attack technique |

---

## 5. Procedure
### 5.1 Scenario 1: Network reconnaissance (Nmap)
**MITRE ATT&CK:** T1046 Network Service Discovery

**What I did:**
From Kali, I ran a SYN scan with service detection against Metasploitable:
```bash
date; nmap -sS -sV 10.10.10.20; date
```
- `-sS` checks which of the 1,000 most common ports are open
- `-sV` connects to each open port to identify the software running on it

| Run | Start | End | Duration |
|-----|-------|-----|----------|
| Scan 1 | 9:04:31 PM | 9:05:12 PM | ~12 sec |
| Scan 2 | 9:19:13 PM | 9:19:24 PM | ~11 sec |

Nmap found **23 open ports**, almost all running old, well-known vulnerable software (vsftpd 2.3.4, OpenSSH 4.7, Apache 2.2.8, Samba 3.x, and a "Metasploitable root shell" on port 1524). One 12-second scan gives an attacker a full target list, which is why catching recon early matters.

**Searches used in Wazuh:**
- `rule.groups:suricata`
- `rule.groups:zeek and data.id_orig_h:10.10.10.10`
- `rule.groups:zeek and data.conn_state:REJ` (refused = closed ports)
- `rule.groups:zeek and data.conn_state:SF` (completed = open ports that were talked to)

**What each tool saw (per scan, approximate):**
| Tool | Events | What they were |
|------|--------|----------------|
| Suricata | ~9 alerts | `ET SCAN Possible Nmap User-Agent Observed` (~8) and `SURICATA Applayer Detect protocol only one direction` (~1) |
| Zeek | ~1,100 records | ~980 refused connections (closed ports) + ~45 completed connections (open ports) |
| Zeek (extra logs) | new files | `ssh.log`, `ftp.log`, `http.log`, `software.log`, `known_services.log` were created because `-sV` actually talked to each service |

<!-- 📸 **Screenshot:** Nmap output on Kali-->
<!-- 📸 **Screenshot:** Suricata alerts in Wazuh for the scan-->
<!-- 📸 **Screenshot:** Zeek records in Wazuh showing the spread of ports-->

### 5.2 Scenario 2: SSH login attempts
**MITRE ATT&CK:** T1110.001 Brute Force: Password Guessing

**What I did:**
I used 50 passwords from Kali's built-in `rockyou.txt` wordlist plus the known password, then tried each one against Metasploitable's SSH service.

```bash
head -n 50 /usr/share/wordlists/rockyou.txt > ~/pw.txt
echo "msfadmin" >> ~/pw.txt
```

| Run | Start | End | Attempts | Duration |
|-----|-------|-----|----------|----------|
| SSH brute force | ~9:33 PM | 9:40:17 PM | 51 | ~7 min |

The attack found the valid password (`msfadmin`) on the last attempt.
<!-- Screenshot of kali finding last password-->

**Searches used in Wazuh:**
- `rule.groups:suricata`
- `rule.groups:zeek and data.id_resp_p:22`
- `rule.groups:zeek and data.auth_success:*` (Zeek's SSH session records)

**What each tool saw:**
| Tool | Result |
|------|--------|
| Suricata | **0 alerts** — the brute force was invisible to it |
| Zeek | **52 SSH sessions** recorded in `ssh.log` (51 attempts + 1 manual test login), each with an `auth_success` field |
| Wazuh | 52 events, all level 3, nothing flagged as a brute force |

<!-- 📸 **Screenshot:** The brute-force loop running on Kali (FAIL list ending in SUCCESS: msfadmin)-->
<!-- 📸 **Screenshot:** `rule.groups:suricata` showing "No results" during the attack window-->
<!-- 📸 **Screenshot:** `rule.groups:zeek and data.auth_success:*` showing 52 SSH sessions-->

### 5.3 Scenario 3: FTP login attempts
**MITRE ATT&CK:** T1110.001 Brute Force: Password Guessing

**What I did:**
Same 51-password list as the SSH test, this time against FTP.

```bash
date
hydra -l msfadmin -P ~/pw.txt 10.10.10.20 ftp -t 4
date
```

| Run | Start | End | Attempts | Duration |
|-----|-------|-----|----------|----------|
| FTP brute force | 9:09:25 PM | 9:10:10 PM | 51 | ~45 sec |

Hydra found the valid password (`msfadmin`) and ran far faster than SSH (45 seconds vs 7 minutes), because there's no slow handshake.

**Searches used in Wazuh:**
- `rule.groups:suricata`
- `rule.groups:zeek and data.id_resp_p:21`
- `rule.groups:zeek and data.user:msfadmin`

I also read Zeek's FTP log directly on the sensor, which turned out to be the most revealing source:
```bash
sudo cat /opt/zeek/logs/current/ftp.log | tail -n 20
```

**What each tool saw:**
| Tool | Result |
|------|--------|
| Suricata | **0 alerts** — nothing, even on the fast 45-second burst (reran to confirm) |
| Zeek (`ftp.log`) | Full plain-text record: username `msfadmin`, every `USER`/`PASS` command, and the server's `530 Login incorrect` reply for each failure |
| Wazuh | 19 connections to port 21 (rule 100200, level 3). The FTP username **is** captured and searchable (`data.user:msfadmin`), but every event arrives at flat level 3 with no brute-force alert by default |

<!-- 📸 **Screenshot:** Hydra finding `msfadmin:msfadmin` on FTP-->
<!-- 📸 **Screenshot:** `sudo cat ftp.log` showing the plain-text USER/PASS commands and `530 Login incorrect` replies-->
<!-- 📸 **Screenshot:** `rule.groups:suricata` showing "No results" during the FTP attack-->

---

## 6. Results / Findings

### Detection comparison
| Scenario | Suricata | Zeek | Wazuh dashboard | Detected? |
|----------|----------|------|-----------------|-----------|
| 1. Nmap recon | ~9 alerts per scan, all `ET SCAN Possible Nmap User-Agent` (level 3, no MITRE mapping) | ~1,100 records per scan, mostly refused connections (`REJ`) | Visible as a big spike in Zeek events, but nothing labels it as a scan | Partially |
| 2. SSH login attempts | **Nothing by default** (rule exists; fires once `HOME_NET` is fixed — see Root cause) | 52 SSH sessions in `ssh.log`, each with an inferred `auth_success` value | 52 low-level (3) events, no brute-force alert, no correlation | Only in raw Zeek data |
| 3. FTP login attempts | **Nothing by default** (fires once `HOME_NET` is fixed — see Root cause) | Full plain-text capture in `ftp.log`: username, commands, and `530 Login incorrect` replies | Username searchable (`data.user`), but all events flat at level 3 |  Visible, but not prioritized |

### Notes

**Scenario 1 (Nmap recon)**
- **Suricata only caught the scan because of `-sV`.** Every alert was `ET SCAN Possible Nmap User-Agent Observed`, which fires when Nmap's service detection sends web requests with "Nmap" in the user-agent. Nothing fired on the port scan itself (`-sS`). A quieter scan without `-sV` may not trigger any Suricata alert at all. That's a detection gap worth testing.
- **Zeek saw everything, but it's raw data.** About 1,000 refused connections from one source in a few seconds is a textbook port scan, but Zeek just records each one. You have to know to look for the pattern.
- **Wazuh treats it all as low priority.** Every Suricata and Zeek event came in at level 3 with no MITRE ATT&CK tag, the same level as routine events like a `sudo` login. Nothing on the dashboard says "port scan detected."

<!-- 📸 **Screenshot:** `rule.groups:suricata` events list showing the `ET SCAN` alerts-->
<!-- 📸 **Screenshot:** `rule.groups:zeek and data.conn_state:REJ` showing ~2,900 refused connections as spikes-->
<!-- 📸 **Screenshot:** `rule.groups:zeek` events list showing Kali → Metasploitable on many different ports-->

**Scenario 2 (SSH brute force)**
- **Suricata missed it — but not for the reason I first assumed.** My first guess was timing: the slow SSH handshake spread the attempts out over ~7 minutes, so maybe they fell under a rate threshold. That turned out to be wrong (FTP missed too, even as a fast burst).
- **Zeek caught all of it, and even guessed the outcome.** Zeek's SSH analyzer records every session in `ssh.log`, and its `auth_success` field infers whether each login worked by watching the encrypted traffic pattern (packet sizes and timing), without being able to read the password. 
- **Wazuh showed 52 flat, low-priority events.** Same as Scenario 1: everything came in at level 3 with no correlation, so a clear brute force looks identical to normal traffic on the dashboard.

**Scenario 3 (FTP brute force)**
- **Suricata missed it again — same root cause as SSH.** I reran FTP as a fast 45-second burst to rule out timing, and it still fired nothings. Reading the Emerging Threats rules confirmed the they exist and work, but expect an external attacker, which Kali was not considered.
- **FTP gives up everything in plain text.** Zeek's `ftp.log` recorded the username (`msfadmin`), every `USER` and `PASS` command, and — the key part — the server's actual `530 Login incorrect` reply for each failed attempt. Unlike SSH, where Zeek had to *infer* success from traffic patterns, FTP's failures are stated in clear text on the wire.
- **Passwords were redacted by default.** The `password` field showed `<hidden>` — Zeek masks FTP passwords unless `FTP::default_capture_password` is enabled. So Zeek saw the usernames and every attempt, but protects the password values by default.
- **SIEM did not assign priority well.** The FTP username and commands do reach Wazuh and are searchable (`data.user:msfadmin`). What's missing is any sense of priority: the 19 connection events all sit at level 3, the same as a routine login, so nothing says "brute force." 

### Overall analysis
Across the three scenarios, no single tool caught everything, and the default configuration caught very little:

| Attack | Suricata (signatures) | Zeek (metadata) |
|--------|----------------------|-----------------|
| Nmap scan | ✅ Caught it | ✅ Logged it (but as 1000s of raw records) |
| SSH brute force | ❌ Missed* | ✅ Logged + inferred success |
| FTP brute force | ❌ Missed* | ✅ Logged in full plain text |

\* Missed with the default `HOME_NET`. Both are caught once `HOME_NET` excludes the attacker — see Root cause below.

- **Signatures catch known tool fingerprints, not behavior.** Suricata flagged Nmap because of its recognizable user-agent, but had no answer for "the same source tried to log in 51 times." Brute force is a *pattern over time*, which signature rules aren't built to see.
- **Zeek sees everything but judges nothing.** It recorded every attack completely, but at a flat priority. The evidence is all there; turning it into an alert is the missing step.
- **A default SIEM is not a working SIEM.** Out of the box, every event landed at level 3 with no correlation, so a port scan, a brute force, and a routine login all look identical on the dashboard. The pipeline collects data well; it doesn't yet *detect*.

### Root cause: why Suricata missed the brute forces
Reading the actual Suricata rules on the sensor explained it. The SSH and FTP brute-force rules all take this shape:

```
alert tcp $EXTERNAL_NET any -> $HOME_NET 22 (... threshold: count 5, seconds 120 ...)
```

They only fire for traffic from `$EXTERNAL_NET` (outside) into `$HOME_NET` (what you're protecting). In Lab 01 I set `HOME_NET` to `10.10.10.0/24`, which covers **both** Kali and Metasploitable. So Suricata saw the attack as inside → inside, and the rule's direction never matched. The threshold (5 attempts in 120 seconds) was met easily; direction was the only thing wrong.

**The fix:** I changed `HOME_NET` to just the target (`10.10.10.20/32`) so Kali became "external," restarted Suricata, and re-ran the FTP brute force. This time it alerted — *ET SCAN Potential FTP Brute-Force attempt response* (8 alerts). SSH traffic was caught now too. 

**The lesson:** default IDS rules are written assuming threats come from outside. An attacker already inside the trusted network is not flagged.

---

## 7. Troubleshooting
| Problem | Cause | Fix |
|---------|-------|-----|
| Events from a scan didn't appear in Wazuh | The Wazuh agent lost track of the log files after an hourly Zeek rotation and a reboot | Restarted `wazuh-agent`; reran the attack so new log lines were captured live |
| Hydra couldn't connect to SSH | Metasploitable's 2009 SSH server only offers legacy crypto that modern Kali has removed | Brute-forced with Kali's own `ssh` client instead, re-enabling the old algorithms in `~/.ssh/config` |
| Hydra SSH bypass would not work | Kali is on the isolated lab network with no internet to reach package repos | Used SSH's built-in `SSH_ASKPASS` to feed passwords, no install needed |

---

## 8. Lessons Learned
- **No single tool is enough.** Suricata caught the scan and missed both brute forces; Zeek caught both brute forces and buried the scan in noise. Defense in depth covered all three instances.
- **Default detection is not very strong.** A fresh Wazuh install collects data well but barely alerts. You need to add your own rules and tune them.
- **Look for IoC in multiple places.** The SIEM showed 19 events and the sensor's `ftp.log` showed the whole attack in plain text. Look in logs and the SIEM to see the full picture.

---

## 9. Next Steps
- **Lab 03 — Detection engineering and tuning.** Use the gaps found here to write custom Wazuh rules that actually alert:
  - Port scan: fire when one source causes many `REJ` (refused) connections in a short window.
  - SSH brute force: fire on many SSH sessions from one source in a short window.
  - FTP brute force: fire on repeated `530 Login incorrect` replies from one source.
- Raise the alert level and add MITRE tags so these stand out instead of sitting at level 3.
- Optionally enable `FTP::default_capture_password` in Zeek to capture attempted passwords, and tune the noisy generic Zeek rule (100200) down.
