## 2026-08-18 — FileDeleteDetected (Event 26) config attempt — unresolved, reverted

**Phase:**       C (Detection Engineering)
**Goal:**        Enable Sysmon Event ID 26 (FileDeleteDetected) on Win11-LTSC-Victim to close the
                  known FileDelete telemetry gap and unblock future T1070.004-family detection work.
**Transcript:**  Long session, heavy diagnostic detour.

### What happened
Added a new `<RuleGroup name="FileDeleteDetection" groupRelation="or">` containing an empty
`<FileDeleteDetected onmatch="exclude">` block to `sysmonconfig-export.xml`, placed immediately
after the existing (commented-out) Event 23 FileDelete section. Verified via schema dump
(`Sysmon64.exe -s`) that `FileDeleteDetected` (event value 26) is the correct, version-confirmed
element name for this build (v15.21, schema 4.91). XML validated cleanly via PowerShell's `[xml]`
parser and reloaded with no parse errors (`Sysmon64.exe -c`).

**Verification never succeeded.** Across two full VM reboots (required since Sysmon blocks
standard service-stop requests as tamper protection — confirmed expected behavior, not a bug),
Event ID 26 never appeared in the Windows Event Log for a deliberately generated test file
deletion, and `FileDeleteDetected`/`FileDelete` never appeared in the live `-c` config dump —
even after renaming the `RuleGroup` from an empty `name=""` to `name="FileDeleteDetection"` on
the theory that an unnamed rule group might be silently dropped.

**Root cause not isolated.** Substantial diagnostic effort went into what turned out to be a red
herring: PowerShell's console/pipeline handling of this Sysmon binary's output introduced null-byte
corruption and premature pipeline truncation (`Select-Object -First`/`-Path`-based `Select-String`
searches against captured Sysmon output were unreliable all session), which repeatedly produced
false "not found" results and cost significant time before being identified as a diagnostic-tooling
issue rather than a config issue. Once ruled out (via `notepad`'s Ctrl+F on a clean file), the
absence of Event 26 was confirmed genuine — the config change is not taking effect in the live
driver, for a reason not yet identified.

**Reverted for cleanliness.** The `RuleGroup` block was wrapped in a comment (matching the existing
commented-out Event 23 section's style) and the config reloaded clean, so the running config now
matches the file with no orphaned/half-working rule group in place.

### Outcome
- **FileDelete/FileDeleteDetected telemetry gap remains open** — file-deletion-based detections
  (T1070.004 and similar) are still not possible on Win11-LTSC-Victim.
- Config file is back in a clean, consistent state (Event 26 section commented out, matching Event 23).
- No lasting change to the VM's detection posture from tonight's session.

### Next-steps (logged, not built)
- Investigate whether PPL/tamper-protection or Defender is silently blocking Event 26 registration,
  similar to the LSASS PPL interaction hit during the T1003.001 session — worth checking
  `Get-MpThreatDetection` and Defender exclusions specifically for this event type before the next attempt.
- Consider testing Event 23 (FileDelete, full archive) instead, to see if the same failure occurs —
  would help isolate whether this is FileDelete-family-specific or a broader Sysmon driver issue on this VM.
- Consider a from-scratch reinstall of Sysmon (`-u` then `-i -c <config>`) rather than a `-c` reload,
  in case the driver itself has a stale registration that a simple reload/reboot doesn't clear.

### Lessons
1. **PowerShell's handling of native console output from this specific Sysmon binary is unreliable
   in this environment** — `Select-String -Path`, `Select-Object -First` piped directly onto a live
   external command, and even `[Console]::OutputEncoding` mismatches all produced silent false
   negatives tonight. When search tools give suspiciously consistent "nothing found" results against
   data you can visually confirm exists, verify with the simplest possible method (`.Contains()` on a
   directly-loaded string, or just `notepad` + Ctrl+F) before trusting the search tool's output.
2. **An empty `name=""` attribute on a Sysmon `RuleGroup` was a reasonable, testable hypothesis but
   was not the actual cause** — worth remembering so it isn't re-tried as a fix next time without new
   evidence.
3. **This is a genuine, currently-unresolved gap** — not a documentation or config-syntax problem.
   Future sessions should start from Defender/PPL interaction as the next hypothesis, not repeat
   tonight's XML/encoding diagnostics.

---

## 2026-08-18 — Phase C.6 kickoff: Kali moved to RANGE30

**Phase:**       C.6 (Attack Surface & AD Expansion) — infra hardening, sub-step 1 of 3
**Goal:**        Move Kali from flat `vmbr1` to RANGE30, isolating it under the same model already
                  proven for Win11-LTSC-Victim, as the first of three infra-hardening tasks that
                  precede the AD/`dc01` build.
**Transcript:**  Short session, GUI-only network change, verified via CLI from inside Kali.

### What happened
- Confirmed baseline state before changing anything: Kali's `net0` network device was on `vmbr1`,
  no VLAN tag set.
- In the Proxmox web UI, changed Kali's `net0` to bridge `vmbr3` (the VLAN-aware trunk carrying
  VLANs 10/20/30), VLAN Tag `30`. Firewall checkbox left unchecked, consistent with the existing
  pattern for other Range VMs — VLAN-level isolation is enforced by pfSense's rules, not per-VM
  Proxmox firewalling.
- Rebooted Kali via Proxmox's own Reboot control (not a guest-OS-level restart), to force the
  hypervisor-side NIC replug and a clean DHCP renegotiation on the new segment.
- First post-reboot `ip addr show` check appeared to show no IP address at all despite a default
  route already being present — this turned out to be a timing gap, not a real failure. DHCP
  completed moments later. Confirmed clean via three separate checks: `ip route show` (default via
  `10.10.30.1` on `eth0`), `ip addr show` (`eth0` holding `10.10.30.101/24`, `dynamic`, no stale
  static config), and `nmcli device status` (single normal `eth0` connection, no leftover profile
  conflict from the old `vmbr1` config).

### Acceptance check
Three pings run from inside Kali post-move:
- `8.8.8.8` (internet) — 100% loss ✅ expected; RANGE30 has no outbound internet by design.
- `192.168.0.1` (home network / MGMT gateway) — 100% loss ✅ expected; RANGE30 cannot reach MGMT
  or the home LAN.
- `10.10.20.100` (`wazuh-host`, INFRA20) — 100% loss ✅ expected; ICMP is not one of RANGE30's three
  explicit allow rules (Pass 1515, Pass 1514, default-deny), so this correctly fails even though
  Kali can still reach `wazuh-host` on the two allowed Wazuh ports specifically.

### Outcome
Kali is confirmed isolated on RANGE30 — same live-proven isolation model as Win11-LTSC-Victim.
This closes the "move Kali to RANGE30" item of Phase C.6 step 1 (infra hardening). Two
infra-hardening items remain before the AD/`dc01` build proper starts:
- pfSense log forwarding into Wazuh (firewall pass/block, DHCP, DNS)
- WireGuard remote-access VPN configuration

### Next-steps (logged, not built)
- pfSense log forwarding into Wazuh — planned as syslog from pfSense to the Wazuh manager
  (pfSense has no native Wazuh agent build; FreeBSD-based), scoped to accept only from pfSense's
  INFRA20 IP.
- WireGuard remote-access VPN — core security logic, reference drafted by Claude, final written
  and applied manually.

### Lessons
1. **A "no IP yet" reading immediately after a VM reboot/NIC change can be a timing artifact, not
   a real failure** — DHCP negotiation on Debian/Kali's NetworkManager stack can lag a few seconds
   behind the interface coming up. Worth a second check before troubleshooting a "failure" that
   may just need a moment to complete.
2. **The three-ping acceptance pattern (internet / home-MGMT / INFRA20-ICMP) used for
   Win11-LTSC-Victim in Phase B applies cleanly to any new Range VM** — a reusable acceptance
   check for future RANGE30 isolation moves (`win11-ws02`, `dc01`, `linux-victim` will each want
   the same three-ping confirmation once built).
