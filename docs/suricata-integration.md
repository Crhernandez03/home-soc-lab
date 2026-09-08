# Closing a Network-Layer Blind Spot — Adding Suricata as a NIDS

## Summary
While exploring Wazuh's MITRE ATT&CK module to see how well the lab mapped attacker
techniques, I discovered a real detection gap: a live `nmap` scan against the Ubuntu
Server produced **zero** Wazuh alerts. Root cause and fix are documented below,
verified end-to-end with a before/after test.

## Problem: a full nmap scan generated no alerts at all
Ran the following from the Kali attacker VM against the Ubuntu Server:

```bash
nmap -sV -sC 192.168.94.129
```

The scan completed normally and returned service/version data (including fingerprinting
the Wazuh dashboard on 8443 and the OpenSearch API on 9200). But checking both the
MITRE ATT&CK → Events tab and the general Security Events view in Wazuh, filtered to
the Ubuntu Server agent, showed nothing new for the scan window.

### Root cause
Wazuh, as deployed in this lab, is a **HIDS** — a Host-based Intrusion Detection
System. The agent works by tailing log files on the endpoint (`auth.log`, PAM logs,
syslog, etc.) and matching new lines against rules. Every alert the lab had generated
up to this point (SSH brute-force, failed logins) happened because those actions wrote
a line to a log file.

A port scan doesn't do that. `nmap` just opens and closes TCP connections against
various ports — no failed login, no PAM event, nothing written to any log Wazuh was
watching. From a log-based system's perspective, nothing happened. Catching this class
of activity requires inspecting raw network traffic — a **NIDS** (Network Intrusion
Detection System) — not just log files.

## Fix: install and integrate Suricata
All work done directly on the Ubuntu Server VM (the defender), no new VM required.

1. **Installed Suricata:**
   ```bash
   sudo apt update && sudo apt install -y suricata jq
   ```

2. **Identified the correct network interface** (`ip a`) — `ens33`, the interface
   carrying the lab's `192.168.94.129` address (as opposed to `tailscale0`, `docker0`,
   or the various Docker `veth`/bridge interfaces also present on the host).

3. **Configured `/etc/suricata/suricata.yaml`:**
   - `HOME_NET: "[192.168.94.0/24]"` — scoped to the actual lab subnet instead of the
     default broad ranges.
   - `af-packet → interface: ens33` — pointed capture at the correct NIC.

4. **Pulled the detection ruleset:**
   ```bash
   sudo suricata-update
   ```
   Loaded the Emerging Threats Open ruleset — 68,626 rules total, 52,675 enabled after
   Suricata's own filtering. Config validated automatically (`suricata -T`) as part of
   the update.

5. **Started the service:**
   ```bash
   sudo systemctl enable --now suricata
   ```
   Confirmed `active (running)` via `systemctl status suricata`, and confirmed live
   traffic logging by tailing `/var/log/suricata/eve.json`.

6. **Wired Suricata's output into the Wazuh agent** by adding a `<localfile>` block to
   `/var/ossec/etc/ossec.conf` (the host agent config, not the manager container):
   ```xml
   <localfile>
     <log_format>json</log_format>
     <location>/var/log/suricata/eve.json</location>
   </localfile>
   ```
   Restarted the agent: `sudo systemctl restart wazuh-agent`.

No custom decoder or rule was needed — Wazuh ships built-in Suricata support out of the
box (`0475-suricata_rules.xml`, generic passthrough alert rule ID **86601**).

## Verification — before/after with the same test
Re-ran the identical scan from Kali: `nmap -sV -sC 192.168.94.129`.

**This time:**
- `eve.json` immediately logged repeated `ET SCAN Possible Nmap User-Agent Observed`
  alerts — Suricata correctly fingerprinted Nmap's scripting-engine `User-Agent` string
  in the HTTP requests it sent probing a service on port 8000.
- Those alerts flowed into the Wazuh dashboard's Security Events view as
  `Suricata: Alert - ...` entries under rule ID 86601, agent `ubuntu-server-01`.
- A dashboard search for `suricata` returned 355 hits over the prior 24 hours —
  confirming continuous live ingestion, not a one-off catch.

| | Before | After |
|---|---|---|
| Port scan detection | None (0 alerts) | Detected — `ET SCAN` signature fires |
| Visibility type | Host-based only (log files) | Host-based + network-based (packet inspection) |
| Suricata alerts reaching Wazuh | N/A | Rule ID 86601, live |

## Lessons learned
- A HIDS and a NIDS answer fundamentally different questions. Wazuh's default agent
  setup excels at anything that shows up in a *log* (failed logins, sudo use, file
  integrity changes) but is structurally blind to raw network traffic that the OS never
  logs in the first place.
- Wazuh's Suricata integration is close to zero-config on the Wazuh side — nearly all
  of the setup work is in correctly pointing Suricata itself at the right interface and
  network range.
- Generic Suricata alerts (rule 86601) land at a flat, low severity (Level 3) with no
  MITRE ATT&CK tagging by default. A good follow-up: custom rules that raise severity
  for specific signature categories (e.g. `ET SCAN`) and add MITRE mappings, mirroring
  the custom `host-deny` active-response work done earlier in this build.

## Next steps
- Optional: custom rule to elevate scan-signature severity and add MITRE tagging
- Continue Module 4: stand up Metasploitable target, explore MITRE dashboard further,
  run additional attack scenarios (privilege escalation, lateral movement, persistence)
