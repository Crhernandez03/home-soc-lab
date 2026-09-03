# Home SOC Lab — Threat Detection & Attack Simulation

![Wazuh](https://img.shields.io/badge/SIEM-Wazuh%204.7.3-blue)
![Docker](https://img.shields.io/badge/Container-Docker-2496ED)
![MITRE ATT%26CK](https://img.shields.io/badge/Mapped-MITRE%20ATT%26CK-red)
![Status](https://img.shields.io/badge/Status-Active-brightgreen)

A home-built Security Operations Center lab for practicing blue team
detection, monitoring, and incident response against simulated attacks.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [What it does](#what-it-does)
- [Stack](#stack)
- [Getting Started](#getting-started)
- [Screenshots](#screenshots)
- [Status](#status)
- [Repo contents](#repo-contents)
- [Why this project](#why-this-project)

## Overview

This lab pairs a monitored "defender" host with an isolated "attacker"
host to generate real attack traffic and practice detecting it end to
end — from raw logs, to SIEM alert, to MITRE ATT&CK technique mapping.

## Architecture

```
┌─────────────────────────────┐        ┌─────────────────────────────┐
│   Ubuntu Server 24.04 LTS   │        │      Kali Linux 2026.1       │
│         (Defender)          │        │         (Attacker)          │
│                              │        │                              │
│  Docker + Portainer CE      │◄──────►│  Nmap recon scans           │
│  Wazuh Manager / Indexer /  │  NAT   │  Hydra SSH brute-force      │
│  Dashboard                  │network │  attempts                    │
│  Wazuh Agent (host)         │        │  Wazuh Agent (host)          │
└─────────────────────────────┘        └─────────────────────────────┘
        Hypervisor: VMware Workstation Pro
        Host: Ryzen 7 5800X / RTX 5070 / 32GB DDR4
        Remote access: Tailscale (SSH + Wazuh dashboard from any device)
```

## What it does

- **Detection & alerting** — Wazuh ingests logs from both the defender
  and attacker hosts, alerting on suspicious activity such as repeated
  failed SSH logins and port scan patterns.
- **ATT&CK mapping** — Alerts are automatically mapped to MITRE
  ATT&CK techniques (e.g. T1110 Brute Force, T1046 Network Service
  Discovery) inside the Wazuh dashboard.
- **Attack simulation** — The Kali VM runs realistic reconnaissance
  and brute-force attacks against the defender host to generate
  detectable activity, rather than relying on synthetic test data.
- **Active Response** — Confirmed attacks automatically trigger
  OS-level blocking of the attacker's IP on the defender host.
- **Remote access** — Tailscale gives secure access to both VMs and
  the Wazuh dashboard from any device, without exposing anything to
  the public internet.

## Stack

| Component         | Tool                                      |
| ------------------ | ------------------------------------------ |
| Hypervisor         | VMware Workstation Pro                    |
| Defender OS        | Ubuntu Server 24.04 LTS                   |
| Attacker OS         | Kali Linux 2026.1                         |
| SIEM               | Wazuh 4.7.3 (manager, indexer, dashboard) |
| Container runtime  | Docker + Portainer CE                     |
| Attack tools        | Nmap, Hydra                               |
| Remote access       | Tailscale                                 |

## Getting Started

This repo documents a working lab rather than a one-click deploy, but
the core pieces can be reproduced:

1. **Provision the two VMs** — Ubuntu Server 24.04 LTS (defender) and
   Kali Linux (attacker) on the same isolated NAT network in your
   hypervisor of choice.
2. **Deploy the Wazuh stack** on the defender host using the sanitized
   [`docker-compose.yml`](docker-compose.yml) in this repo as a
   reference (replace the placeholder IPs/secrets with your own).
3. **Install a Wazuh agent** on both the defender host and the Kali
   attacker VM, pointed at the manager's internal NAT IP.
4. **Generate attack traffic** from Kali using Nmap (recon) and Hydra
   (SSH brute-force) against the defender host.
5. **Watch it get detected** in the Wazuh dashboard — alerts, MITRE
   ATT&CK mapping, and Active Response blocking all fire from real
   traffic, not synthetic test data.

See [`docs/active-response-debugging.md`](docs/active-response-debugging.md)
for a full write-up of the trickiest issue in the build (Active
Response silently failing) and how it was diagnosed and fixed.

## Screenshots

**Dashboard overview** — Wazuh detecting a simulated SSH brute-force attack: 424 total alerts, 368 authentication failures, and MITRE ATT&CK technique breakdown (Password Guessing, SSH, Brute Force).

[![Wazuh dashboard overview](https://github.com/Crhernandez03/home-soc-lab/raw/main/docs/wazuh-dashboard-overview.png)](/Crhernandez03/home-soc-lab/blob/main/docs/wazuh-dashboard-overview.png)

**Alert detail table** — Individual alerts showing technique mapping across the attack, including T1110 (Password Guessing), T1110.001, and T1021.004 (Remote Services / Lateral Movement).

[![Wazuh alert table](https://github.com/Crhernandez03/home-soc-lab/raw/main/docs/wazuh-alert-table.png)](/Crhernandez03/home-soc-lab/blob/main/docs/wazuh-alert-table.png)

### Active Response — Automatic Attacker Blocking

Wazuh detects the SSH brute-force attempt (rule 5763, MITRE T1110) and
automatically triggers the `host-deny` active response, blocking the
attacker's IP at the OS level. Confirmed both in the active-response log
(rule fires → host-deny executes) and in `/etc/hosts.deny` (`ALL:192.168.94.130`).

Full debugging write-up: [`docs/active-response-debugging.md`](docs/active-response-debugging.md)

[![Active Response Proof](https://github.com/Crhernandez03/home-soc-lab/raw/main/docs/active-response-proof.png)](/Crhernandez03/home-soc-lab/blob/main/docs/active-response-proof.png)

### Kali Attacker VM as a Monitored Agent

The Kali attacker VM now runs its own Wazuh agent, in addition to acting as
the attack source. This gives visibility into both sides of the simulations
— attacker-side activity and defender-side detections — not just the
defender host.

[![Wazuh Agents List](https://github.com/Crhernandez03/home-soc-lab/raw/main/docs/kali-agent-registered.png)](/Crhernandez03/home-soc-lab/blob/main/docs/kali-agent-registered.png)

## Status

### Completed

- [x] Deploy Wazuh stack (manager, indexer, dashboard) via Docker on the defender host
- [x] Install a Wazuh agent on the defender host and confirm log ingestion
- [x] Simulate SSH brute-force and Nmap recon attacks from Kali and confirm detection + MITRE ATT&CK mapping
- [x] Fix Wazuh Active Response (automatic attacker IP blocking) —
      manager config was missing the active-response block, and a
      file-ownership mismatch was silently breaking wazuh-db
- [x] Add a Wazuh agent on the Kali attacker VM — installed, registered
      against the manager, confirmed active in the dashboard (100% agent coverage)
- [x] Set up remote access via Tailscale (SSH + Wazuh dashboard from any device)

### Planned

- [ ] Explore the MITRE ATT&CK dashboard module further
- [ ] Stand up a Metasploitable target VM
- [ ] Run additional attack scenarios: privilege escalation, lateral movement, persistence

## Repo contents

- `docker-compose.yml` — sanitized Wazuh stack configuration used on
  the defender VM (internal IPs and secrets removed/replaced with
  placeholders — see comments)
- `docs/` — write-ups and screenshots of alerts, ATT&CK mappings, and
  attack walkthroughs (added as the lab progresses)
- `LICENSE` — MIT license

## Why this project

Built to get hands-on, practical experience with the core blue team
workflow — deploying a SIEM, generating real attack traffic, tuning
detections, and mapping activity to a recognized threat framework —
as part of my path toward a SOC analyst role.
