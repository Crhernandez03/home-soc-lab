# Debugging Wazuh Active Response: Two Stacked Bugs

## The goal

Get Wazuh to automatically block an attacker's IP at the OS level
(via the built-in `host-deny` active response) the moment it detects
a brute-force pattern — without any manual intervention.

## The symptom

A live Hydra SSH brute-force run from the Kali attacker VM was
generating alerts correctly (rule 5763, MITRE ATT&CK T1110 — Brute
Force) and showing up in the Wazuh dashboard. But the attacker's IP
was never actually being blocked. No entry in `/etc/hosts.deny`, no
active-response log entries — the response simply wasn't firing.

## Bug #1: the active-response block was missing from the running config

The manager's `ossec.conf` — the file that defines *which* rules
trigger *which* active responses — didn't have an `<active-response>`
block defined at all. Without it, Wazuh has no instruction to run
`host-deny` (or anything else) when rule 5763 fires, regardless of
how correctly the alert itself is generated.

**The catch:** editing the config *inside* the running manager
container doesn't actually persist or take effect the way you'd
expect, because the real, authoritative config lives on the host
filesystem and is bind-mounted into the container — in this setup, at:

```
~/wazuh-docker/single-node/config/wazuh_cluster/wazuh_manager.conf
```

The copy inside the container is just a view of that mounted file.
Editing a copy inside the container (or assuming the container's
internal `/var/ossec/etc/ossec.conf` is the source of truth) means
changes get silently lost on the next restart, or never take effect
because the wrong file was edited in the first place.

**Fix:** add the `<active-response>` block directly to the host-side
`wazuh_manager.conf`, restart the manager container so it picks up
the mounted config, and confirm the block is present via
`docker exec <manager> cat /var/ossec/etc/ossec.conf`.

## Bug #2: wazuh-db was silently failing due to a UID mismatch

Even after the config fix, active response still wasn't firing
reliably. The root cause was a second, unrelated problem: `wazuh-db`
(the internal Wazuh service that active response depends on) was
failing to start.

The cause was a file ownership mismatch: `ossec.conf` on disk was
owned by UID `1000` (the default first non-root user on the Docker
host), but the Wazuh manager container runs its processes as the
`wazuh` user, which is UID `101` inside the container. From the
container's point of view, it didn't have permission to properly read
the file it needed, and `wazuh-db` failed to start — without an
obvious error pointing directly at "fix this file's ownership."

**Fix:** `chown` the config file (and surrounding config directory) on
the host to UID `101` so the containerized `wazuh` user has proper
read access, then restart the stack.

## Verification

With both fixes in place, a repeat Hydra brute-force run against the
defender host:

1. Triggered rule 5763 (MITRE T1110) as expected.
2. Fired the `host-deny` active response — confirmed in
   `active-responses.log` showing the rule match and the script
   execution.
3. Actually blocked the attacker at the OS level — confirmed with
   `ALL:192.168.94.130` appearing in `/etc/hosts.deny` on the defender
   host.

## Takeaways

- **Docker bind mounts mean the container's filesystem view isn't
  always the source of truth.** When a config change "doesn't take,"
  check whether you're editing the mounted host file or a copy inside
  the container.
- **Permission errors in containerized services aren't always loud.**
  A UID mismatch between the host and the container's internal user
  can cause a dependent service to fail without an error that points
  directly at "this is a file permissions problem."
- **Test the full loop, not just the alert.** An alert firing
  correctly in the dashboard doesn't guarantee the response tied to
  it is actually executing — verify both the trigger and the action.
