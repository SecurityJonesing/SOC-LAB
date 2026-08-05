# build_log.md
### Home SOC Lab — running record

Every change, its **actual** output, the decision behind it, and every rollback point. Appended as it happens — never reconstructed afterward.

**Why this file exists:** a previous attempt at Cisco + pfSense configuration (via a different AI tool) caused a lockout. There was no record of what had been changed, so there was no way to reverse it — which is what forced a full Proxmox reinstall. This file is the answer to "what exactly did we change?" before you need to ask it.

**What goes where:**
- **Raw terminal output** → session transcripts in `C:\Users\micha\SOC-Lab\Build-Transcripts\logs` (PuTTY logging, `Start-Transcript`, `script`). Everything, including typos and dead ends.
- **This file** → the narrative. What you tried, what actually happened, what you decided, and how to undo it. Readable six months from now.

**Rules:**
- Record **actual** output, not what you expected. "It worked" is not an entry.
- Log dead ends too. The thing that *didn't* work is often the most useful line in the file.
- Every rollback point gets recorded when it's created, not when you need it.
- Append as you go. If you're writing it up at the end of the session, it's already too late to be accurate.

**Entry format:**

```
## YYYY-MM-DD — <what changed>
**Phase:**       <A / A.5 / B / ... / pre-A>
**Goal:**        <one line>
**Rollback:**    <snapshot name, config export, or "none — read-only">
**Transcript:**  <log filename>

### What happened
<commands, actual output, decisions>

### Outcome
<current state after this change>

### Lessons
<anything that should go in Project instructions as a gotcha>
```

---

## 2026-07-16 — iDRAC recovery

**Phase:** pre-A (Phase A gate: console access must be confirmed)
**Goal:** Regain iDRAC access on `pve01`. Password unknown; no recovery path documented.
**Rollback:** None needed — iDRAC config reset doesn't touch Proxmox or VMs.
**Transcript:** *(not captured — this predates the session-recording rule)*

### What happened

**Attempt 1 — `racadm` from the Proxmox shell.** Considered resetting the iDRAC password from inside the running OS without a reboot. Abandoned: local RACADM needs Dell's OMSA / iDRAC Service Module, which isn't reliably available on Proxmox (Debian isn't a Dell-supported OS). Not worth the rabbit hole for a password reset.

**Attempt 2 — iDRAC web virtual console.** Downloading the console produced a `viewer.jnlp` file. Windows had no handler for it. Root cause: iDRAC6 uses Java Web Start, which Oracle removed from the JRE at Java 11. Would have required installing legacy Java 8 or IcedTea-Web. Abandoned as not worth it.

**Attempt 3 — BIOS (F2) → iDRAC Settings.** No iDRAC option present in System Setup. Root cause: on the R710, iDRAC6 is **not** inside the F2 menu — it has its own entry point.

**What actually worked — `Ctrl+E` during POST.** The R710 shows a five-second window at POST offering "Remote Access Setup". That opens the iDRAC6 Configuration Utility, which is separate from F2's System Setup. Reset iDRAC configuration to defaults from there.

**After the reset:**
- Set iDRAC to DHCP; created a DHCP reservation on the home router → `192.168.0.100`
- Logged in with factory defaults (`root` / `calvin`)
- **Changed the password immediately** — a freshly-reset iDRAC on factory credentials is an open door on the network
- Recorded IP + username + password together in one place

### Outcome
iDRAC confirmed reachable and working at **`192.168.0.100`**. Console access to `pve01` now exists independently of the network config — which is the prerequisite for touching anything network-related.

### Lessons
1. **iDRAC6 config is `Ctrl+E` during POST, NOT F2.** Five-second window, easy to miss. Separate menu from System Setup.
2. **The R710 is legacy BIOS only** — predates UEFI on PowerEdge. Don't look for UEFI options.
3. **iDRAC6's virtual console needs legacy Java Web Start** and modern Windows can't run it. **Monitor + keyboard plugged directly into the server is faster and always works** — prefer it over fighting Java.
4. Resetting iDRAC also clears its network config, not just the password.

---

## 2026-07-16 — Management NIC move: NIC2 → NIC1

**Phase:** pre-A (frees NIC2 to become pfSense's WAN in Phase A)
**Goal:** Move Proxmox host management from NIC2 to NIC1, so NIC2 is available for pfSense WAN.
**Rollback:** iDRAC (`192.168.0.100`) confirmed working first; monitor + keyboard on the server as backup. Both tested before starting.
**Transcript:** *(not captured — predates the session-recording rule)*

### What happened

**Created `vmbr1` on NIC1 (`eno1`) via the Proxmox GUI.** First attempt failed: Proxmox rejected the Create with *"default gateway already exists on interface vmbr0"*. Proxmox allows only one default gateway per host. Left the Gateway field blank — and the IP field blank too, deliberately, to avoid a duplicate-IP conflict before the cable had moved.

**Moved the physical cable from NIC2's port to NIC1's port.**

**`vmbr1` initially showed state DOWN.** Checked `ip link show` — `eno1` itself came up, and `vmbr1` followed once its member interface had carrier. Confirmed `eno1` was genuinely NIC1 rather than assuming the chassis label matched the Linux name.

**Assigned the IP via the local console** (monitor + keyboard, since neither bridge was reachable at that moment — `vmbr0` had the IP but no cable, `vmbr1` had the cable but no IP):
```
nano /etc/network/interfaces     # added address 192.168.0.201/24 + gateway to vmbr1
ifreload -a
ip addr show vmbr1               # confirmed 192.168.0.201/24 present
ip link show vmbr1               # confirmed UP
```

**Then: ping to 192.168.0.201 timed out anyway.** Everything looked correct — bridge UP, IP assigned, same subnet as the PC, cable in the same dumb switch.

**Root cause: `vmbr0` still held `192.168.0.201` as well.** The earlier edit to comment out its address hadn't taken. Two bridges on the same host claiming the same static IP creates ARP ambiguity — the kernel doesn't know which interface should answer, so replies get dropped or sent from the disconnected bridge. **Silent failure, no error message anywhere.**

**Fix:**
```
nano /etc/network/interfaces     # commented out vmbr0's address line
ip addr flush dev vmbr0          # cleared it at the kernel level immediately
ifreload -a
```
Ping replied. Proxmox web UI loaded at `192.168.0.201:8006`.

**Second problem, immediately after: Kali and Win11-LTSC-victim both reported "network unreachable."** Root cause: both VMs' network devices were still attached to `vmbr0` — which now had no IP *and* no cable. Moving the physical cable did not move the VMs.

**Fix:** shut down each VM → Hardware tab → Network Device → changed Bridge from `vmbr0` to `vmbr1` → booted. Both reached the internet again.

### Outcome
- **NIC1 / `vmbr1`** = Proxmox management, `192.168.0.201`. Confirmed working.
- **NIC2 / `vmbr0`** = no IP, no VMs attached. **Free — becomes pfSense WAN in Phase A.**
- Kali + Win11-LTSC-victim both on `vmbr1`, internet-reachable, **not yet isolated**.
- NIC3 (pfSense LAN trunk) and NIC4 (spare / SPAN fallback) unconfigured.

### Lessons
1. **Duplicate IP across bridges = silent ARP failure.** Proxmox permits two bridges to hold the same static IP even with one physically disconnected. No error, no warning, pings just vanish. **Always check `ip addr show` on BOTH bridges after any reassignment.** → now Project-instructions gotcha #1.
2. **Moving a cable or bridge orphans every VM still pointed at the old one.** The VM reports "network unreachable" with no obvious cause, because nothing about the VM changed. **Check every VM's Hardware tab after any bridge change.** → now Project-instructions gotcha #2.
3. Proxmox allows exactly one default gateway per host — a second bridge on the same subnet doesn't need its own.
4. Create the bridge with **no IP**, move the cable, *then* assign the IP. Ordering avoids the duplicate-IP window entirely.
5. Don't assume the chassis port label matches the Linux interface name. Confirm with `ip link show` / `ethtool <iface>` and watch which one loses carrier when you unplug it.
6. **Both of these were found by console access, not by SSH** — which is exactly the situation the Recovery Ladder exists for. The iDRAC recovery the day before is what made this change safe to attempt.

---

## 2026-07-17 — Identifying the Cisco switch

**Phase:** A
**Goal:** Identify the Cisco switch before writing any command for it — model, IOS version, real interface names.
**Rollback:** None needed — `show` commands are read-only.
**Transcript:** `C:\Users\micha\SOC-Lab\Build-Transcripts\logs` (PuTTY → Session → Logging → All session output, set BEFORE connecting)

### What happened
```
enable 
show version 
show interfaces status
```


### Outcome
Model: WS-C2960X-48FPS-L  IOS: 15.2(7)E9
Interface naming: GigabitEthernet1/0/1 through 1/0/52 (shorthand Gi1/0/1), plus a separate Fa0 management por
`switchport trunk encapsulation dot1q` supported? _______________
```
**Port summary** (`show interfaces status`, pre-config baseline):
- **Gi1/0/1, Gi1/0/48-52:** VLAN 1 (default)
- **Gi1/0/2-47:** VLAN 10 (leftover from a prior config — wiped in the factory reset below)
- **Gi1/0/49-50:** err-disabled (later confirmed hardware-faulty, see 2026-07-23 entry)
- **Gi1/0/51-52:** "Not Present" (unpopulated SFP/stacking ports)
- **All ports:** notconnect — nothing cabled yet; Fa0 (mgmt-only) also notconnect

Full raw output archived in the session transcript, not duplicated here.
```

### Lessons
*(none — this was a read-only identification step; see factory-reset entry below for the actual config change)*

---

## 2026-07-17 — Cisco switch factory reset

**Phase:** A
**Goal:** Wipe switch to factory defaults, confirm clean console access, before any VLAN/trunk config.
**Rollback:** None needed — this was an intentional, planned reset.
**Transcript:** switch-2026-07-17-*.log (PuTTY, COM14)

### What happened
- Identified switch first: WS-C2960X-48FPS-L, IOS 15.2(7)E9. Real interface
  naming confirmed: GigabitEthernet1/0/1–1/0/52, plus Fa0 (mgmt-only).
- Pre-reset `show interfaces status` showed prior config: hostname
  Lab-Switch, most ports in VLAN 10, ports Gi1/0/49-50 err-disabled.
- `write erase` completed cleanly.
- First `reload` attempt: answered [confirm] with "n" instead of Enter,
  which CANCELED the reload — NVRAM was erased but switch kept running
  old config in RAM. Caught this before assuming reload had happened.
- Re-ran `reload`, pressed Enter at [confirm] this time. Reboot completed.
  Declined initial config dialog (no).
- Session log has garbage lines from accidental clipboard paste (right-click
  in PuTTY pastes clipboard directly; a chat response was on the clipboard
  at the time) — switch correctly rejected all of it as invalid input, no
  actual config impact. Confirmed via `show run` below.
- Verified clean state: `enable` required no password, `show run` shows
  hostname "Switch" (default), no VLAN assignments, no err-disabled state,
  all 52 Gi ports at default, no aaa/line passwords configured.
-Ran `write memory` to save the clean state to NVRAM. Confirmed via
  `show config` — saved config matches the verified clean running-config
  exactly (hostname Switch, no VLANs, no passwords, all ports default).
- Powered off switch normally (no graceful-shutdown command exists on IOS;
  physical power-off at an idle prompt is safe).    
### Outcome
Switch factory reset confirmed clean, saved to NVRAM, and powered down
safely. Console access (PuTTY, COM14, 9600 8N1) confirmed working
post-reset. Ready for Phase A step 3 (pfSense VM build) and step 4 (VLAN
creation on switch) next session.

### Lessons
1. On Cisco, `[confirm]` prompts want Enter — any other keystroke (including
   "n") CANCELS the action rather than confirming "no". Different from a
   [yes/no] prompt, which does take a typed answer.
2. Right-click in PuTTY pastes the OS clipboard directly as keystrokes with
   no preview. Check clipboard contents before right-clicking, or use
   left-click-drag to select+copy just the intended text.

---

## 2026-07-21 — NIC troubleshooting + vmbr3 creation (Phase A step 3)

**Phase:** A
**Goal:** Identify NIC3 physically, create trunk-capable bridge for pfSense LAN.
**Rollback:** None needed — no destructive changes; only bridge creation (additive) and interface up/down toggles.
**Transcript:** session logs from today, pve01 console + SSH

### What happened
- Chassis port labels (`Gb1`-`Gb4`) confirmed NOT self-evident from `ip link show` alone —
  had to physically trace cables to labels. `Gb1` = `eno1`, confirmed via physical
  inspection AND behaviorally (unplugging it dropped the SSH/management session).
- `eno2`, `eno3`, `eno4` all showed `state DOWN` / `Link detected: no` across multiple
  cables, multiple switch ports, and even a direct PC-to-server connection — appeared
  to rule out cabling entirely.
- Extensive troubleshooting followed: `lspci`/`dmesg` (confirmed all 4 NICs detected
  cleanly by kernel, no driver errors), BIOS Integrated Devices check (all 4 ports
  shown Enabled), iDRAC Lifecycle Log/SEL review (no explicit NIC fault logged, though
  real historical CPU/memory/IO fault entries exist from Feb-Mar 2026 — unrelated to
  this issue as it turned out).
- **Root cause, finally found:** `eno2`/`eno3`/`eno4` were simply never administratively
  brought up (`ip link show` showed no `UP` flag, only `eno1` had it via vmbr1
  membership). Physical hardware was never at fault. Fix:
  ```
  ip link set eno2 up
  ip link set eno3 up
  ip link set eno4 up
  ```
  All three immediately showed `LOWER_UP` and clean `1000Mb/s Full duplex` link once
  brought up.
- Created `vmbr3` via Proxmox GUI (System > Network > Create > Linux Bridge):
  bridge-port `eno3`, no IP, VLAN-aware enabled (`bridge-vids 2-4094`). Applied via
  GUI "Apply Configuration" — took effect live, no reboot needed. Confirmed written
  to `/etc/network/interfaces` (persistent, not just runtime state).
- Naming: chose `vmbr3` (matching `eno3`/NIC3) instead of the "next sequential"
  `vmbr2`, as a deliberate one-off exception to keep NIC-to-bridge naming legible,
  despite `vmbr0`(NIC2)/`vmbr1`(NIC1) already being a mismatched, unfixable-without-risk
  precedent from earlier setup.

### Outcome
All 4 physical NICs confirmed healthy: `eno1` (Gb1, management, vmbr1),
`eno2` (Gb2, confirmed working, unused), `eno3` (Gb3, now `vmbr3`, pfSense LAN
trunk target), `eno4` (Gb4, confirmed working, spare/SPAN fallback per plan).
`vmbr3` created, VLAN-aware, ready for pfSense VM's LAN interface (net1) in the
next step.

### Lessons
1. **Check administrative interface state (`ip link set <iface> up`) BEFORE any
   hardware-level troubleshooting** (cabling, BIOS, iDRAC logs). A `DOWN` interface
   with no `UP` flag will show `Link detected: no` in `ethtool` even with perfectly
   healthy hardware and a good cable — this looks identical to a real hardware fault
   and cost significant time to diagnose here. This is now the FIRST thing to check
   on any "port shows no link" problem going forward.
2. Chassis port labels are not reliable without physical confirmation — verify by
   unplug/replug + `ip link show`, not by reading a photo or assuming left-to-right
   ordering matches `eno` numbering.
3. `vmbr` bridge names don't have to match NIC numbers, and neither convention is
   objectively correct — but once `vmbr0`/`vmbr1` exist with one convention, consider
   deciding on a naming scheme deliberately rather than defaulting to "next sequential
   number" without thinking about it.

---

## 2026-07-21 — vmbr0→vmbr2 swap, missing gateway fix, package upgrade + reboot

**Phase:** A
**Goal:** Rename NIC2's bridge to match NIC-number convention (vmbr0→vmbr2); discovered and fixed a missing default gateway in the process; upgraded and rebooted pve01.
**Rollback:** None needed — vmbr0 had no IP/no VMs attached (safe to delete); gateway addition tested live before persisting; package upgrade left prior kernels installed and selectable in GRUB as fallback.
**Transcript:** session logs from today, pve01 console + SSH

### What happened
- Deleted `vmbr0` via Proxmox GUI (System > Network > Remove > Apply Configuration).
  Confirmed removed both live (`ip addr show vmbr0` → "Device does not exist") and in
  `/etc/network/interfaces` (no vmbr0 stanza).
- Created `vmbr2` on `eno2` (System > Network > Create > Linux Bridge): no IP, VLAN-aware
  left UNCHECKED (this is pfSense's planned WAN side, not a VLAN trunk — only vmbr3/NIC3
  needs VLAN-aware). Confirmed UP, no IPv4, present in `/etc/network/interfaces`.
- While troubleshooting `apt update` failures (all repos returning "Network is
  unreachable"), found `ip route show` had no default route — only the local
  `192.168.0.0/24` subnet route via vmbr1. `/etc/network/interfaces` confirmed no
  `gateway` line existed anywhere, despite the 2026-07-16 log entry stating one was
  added to vmbr1 at that time. Root cause of the discrepancy not determined — either
  removed at some point after 07-16, or the original apply never fully completed.
- Before editing vmbr1 (live management interface): confirmed iDRAC (192.168.0.100)
  and monitor+keyboard fallback access, confirmed session recording on.
- Fix: added `gateway 192.168.0.1` line under vmbr1's `iface` stanza in
  `/etc/network/interfaces`, then `ifreload -a`. Verified live: `ip route show` showed
  new default route; `ping 192.168.0.1` and `ping 8.8.8.8` both succeeded; SSH session
  stayed connected throughout.
- Ran `apt update` — succeeded once gateway was fixed (582 kB fetched, 72 packages
  upgradable). Reviewed `apt list --upgradable` before proceeding: included 2 kernel
  packages (proxmox-kernel-6.17, proxmox-kernel-7.0), pve-manager, qemu-server,
  pve-firewall, pve-container, ZFS packages, and standard Debian libs — no unexpected
  removals (Removing: 0).
- Chose not to snapshot before upgrade (pve01 is the bare-metal host, not a VM —
  `qm snapshot` doesn't apply to the host itself; confirmed prior kernels remain
  selectable in GRUB as the actual fallback).
- Ran `apt upgrade`, confirmed `Y` at the prompt. All 72 packages + 2 new kernel
  packages installed cleanly, no errors, no interactive config-conflict prompts.
  GRUB regenerated with 5 kernel entries available (7.0.14-5, 7.0.12-1, 6.17.13-18,
  6.17.13-13, 6.17.2-1).
- Confirmed both VMs (Kali, Win11-LTSC-Victim) were `stopped` before reboot — no
  running VM sessions to interrupt.
- Rebooted. Confirmed post-reboot: new kernel active (`7.0.14-5-pve`), vmbr1/192.168.0.201
  up, default route persisted, ping to 8.8.8.8 succeeded — gateway fix and bridge changes
  both survived reboot cleanly.

### Outcome
- NIC2 / `vmbr2` = no IP, no VMs, ready for pfSense WAN assignment.
- NIC1 / `vmbr1` = management, `192.168.0.201/24`, now with a working default
  gateway (`192.168.0.1`) — internet-reachable, confirmed persistent across reboot.
- pve01 fully patched: kernel `7.0.14-5-pve` running, pve-manager 9.2.4, all other
  packages current as of this date.
- Both VMs (Kali, Win11-LTSC-Victim) remained `stopped` throughout — unaffected by
  the reboot since neither was running beforehand.
- Current bridge state: `vmbr1` (NIC1, management) · `vmbr2` (NIC2, pfSense WAN,
  unconfigured) · `vmbr3` (NIC3, pfSense LAN trunk, VLAN-aware, unconfigured) ·
  NIC4 unbridged (spare / SPAN fallback per plan).

### Lessons
1. **A "no default gateway" symptom can look identical to a DNS or firewall
   problem** (`apt update` failing on every repo) — check `ip route show` for a
   default route early when package manager connectivity fails entirely, not just
   DNS resolution or specific host reachability.
2. **A prior log entry stating a change was made is not the same as confirming
   it's still true.** The 2026-07-16 entry documented adding a gateway to vmbr1,
   but it was absent from the live config weeks later with no record of removal.
   Verify current state against the live system before trusting a past log entry
   as still-accurate, especially for anything network-related.
3. Proxmox's `qm snapshot` only covers VMs, not the bare-metal host itself — there
   is no built-in Proxmox mechanism to snapshot pve01's own OS/filesystem. The
   actual fallback for a host-level package upgrade is confirming prior kernels
   remain selectable in GRUB, plus the standard Recovery Ladder (iDRAC, console).
4. Before any host-level `apt upgrade` involving a kernel: confirm all VMs' running
   state first — a reboot's impact scope depends entirely on what's actually
   running at the time, not on what's configured to autostart.
---

## 2026-07-23 — VLAN creation, trunk to pfSense, management access port (Phase A Steps 4-6)

**Phase:** A
**Goal:** Create the three VLANs, configure the trunk port to pfSense (NIC3/Gi1/0/3), and put the PC's management port into the Management VLAN.
**Rollback:** None needed for VLAN creation (additive, no prior state to preserve). Config saved to NVRAM only after each step's output was confirmed correct.
**Transcript:** session logs from today, PuTTY (switch, COM14)

### What happened

**Pre-check — `show run` reviewed before touching anything.** Confirmed switch still matched the clean, factory-reset baseline from 2026-07-17: hostname `Switch`, no VLANs beyond default `Vlan1`, no `aaa new-model`, no passwords on `con 0`/`vty` lines, all 52 `Gi1/0/1`-`1/0/52` ports present and unconfigured.

**Step 4 — created VLANs 10 (MGMT), 20 (INFRA), 30 (RANGE):**
```
vlan 10
 name MGMT
vlan 20
 name INFRA
vlan 30
 name RANGE
```
`show vlan brief` confirmed all three as `active`, no ports assigned yet. Clean on the first attempt.

**Step 5 — trunk port to pfSense (Gi1/0/3).** First identified which physical port NIC3 is actually cabled into via `show interfaces status` (read-only) — confirmed `Gi1/0/3` as the only port showing `connected`, matching Gb3/NIC3. Configured:
```
switchport mode trunk
switchport trunk allowed vlan 10,20,30
```
- `switchport trunk encapsulation dot1q` was rejected (`% Invalid input detected`) — expected on this 2960X, which only supports 802.1Q natively (unlike 3560/3750 models, which need the encapsulation selected explicitly). Skipped, no impact.
- **First pass missed the `switchport trunk allowed vlan 10,20,30` line** — it appears to have been dropped from the paste. `show interfaces trunk` still showed success (`trunking`, `802.1q`) but caught on closer inspection: "Vlans allowed on trunk" read `1-4094`, not `10,20,30`. Re-ran the missing line; second `show interfaces trunk` confirmed `10,20,30` correctly on both the "allowed" and "allowed and active" lines.
- Interface flapped (`Gi1/0/3` down/up, `Vlan1` SVI down/up) twice — once after `switchport mode trunk`, once after restricting the VLAN list. Both were transient STP renegotiation, self-resolved within ~30 seconds each time. No action needed.

**Step 6 — PC's management access port.** Initial plan was to use `Gi1/0/48` directly, but investigation revealed the PC's cable was actually running through the home dumb switch as a shared uplink — not a dedicated line. Putting `Gi1/0/48` into VLAN 10 access mode as originally planned would have pulled the entire dumb switch (and everything behind it) onto VLAN 10, not just the PC. Confirmed multiple other devices are plugged into that dumb switch, including a possibility the home router is among them — decided against consolidating the dumb switch into the managed switch for now, to avoid stacking a second, larger network change on top of an unverified one.

**Fix:** used one of the two USB-to-Ethernet adapters (purchased for Phase C.5 SPAN duty, borrowed for this in the meantime) to run a dedicated point-to-point link from the PC directly into `Gi1/0/48`, bypassing the dumb switch entirely. PC's onboard NIC stays on the dumb switch/home network for normal internet; the USB adapter carries only lab management traffic.

Also discovered during port selection: `Gi1/0/49` and `Gi1/0/50` still show `err-disabled` even after the full `write erase`/`reload` factory reset. Since `err-disabled` is normally caused by a runtime condition (port security, BPDU guard, storm-control, etc.) that a factory erase wipes clean, both coming back err-disabled with no config present points to a **hardware-level fault on those two specific ports**, not leftover configuration. Decided to avoid `Gi1/0/49`/`Gi1/0/50` going forward rather than spend time debugging a likely-dead port.

With the dedicated link in place:
```
interface GigabitEthernet1/0/48
 switchport mode access
 switchport access vlan 10
```
`show vlan brief` confirmed `Gi1/0/48` listed under VLAN 10 (MGMT) and no longer under VLAN 1.

**Saved config:** `copy running-config startup-config` → `Building configuration... [OK]`. Confirmed written to NVRAM.

### Outcome
- VLANs 10 (MGMT), 20 (INFRA), 30 (RANGE) created and active.
- `Gi1/0/3` trunking correctly scoped to `10,20,30` only (not `1-4094`) — carries to pfSense's future LAN side (`vmbr3`).
- `Gi1/0/48` in VLAN 10 access mode, now a dedicated point-to-point link from the PC (via USB-Ethernet adapter) — isolated from the dumb switch and everything behind it.
- Switch config saved to NVRAM; survives reload.
- **`Gi1/0/49` and `Gi1/0/50` flagged as likely hardware-faulty** — avoid using them.
- One USB-to-Ethernet adapter is now in use for PC management access — remember to move it back before Phase C.5 needs it for SPAN, or confirm a second unit is available.
- Phase A Step 3 (building the pfSense VM itself) is **still not done** — bridges `vmbr2`/`vmbr3` exist, but no VM is attached to them. This is the next step, ahead of Step 7 (pfSense VLAN interfaces), which can't happen until the VM exists.

### Lessons
1. **A missing line in a multi-line paste can silently succeed** — the trunk showed `trunking`/`802.1q` correctly even with `switchport trunk allowed vlan` missing, because "no errors" and "matches the intended config" are not the same check. Always compare *every* relevant field in the `→ expect:` output, not just whether the command was accepted.
2. **`end` typed at the `Switch#` EXEC prompt (not inside a config sub-mode) is not a no-op** — IOS tries to resolve it as a hostname/telnet target instead, producing a DNS-lookup failure. Harmless, but confusing if unexpected. Only use `end` to exit a config sub-mode.
3. **A "management port" assumption should be physically verified, not taken on faith** — the PC's cable was assumed to be a dedicated line but was actually a shared dumb-switch uplink. Confirmed via direct question before applying the VLAN assignment, avoiding an accidental home-network-wide VLAN 10 membership.
4. **`Gi1/0/49`/`Gi1/0/50` remaining `err-disabled` after a full factory erase is a hardware-fault signal**, not a config leftover — `err-disabled` requires an active trigger condition to persist, and factory erase removes all such conditions. Avoid these two ports going forward.

---

## 2026-07-24 — pfSense VM built and installed (Phase A Step 3)

**Phase:** A
**Goal:** Build the pfSense VM on `pve01`, attach it to the two existing bridges (`vmbr2` WAN, `vmbr3` LAN trunk), and get pfSense CE actually installed and booted.
**Rollback:** None needed — new VM, no prior state. ISO confirmed present before starting; bridges re-verified live rather than trusted from the log (see Lessons).
**Transcript:** session logs from today, pve01 console/web UI + PuTTY (switch, for pre-checks)

### What happened

**Pre-checks before touching Proxmox:**
- Confirmed pfSense install media: `netgate-installer-v1.2-RELEASE-amd64.iso` present in `/var/lib/vz/template/iso/`. Note: Netgate no longer ships version-pinned pfSense ISOs directly — the "Netgate Installer" is a bootstrap image whose own version (v1.2) is unrelated to the pfSense version installed; the actual pfSense release is chosen later, inside the installer, once it has internet access.
- Re-verified `vmbr2`/`vmbr3` live rather than trusting the 07-21 log entry (per gotcha #7): both initially showed `NO-CARRIER`/`DOWN` — traced to the Cisco switch simply being powered off since last session, not a real fault. Powered the switch back on; `eno3` came up (`LOWER_UP`) and `vmbr3` showed it as a forwarding member, matching `Gi1/0/3`'s `connected` status on the switch. `vmbr3`'s `ip -d link show` confirmed `vlan_filtering 1` — VLAN-aware flag genuinely active at the kernel level. Both bridges confirmed no IPv4 address (gotcha #1 still avoided).

**VM creation (VM 102, named `pfsense`):**
- OS: "Other" guest type, ISO = `netgate-installer-v1.2-RELEASE-amd64.iso`
- Disk: 32GB, VirtIO SCSI
- CPU/RAM: started at 2 cores/2GB; **bumped to 3 cores/4GB mid-install** because first-boot resource usage pegged out and installation was crawling. **TODO: revert to 2 cores/2GB (or whatever's actually needed) once past first-boot, if the higher allocation isn't required for steady-state operation.**
- Network: **`net0` → `vmbr2`** (WAN), **`net1` → `vmbr3`** (LAN trunk) — deliberately NOT left on the wizard's default `vmbr1`. Both NICs: VirtIO model, **no VLAN Tag** (blank — the trunk needs to pass VLANs 10/20/30 through untagged at this layer; pfSense creates the actual VLAN sub-interfaces later in Step 7), **firewall checkbox left unchecked** on both (Proxmox's firewall layer would otherwise double-filter traffic pfSense is already meant to be the sole firewall for).
- Confirmed `net0`/`net1` → `vmbr2`/`vmbr3` MAC pairing via the VM's Hardware tab before booting, to have an authoritative cross-check available during the installer's WAN/LAN interface-selection screens.

**Installer walkthrough:**
- WAN interface: `vtnet0` (confirmed via MAC match to `net0`/`vmbr2`) → DHCP client, VLAN tagging disabled (correct — WAN isn't a trunk)
- LAN interface: `vtnet1` (confirmed via MAC match to `net1`/`vmbr3`, and independently corroborated by the switch already showing `Gi1/0/3` connected) → left at factory-default `192.168.1.1/24` static, DHCPD enabled 192.168.1.100-199, VLAN tagging disabled on this parent interface (correct — VLAN 10/20/30 sub-interfaces get created on top of this in Step 7, not configured here)
- **Internet connectivity check failed** — WAN (`vmbr2`) had no physical cable yet at that point in the process; the Netgate Installer requires live internet access to fetch pfSense packages (they're not bundled in the small bootstrap ISO). **Fix:** identified and plugged NIC2's physical port into the dumb switch, confirmed via `ip link show eno2` showing `LOWER_UP`. Retried — WAN pulled a DHCP lease and the connectivity check passed.
- Subscription validation: prompted for pfSense Plus vs CE (device has no active Plus subscription, as expected) → selected **Install CE**.
- Software version: **2.8.1 (current stable)** — matches what was confirmed via web search before starting this phase.
- Filesystem: ZFS, Stripe (no redundancy — correct/only option for a single virtual disk), GPT partitioning. All defaults, all appropriate.
- Disk selection: single 32GB virtual disk, confirmed, destroyed/formatted (empty disk, no data at risk).
- Installation completed cleanly (`pfSense Post Installation setup .. done`).
- **Halted (not rebooted) before removing installer media** — detached the ISO from the VM's CD/DVD Drive (Hardware tab → "Do not use any media") to avoid it booting back into the installer. Proxmox needed a manual Stop after the guest's internal halt completed, since a FreeBSD-based halt doesn't power off the VM at the hypervisor level automatically.
- **First real boot succeeded:** `pfSense 2.8.1-RELEASE amd64`, "Bootup complete". Console menu confirmed both interfaces live: **WAN (vtnet0): 192.168.0.131/24 via DHCP** (from the dumb switch), **LAN (vtnet1): 192.168.1.1/24 static** (factory default, to be replaced by VLAN 10/20/30 interfaces in Step 7).

### Outcome
- **pfSense CE 2.8.1 installed and running** on VM 102 (`pfsense`), attached to `vmbr2` (WAN) and `vmbr3` (LAN trunk) correctly.
- Phase A Step 3 is now **done**. Steps 4-6 (VLANs, trunk, management access port) were already done as of 2026-07-23 — Phase A is now fully caught up through Step 6, with **Step 7 (create matching VLAN interfaces in pfSense's GUI) next**.
- WAN's physical cable (NIC2 → dumb switch) is not actually temporary — it matches the blueprint's intended final WAN destination, so no cable move needed later.
- **Outstanding TODO:** VM resources were bumped to 3 cores/4GB RAM mid-install to push through a slow first boot. Revisit once steady-state load is known — may not need to stay this high.
- pfSense's web GUI is not yet reachable from the management PC — LAN currently only knows `192.168.1.1/24`, not VLAN 10's real subnet. That gap closes in Step 7.

### Lessons
1. **The Netgate Installer needs internet access on WAN to complete installation** — it's a bootstrap image that fetches the actual pfSense packages live, not a self-contained ISO. Plan for WAN to have real connectivity before starting a pfSense install, not just before going into production.
2. **A build_log entry is a snapshot, not a guarantee** — `vmbr2`/`vmbr3` showing `NO-CARRIER` today wasn't a config regression, just the switch being powered off since last session. Re-verifying live state before trusting it (gotcha #7) caught this immediately instead of chasing a phantom problem.
3. **VM resource allocation may need temporary headroom for first boot/install**, separate from steady-state requirements — worth revisiting after initial setup rather than leaving an oversized allocation permanently by default.
4. **MAC address cross-checking between Proxmox's Hardware tab and the guest OS's interface-selection screen is a reliable way to confirm `vtnetN` → `netN` → bridge mapping**, more trustworthy than assuming numbering order alone.

---

## 2026-07-25 — pfSense VLAN interfaces, setup wizard, first snapshot (Phase A Step 5, acceptance check passed)

**Phase:** A
**Goal:** Create the three VLAN sub-interfaces inside pfSense, assign them real subnets, confirm the web GUI is reachable from the Management VLAN, and take a clean rollback snapshot before any firewall rules exist.
**Rollback:** Snapshot `pfsense-clean-install` taken at the end of this session, VM running (RAM not included) — first real rollback point for this VM.
**Transcript:** session logs from today, Proxmox VM console (pfSense) + browser (pfSense GUI)

### What happened

**VLAN sub-interfaces created** via console menu option `1) Assign Interfaces`, entering `y` to configure VLANs:
- `vtnet1.10` (tag 10, parent `vtnet1`) — Management
- `vtnet1.20` (tag 20, parent `vtnet1`) — Infra
- `vtnet1.30` (tag 30, parent `vtnet1`) — Range

Confirmed WAN/LAN/OPT1/OPT2 mapping before applying: WAN→`vtnet0`, LAN→`vtnet1.10`, OPT1→`vtnet1.20`, OPT2→`vtnet1.30`.

**IP assignment** via console menu option `2) Set interface(s) IP address`, once per interface — same pattern each time (static, no upstream gateway on any of the three, IPv6 skipped, DHCP server enabled, kept HTTPS for the webConfigurator):
- LAN (`vtnet1.10`): `10.10.10.1/24`, DHCP range `10.10.10.100`–`10.10.10.199`
- OPT1 (`vtnet1.20`): `10.10.20.1/24`, DHCP range `10.10.20.100`–`10.10.20.199`
- OPT2 (`vtnet1.30`): `10.10.30.1/24`, DHCP range `10.10.30.100`–`10.10.30.199`

**Client-side confirmation:** PC's USB-Ethernet adapter picked up a DHCP lease on the first attempt — `10.10.10.100/24`, gateway `10.10.10.1`. Browsed to `https://10.10.10.1/` and reached pfSense's login page (self-signed cert warning, expected/accepted). **This is Phase A's acceptance check, passed.**

**Logged in** with default credentials (forced password change immediately — expected pfSense CE behavior, kept as a real security step rather than skipped).

**Ran the pfSense setup wizard** (auto-prompted on first GUI login):
- General info: hostname `pfSense`, domain `home.arpa` (already correct default, not `.local`) — left as-is.
- NTP: kept default time server (`2.pfsense.pool.ntp.org`, Netgate's own pool) — no reason to point elsewhere.
- **WAN configuration screen — one real fix required here:** "Block RFC1918 Private Networks" was checked by default. **Unchecked it.** This build's WAN doesn't face a real ISP — it faces the home network (`192.168.0.0/24`, itself an RFC1918 range), so leaving this checked would have caused pfSense to block its own WAN's legitimate upstream traffic as if it were spoofed. Left "Block bogon networks" checked — that one's still correct regardless of WAN's private/public nature.
- LAN configuration screen: showed the already-configured `10.10.10.1/24` correctly — confirms the wizard picked up existing state rather than overwriting it.
- Reloaded configuration, wizard completed cleanly.

**Snapshot taken:** `pfsense-clean-install`, VM 102, live (VM running throughout, RAM not included in the snapshot — unnecessary for a config checkpoint like this). Confirmed via the Snapshots tab.

### Outcome
- **Phase A is now fully complete.** All six steps done: VLANs created, trunk configured and scoped correctly, PC's management access isolated via dedicated USB-adapter link, pfSense VM built and installed, VLAN interfaces created and IP-addressed, GUI reachability confirmed from the Management VLAN.
- pfSense reachable at `https://10.10.10.1/` from any device on VLAN 10.
- **First rollback point for the pfSense VM exists** (`pfsense-clean-install`) — a clean, no-firewall-rules-yet state to return to if Phase A.5's rule-writing goes sideways.
- **Next up: Phase A.5 — writing the Range VLAN isolation rule.** Per the project's standing two-tier rule, this is core security logic: Claude drafts a reference version and explains the reasoning, Michael writes the final rule that actually gets applied.

### Lessons
1. **"Block RFC1918 Private Networks" on WAN is the wrong default for any home-lab topology where WAN's upstream is itself a private network** (a home router, not a real ISP handoff). This is a genuine pfSense wizard default that needs deliberate correction in setups like this one — it isn't obviously wrong until you think through what WAN is actually plugged into.
2. **The pfSense setup wizard re-displaying already-configured values (LAN's `10.10.10.1/24`) rather than blank defaults is a good sign**, not a coincidence — it confirms the wizard is reading and preserving existing config rather than silently resetting it. Worth explicitly checking for on any wizard that runs after manual pre-configuration.
3. **A live VM snapshot (no RAM) is sufficient for a configuration checkpoint** — no need to stop the VM or capture memory state when the goal is "return to this config later," not "resume this exact in-progress session."

---

## 2026-07-30 — Range isolation rule written and applied (Phase A.5)

**Phase:** A.5
**Goal:** Write and apply the pfSense Range-VLAN isolation rule — deny all outbound from Range, with one explicit allow to Infra's Wazuh ingest port.
**Rollback:** Proxmox snapshot `pre-isolation-rule` (VM 102, taken via `qm snapshot 102 pre-isolation-rule` from the `pve01` shell — confirmed via `Logical volume "snap_vm-102-disk-0_pre-isolation-rule" created`). pfSense config also exported as XML via Diagnostics > Backup & Restore > Download Configuration, saved to `C:\Users\micha\SOC-Lab\Logs\`.
**Transcript:** session recording on throughout (PowerShell transcript).

### What happened

**Gate items confirmed before any firewall change:**
1. Fresh Proxmox snapshot `pre-isolation-rule` taken on VM 102 — succeeded cleanly.
2. pfSense XML config exported and saved locally, opened to confirm it wasn't empty.
3. Session recording confirmed on.

**Interface labels renamed for clarity** (cosmetic only, no functional change): LAN → MGMT10, OPT1 → INFRA20, OPT2 → RANGE30, via Interfaces > [interface] > Description, applied once after all three renames. Firewall > Rules tabs picked up the new names automatically.

**Rule 1 — the allow, built on the RANGE30 tab:**
- Action: Pass
- Interface: RANGE30
- Address Family: IPv4
- Protocol: TCP
- Source: RANGE30 subnets
- Destination: INFRA20 subnets *(placeholder — Wazuh doesn't exist until Phase B; to be tightened to Wazuh's single host IP once built)*
- Destination Port Range: 1514 to 1514
- Description: `allow range -> wazuh agent ingest ONLY`

**Rule 2 — the default deny, built second:**
- Action: Block
- Interface: RANGE30
- Address Family: IPv4
- Protocol: any
- Source: RANGE30 subnets
- Destination: any
- Description: `default deny - range is isolated`

Rules were built and read back field-by-field using Claude in Chrome in a strict "point/describe only, never click or type" mode — the extension highlighted each field and stated the value to enter; every field was typed in by hand and confirmed via screenshot before Save.

**Reordering gotcha, caught live:** dragged Rule 2 to confirm position relative to Rule 1. Discovered that a drag-to-reorder is **not committed by the drag alone** — clicking **Apply Changes** immediately after a drag (without an intervening **Save** on the rule list) triggered pfSense's own "unsaved changes" warning. Cancelled out of that warning, clicked **Save** on the rule list first (which committed the new order), *then* clicked Apply Changes. Confirmed the browser-level warning is a real safety net here, not just a mechanical prompt — it caught exactly the failure mode it exists to catch.

**Applied successfully:** pfSense returned "The changes have been applied successfully. The firewall rules are now reloading in the background," rule order preserved on reload (Pass above Block, confirmed on both interface and description columns).

**Post-apply sanity check:** confirmed continued access to the pfSense GUI from MGMT10 immediately after Apply — the rule set only targets RANGE30, MGMT10 untouched, but verified rather than assumed per the standing "confirm console/recovery access" rule.

### Outcome
- Both rules live on the RANGE30 interface: Pass (RANGE30→INFRA20 subnets, TCP 1514) above Block (RANGE30→any, any).
- pfSense GUI access from MGMT10 confirmed unaffected.
- **Rule exists and is correctly scoped, but isolation is not yet acceptance-tested live** — Kali and Win11-LTSC-victim are still on `vmbr1` (flat network), not yet moved to the Range VLAN. That move is Phase B steps 6–7. The real "no internet, no home net, no MGMT, only Wazuh:1514" test can't run until a VM actually sits on RANGE30.
- Destination on Rule 1 (INFRA20 subnets) is intentionally broader than the final design (single Wazuh host) — revisit once Wazuh is deployed in Phase B.

### Lessons
1. **A drag-to-reorder in pfSense's rule list is not committed until you click Save on the rule list itself** — clicking Apply Changes right after a drag, with no Save in between, triggers pfSense's "unsaved changes" warning rather than silently applying the old order. Sequence going forward: drag → **Save** → **Apply Changes**.
2. **Renaming interface descriptions (LAN/OPT1/OPT2 → MGMT10/INFRA20/RANGE30) is purely cosmetic** but meaningfully reduces the chance of picking the wrong tab under pressure — worth doing early in any pfSense build, before writing rules against generic slot names.
3. **A rule being applied cleanly is not the same as the isolation being proven.** Phase A.5's acceptance check (no internet/no home net/no MGMT reachability from an actual Range-VLAN device) can't run until Phase B moves a real VM onto RANGE30 — don't mark this phase fully complete until that check has actually run.
---

## 2026-07-31 — Ubuntu Server VM built (Phase B Step 1), Infra outbound rules written and live-tested

**Phase:**       B (Step 1)
**Goal:**        Build the Ubuntu Server VM on pve01/Infra VLAN for the Docker/Wazuh substrate; enable OpenSSH. Along the way, write and verify the Infra outbound internet rule that had been sitting unresolved on the Open Decisions list.
**Rollback:**    pfSense config exported as XML (`pfsense-config-2026-07-31-pre-infra-rules.xml`, saved to `C:\Users\micha\SOC-Lab\Logs\`) before any firewall rule changes. No Proxmox snapshot taken — additive rule changes only, no risk to MGMT10 access. VM 103 itself is new-build; no rollback needed pre-install.
**Transcript:**  session logs from today — Proxmox console (VM 103) + pfSense GUI via Claude in Chrome + PowerShell (SSH)

### What happened

**ISO selection.** Confirmed Ubuntu 26.04 LTS is now current, but chose **24.04.4 LTS** deliberately — most Wazuh/Docker Compose guides are still written against 24.04, and 26.04 is only ~3 months old. First upload attempt was actually the **Desktop** ISO by mistake — caught before VM creation via the ISO Images list; correct Server ISO (`ubuntu-24.04.4-live-server-amd64.iso`, 3.17 GiB) uploaded and confirmed.

**VM 103 (`ubuntu-soc-host`) created** on `pve01`: 8GB RAM (ballooning disabled, min=max), 100GB disk (VirtIO SCSI, `local-lvm`), network on `vmbr3` tagged VLAN 20 (Infra). **Two Confirm-screen catches before Finish:** cores/sockets were inverted (4 sockets x 1 core instead of 1 socket x 4 cores) - fixed. CPU type defaulted to x86-64-v2-AES instead of host - first edit didn't persist through to Confirm, caught and re-applied correctly on retry.

**Installer walkthrough:** chose "Try or Install Ubuntu Server" (GA kernel, not HWE). Full "Ubuntu Server" install (not minimized). No third-party drivers, no Ubuntu Pro, no Featured Server Snaps selected (Docker will be installed via the apt-repo method per blueprint Phase B step 3, not snap or curl|sh).

**Network config initially failed** ("autoconfiguration failed" on ens18) - root cause was simply that pfSense (VM 102) wasn't powered on. Confirmed the nightly full shutdown of pve01 + the Cisco switch is a current standing habit; flagged (not resolved) that this conflicts with the blueprint's "always-on service host" design intent for Phase D+ continuous monitoring. Powered switch -> pve01 -> pfSense VM back up in order; DHCP succeeded, ens18 got 10.10.20.100/24.

**Mirror test failed** next: "Temporary failure resolving 'archive.ubuntu.com'". Root cause: INFRA20 (OPT1) had zero firewall rules - unlike RANGE30 (explicit Phase A.5 rules), OPT interfaces get no automatic allow rule the way the LAN-role interface does. Implicit deny blocked everything on that interface, including DNS queries to pfSense's own resolver - DHCP worked because it's handled below the packet-filter layer, but nothing else was. This is exactly the "Infra VLAN outbound internet rule" item that had been sitting on the Open Decisions list.

**Wrote and applied 3 Pass rules on INFRA20** (config exported first): DNS (TCP/UDP 53), HTTP (TCP 80), HTTPS (TCP 443), all scoped INFRA20 subnets -> any. Chose scoped-by-port over a blanket allow-all, consistent with the least-privilege pattern used everywhere else in this build. Built via Claude in Chrome's point-and-describe workflow, one rule at a time. Two description-field mixups along the way - a stray "-> " copy-paste artifact, then an over-correction that replaced the description with a restated field dump - both caught on screenshot review and fixed manually. New standing convention adopted: rule descriptions now read "allow <INTERFACE_NAME> -> <PURPOSE> (<PORT>)" (e.g. "allow INFRA20 -> HTTPS (443)") - saved to memory for future sessions.

**Retried mirror test - hung twice**, ~24 minutes unresponsive each time, both right after a small InRelease fetch succeeded but before the actual package index (Packages.gz) came through. Diagnosed via the installer's built-in debug shell (Help -> Enter shell / Ctrl+Z): nslookup resolved fine, curl -I succeeded instantly, but ping produced no result. Root cause: ICMP was completely unhandled on INFRA20. The three rules written covered TCP/UDP 53/80/443 only - no ICMP - which meant Path MTU Discovery had no way to signal "fragmentation needed" back to the client. Small single-packet transfers worked fine; anything requiring an oversized packet along the path hung indefinitely instead of failing cleanly.

**Added a 4th rule - first attempt set Protocol to IGMP by mistake** (not ICMP) - caught reviewing the full rule list screenshot post-apply, corrected to ICMP/Subtype "any", re-applied. Confirmed working: curl -I succeeded on retry with a full 252KB package-index fetch, and the installer reported "This mirror location passed tests."

**Console input became unreliable** immediately after (noVNC session split every keystroke into its own line, "command not found" per character) - unable to get a clean ping confirmation in-shell. Rather than keep fighting it, did a hard Stop/Start power cycle on VM 103 (no data at risk - pre-storage-configuration). Came back up cleanly into the installer at the same point.

**Storage:** "Use an entire disk" + LVM (chosen over plain partitioning specifically for future resize flexibility). LUKS encryption explicitly skipped - this VM's operating pattern (nightly full power-cycle) would mean manually entering a passphrase every morning before Docker/Wazuh could even start; the threat model doesn't justify that recurring friction for a home lab. Caught a default-guided-partitioning gap: Subiquity's guided layout only allocated ~49G of the 98G volume group to /, leaving 49G stranded as unallocated free space. Manually edited ubuntu-lv to the full 97.996G, mounted at /, confirmed zero free space remaining before proceeding.

**Profile:** name "Michael Mathews", hostname wazuh-host, username michael (matches ai-vm's existing account convention). OpenSSH server install checked, password auth left enabled (no key imported at install time; hardening to key-only auth using the existing id_ed25519 is a follow-up, not done tonight).

**Install completed** - openssh-server installed, security updates pulled successfully (a second real-world confirmation the INFRA20 rules work, this time for actual package traffic rather than just the mirror test).

**Mistake, mine:** instructed ejecting the ISO from the VM's CD/DVD drive before the installer's final "Reboot Now" prompt actually appeared, rather than after. The live installer environment's own root filesystem was mounted from that ISO - pulling it while still live caused casper.service/cdrom unmount failures and a stuck shutdown (~15 minutes at subiquity/late/run:), followed by a second failed reboot attempt (Failed unmounting cdrom.mount, Failed to start casper.service, Failed to execute shutdown binary) since the live environment had no working root fs left to shut down properly. No actual damage - curtin (partitioning, GRUB install, package installs) had already fully completed before this happened; only the dying live environment was affected. Recovered with a hard Stop/Start (CD/DVD already set to "Do not use any media") - booted cleanly straight into the installed OS.

**Confirmed working:** console login as michael succeeded; ip a show ens18 showed 10.10.20.100/24 as expected; SSH from PowerShell (ssh michael@10.10.20.100) connected without the KexAlgorithms workaround pve01 needed. Host key fingerprint manually verified against the one printed in the VM's own boot log before accepting (matched exactly). Session ended with a graceful Shutdown (not a hard Stop, since this is now a real installed OS).

### Outcome
- **VM 103 (ubuntu-soc-host / hostname wazuh-host)** built and fully installed: Ubuntu Server 24.04.4 LTS, 8GB RAM/4 cores/host CPU, 100GB disk (single LVM volume, no leftover free space, no encryption), on vmbr3 tagged VLAN 20.
- **SSH confirmed working**: michael@10.10.20.100, password auth. Key-only auth hardening still open for later.
- **INFRA20 now has 4 firewall rules**: DNS (53), HTTP (80), HTTPS (443), ICMP (any subtype) - all INFRA20 subnets -> any, scoped rather than blanket-allow. This closes the "Infra VLAN outbound internet rule" item from Open Decisions, and it's now live-tested by a real OS install rather than just theoretical.
- **Phase B Step 1 is complete.** Step 2 (host prep - vm.max_map_count, Docker install via apt) is next, not started tonight.
- **Still open, unchanged:** pve01/switch nightly shutdown vs. the blueprint's always-on-service-host design - flagged, not resolved. Docx workbooks not touched tonight. agent-registry.md in project knowledge is still a stale build_log.md duplicate - parked, not urgent until Phase D.

### Lessons
1. **OPT interfaces in pfSense get zero rules by default - only the LAN-role interface gets an automatic allow-all.** Any new VLAN/segment beyond the original three needs its outbound rule written explicitly before assuming internet reachability, the same way Range needed its Wazuh-allow rule.
2. **DNS-resolution-only failures and full-hang-after-partial-success are different symptoms pointing at different rule gaps.** The first meant no rules existed at all; the second - succeeding on small transfers, hanging on larger ones - specifically means ICMP/PMTUD is missing, not DNS or the main port itself.
3. **A live installer's root filesystem may still be the mounted ISO, even well into the install process.** Don't eject virtual media until the installer's own explicit "Reboot Now" (or equivalent) prompt appears.
4. **Subiquity's guided "entire disk" LVM layout doesn't use the full disk by default** - it conservatively splits it, leaving real leftover space unallocated unless you manually resize the logical volume up to the max before finishing storage configuration.
5. **A stuck/glitchy noVNC console (garbled or per-character input) is often better solved by a clean power-cycle than by fighting the session** - especially pre-storage-configuration, where there's nothing to lose.

---

## 2026-08-03 — Phase B Steps 2-4 complete: host prep, Docker, Git/SSH + repo clone; INFRA20 gains a 5th rule; docx workbooks retired

**Phase:**       B (Steps 2, 3, 4)
**Goal:**        Finish host prep on `wazuh-host` for Wazuh's indexer, install Docker + Compose, install Git and get the repo cloned onto the box with its own SSH deploy key. Along the way: fix a new INFRA20 gap (port 22), retire the docx workbooks in favor of the `.md` files as sole source of truth, and rewrite `PROJECT-INSTRUCTIONS.md` in first person.
**Rollback:**    No Proxmox snapshot — all changes are additive (package installs, a new firewall Pass rule, a fresh SSH keypair, a repo clone). No risk to MGMT10 or existing services.
**Transcript:**  session logs today — SSH (PowerShell → `wazuh-host`), pfSense GUI via Claude in Chrome, GitHub web UI, PowerShell (repo work on the Windows PC)

### What happened

**Step 2 — host prep.** Set `vm.max_map_count=262144` live (`sudo sysctl -w`), then persisted it to `/etc/sysctl.conf` via `echo "..." | sudo tee -a` — confirmed necessary because a plain `sudo echo ... >> file` would fail: the shell sets up `>>` redirection using the calling user's own permissions *before* `sudo` ever runs, so only `tee` (itself elevated by `sudo`) can actually write to a root-owned file this way. Verified with `grep` that the line landed in the file, not just echoed to the terminal.

**Step 3 — Docker + Compose, official apt method.** Ran the defensive removal of six potentially-conflicting packages (docker.io, docker-doc, docker-compose, docker-compose-v2, podman-docker, containerd, runc) — all six came back "not installed," a clean baseline. Added Docker's GPG key to `/etc/apt/keyrings/docker.asc` and the official repo to `/etc/apt/sources.list.d/docker.list`, scoped to `amd64`/`noble`/`stable` via `dpkg --print-architecture` and `/etc/os-release` rather than hardcoding either. `apt-get update` confirmed the new repo was actually being read (fetched from `download.docker.com` alongside the Ubuntu mirrors). Installed `docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin` — 103MB, 7 packages, clean install, `docker.service`/`docker.socket` both enabled via systemd on install. Verified via `systemctl status docker` (active/running) and `docker run hello-world` (pulled and ran successfully — a real test of INFRA20's HTTPS rule against Docker Hub, not just apt mirrors). Added `michael` to the `docker` group (`sudo usermod -aG docker michael`) for non-root Docker usage going forward — flagged explicitly that this is functionally equivalent to root access on this box (containers can mount the host filesystem), acceptable here since it's a single-user lab box and `michael` already has `sudo`. Required a full SSH session restart (group membership is read at login, not live) — confirmed via `groups` showing `docker` in the list, then `docker run hello-world` again with no `sudo` needed.

**Step 4 — Git, identity, SSH, clone.** Git was already present (2.43.0, stock on 24.04). Set `git config --global user.name "Michael Mathews"` and `user.email` — flagged the choice between real email vs. GitHub's `@users.noreply.github.com` alias before setting it, given this repo is public and deliberately framed for employers to browse; decided to keep the real email (`michaelsoup@gmail.com`).

Generated a fresh ED25519 keypair on `wazuh-host` itself (`ssh-keygen -t ed25519 -C "wazuh-host-github"`, no passphrase, deliberately separate from `pve01`'s existing key — each machine gets its own). Added the public key to GitHub as a **repo-scoped Deploy Key** (not an account-level key) on `SOC-LAB`, with **write access enabled** — since this box will eventually push Wazuh Compose files/config. Confirmed added correctly via the Deploy Keys screen (fingerprint match, "Read/write," "Never used" until the next step).

**`ssh -T git@github.com` hung** on first attempt — diagnosed immediately as the same INFRA20 gap pattern from the DNS/ICMP issue during the OS install: port 22 (SSH) was never in the original four rules (DNS/HTTP/HTTPS/ICMP), so it fell to implicit deny with no response either way. **Added a 5th Pass rule to INFRA20** — TCP/22, INFRA20 subnets → any, description `allow INFRA20 -> SSH (22)` — same pattern as the prior four, no gate ceremony needed (purely additive, zero MGMT10 risk). One process hiccup building it: Claude in Chrome's point-and-confirm workflow briefly landed on the wrong tab (pfSense's Dashboard rather than the actual Edit Firewall Rule page) and flagged the mismatch itself rather than guessing — caught before any wrong values were entered; fields were typed in manually instead once on the correct page.

Retried `ssh -T git@github.com` — connected, host key fingerprint (`SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU`) manually verified against GitHub's officially published ED25519 fingerprint before accepting. Got the expected success message ("Hi SecurityJonesing/SOC-LAB! ... does not provide shell access").

**Cloned the repo** to `~/soc-lab` on `wazuh-host` via SSH (`git clone git@github.com:SecurityJonesing/SOC-LAB.git ~/soc-lab`) — 88 objects, clean, no errors. Confirmed via `ls` that the file set matches the current repo state exactly (four `.md` files, `README.md`, `Diagrams/` — docx files correctly absent).

**Docx workbook retirement (documentation cleanup, not lab infrastructure).** Discovered the `Updated 7-16-2026` folder (renamed to `Build-Transcripts` earlier this session) is actually the **repo root itself**, not a subfolder holding old transcripts as previously assumed — confirmed via `git remote -v` pointing straight at `SecurityJonesing/SOC-LAB.git` and being the only location where git commands worked. The rename itself caused zero git issues (git doesn't track its own containing folder's name). This corrected a standing misunderstanding about the repo layout that had been carried since early sessions.

Reviewed all five `.docx` files in `Docx Files/`: `SOC-Lab-Agent-Registry.docx` (7/16, predates any real agent content, stale duplicate of the same problem `agent-registry.md` itself had), `SOC-Lab-Project-Instructions.docx` (7/27, redundant with the `.md` version), and `SOC-Lab-Cheatsheet.docx` / `SOC-Lab-Lesson-Plans.docx` / `SOC-Lab-Phase-Checklists.docx` (all 7/29, the only three ever actually documented as the "official" workbook set, but all predating Phase A.5 through tonight). Decided — explicitly, to reduce ongoing maintenance burden of keeping two documentation formats in sync — to **delete all five** and treat the four `.md` files as the sole source of truth going forward. Deleted the folder locally before running `git rm`; used `git add -A` instead (correct tool for already-deleted-outside-of-git files) after confirming via `git status` that exactly the five expected files showed as deleted and nothing else. Committed and pushed (`643c0bd`).

**`agent-registry.md` replaced** with a real governance scaffold (previously a stale, mislabeled duplicate of `build_log.md` itself) — states its purpose, lays out planned scope for `wazuh-triage-01` (Phase D) and `triage-router-01` (Phase E) ahead of when they're actually built, plus four governing principles (own identity, least privilege, separate audit trail, keep current) that apply to any future agent.

**`PROJECT-INSTRUCTIONS.md` rewritten in first person** throughout (was third-person "Michael does X" — now "I do X"), per explicit preference. Folded in several facts that had drifted out of sync: `wazuh-host` added to the VM list, INFRA20's firewall rules reflected, the `sudo tee` redirection gotcha, the pfSense rule-description naming convention, and the nightly-shutdown-vs-always-on-host tension moved into Open Decisions. Committed alongside the `agent-registry.md` replacement in `f5c420c` (message corrected via `git commit --amend` after the first pass had a mismatched description).

### Outcome
- **Phase B Steps 2, 3, and 4 are all complete.**
  - Step 2: `vm.max_map_count=262144`, live and persisted.
  - Step 3: Docker CE + Compose plugin installed via official apt repo, verified working, `michael` in the `docker` group for non-root use.
  - Step 4: Git installed, identity configured, dedicated SSH deploy key (`wazuh-host`, read/write) added to GitHub, repo cloned to `~/soc-lab` on the VM.
- **INFRA20 now has 5 firewall rules**: DNS (53), HTTP (80), HTTPS (443), ICMP (any), and now SSH (22) — all `INFRA20 subnets → any`.
- **Documentation set simplified**: `LAB-BLUEPRINT.md`, `PROJECT-INSTRUCTIONS.md`, `build_log.md`, `agent-registry.md` are now the entire documentation set — no more `.docx` companions to keep in sync. `LAB-BLUEPRINT.md` and `PROJECT-INSTRUCTIONS.md` still need their "source of truth" references to the deleted docx files cleaned up (see next entry/session).
- **Repo layout corrected**: `Build-Transcripts` (formerly `Updated 7-16-2026`) is the actual repo root, not a subfolder — all future path references should reflect this.
- **Phase B Step 5 is next** — deploying Wazuh (manager + indexer + dashboard) via Docker Compose. Not started tonight.

### Lessons
1. **Any new port/protocol beyond the original four INFRA20 rules needs its own explicit Pass rule** — same implicit-deny-on-OPT-interfaces story as before, this time SSH (22) rather than DNS/ICMP. Worth checking this proactively before assuming a new tool/service will "just work" on INFRA20.
2. **`sudo command >> file` silently fails on root-owned files even though the command itself is elevated** — the shell sets up the redirect using the calling user's permissions before `sudo` ever runs. `echo "..." | sudo tee -a file` is the correct pattern, since `tee` itself (not the shell's redirect) gets elevated.
3. **A folder's name inside a git repo, including the folder holding `.git` itself, is not tracked by git and can be renamed freely with zero repo-side consequences** — confirmed by testing directly rather than assuming. This also means "which folder is actually the repo root" is worth verifying with `git remote -v` rather than assumed from memory, especially in a project whose folder structure changed over time.
4. **Deploy keys, not account-level SSH keys, are the right scope for a single-repo, single-purpose credential** — same least-privilege pattern used throughout this build (Wazuh's agent scope, Range's firewall rule). Worth defaulting to the narrower option unless a broader one is specifically needed.
5. **Maintaining parallel documentation formats (markdown + docx) doubles the update burden for no real information gain**, when the markdown files are already the actual source of truth being edited live. Decided to consolidate rather than let the docx versions silently drift out of date the way they already had.

---

## 2026-08-03 — Wazuh deployed via Docker Compose (Phase B Step 5, in progress)

**Phase:**       B (Step 5)
**Goal:**        Deploy Wazuh (manager + indexer + dashboard) via Docker Compose on `wazuh-host`, pinned to a known-current release tag; confirm all three containers healthy and talking to each other; change the default admin password.
**Rollback:**    Not needed — new deployment, no prior state. `wazuh-docker` kept as a separate local-only clone (`~/wazuh-docker`), deliberately not folded into the tracked `~/soc-lab` project repo.
**Transcript:**  `session-2026-08-03-1901.txt` (PowerShell) + SSH session on `wazuh-host`

### What happened

**Version selection.** Confirmed via web search and cross-checked live against the repo's own tag list (`git tag -l`, `git describe --tags`) that `v4.14.6` is the current stable release — verified rather than trusted from the search result alone, per the standing "verify against live system" rule.

**Cloned `wazuh-docker`** to `~/wazuh-docker` on `wazuh-host` (deliberately separate from `~/soc-lab`, the tracked project repo — this is third-party deployment tooling, not something we're authoring). Checked out cleanly at the `v4.14.6` tag (detached HEAD, expected).

**Generated TLS certificates** via `docker compose -f generate-indexer-certs.yml run --rm generator` — root CA, admin, indexer, dashboard, and manager certs all created successfully. One cosmetic, non-fatal error in the script (`find: command not found`, a known quirk in the `wazuh-certs-generator:0.0.4` image) — didn't stop the cert generation, verified all 10 expected cert files landed on disk (`ls -la config/wazuh_indexer_ssl_certs/`).

**Brought up the full stack** via `docker compose up -d` — all three images (`wazuh-indexer`, `wazuh-dashboard`, `wazuh-manager`, all `4.14.6`) pulled cleanly, 14 volumes created, all three containers started with no errors.

**Verified health beyond "container started":**
- Indexer: `curl -k -u admin:SecretPassword https://localhost:9200` returned a valid cluster identity (`wazuh-cluster`), confirming it was genuinely responding, not just running.
- Manager: `docker logs single-node-wazuh.manager-1 --tail 30` showed a clean connection to the indexer, alert templates loaded, ~15 index-connector streams initialized, vulnerability scanner started — no errors, no restart loop.

**Logged into the dashboard** (`https://10.10.20.100/`) via Claude in Chrome in point-and-describe mode — Claude navigated/screenshotted, Michael typed credentials and clicked. Default credentials (`admin` / `SecretPassword`) worked on the first try and landed directly on the Overview dashboard (0 agents registered, as expected — no agents deployed yet). Alerts already showing (45 medium, 141 low severity) with zero agents connected — these are Wazuh's own internal self-monitoring alerts (API activity, indexer health), not real detections.

**Password change attempt failed as expected, for a real reason.** The dashboard's own "Reset password" dialog returned `Failed to reset password. {"status":"FORBIDDEN","message":"Resource 'admin' is reserved."}`. Looked this up against current Wazuh docs rather than guessing: the `admin` indexer user is flagged `reserved: true` specifically to prevent password changes through the UI/API — the real password lives as a hash in `config/wazuh_indexer/internal_users.yml`, and `docker-compose.yml` carries a matching plaintext copy for the manager/dashboard containers to authenticate with. Changing one without the other breaks inter-container auth.

Confirmed the correct multi-step procedure from current docs (not yet executed — scoped out for next session):
1. Log out of the dashboard (persistent session cookies cause errors otherwise)
2. `docker compose down`
3. Generate a password hash via the indexer image's own tool: `docker run --rm -ti wazuh/wazuh-indexer:4.14.6 bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/hash.sh` (matched to our deployed `4.14.6` tag, not the doc example's `4.14.7`, to avoid any tool/runtime version mismatch)
4. Edit `config/wazuh_indexer/internal_users.yml` — replace `admin`'s hash
5. Edit `docker-compose.yml` — replace `INDEXER_PASSWORD` in both the `wazuh.manager` and `wazuh.dashboard` service blocks
6. `docker compose up -d`
7. `docker exec -it single-node-wazuh.indexer-1 bash`, then run `securityadmin.sh` inside the container to push the updated security config live
8. Log into the dashboard with the new password to confirm

Session ended here — logged out of the dashboard as step 1, remaining steps deferred to next session. Standard nightly shutdown followed (`docker compose down` on wazuh-host, VM shutdowns, pve01 shutdown, switch power-off).

### Outcome
- **Wazuh 4.14.6 deployed and confirmed healthy**: manager, indexer, and dashboard all running, verified via direct API/log checks rather than `docker compose ps` status alone.
- **Dashboard reachable and logging in successfully** from MGMT10 at `https://10.10.20.100/` — no new pfSense rule needed, MGMT10's default allow-all covered it.
- **Still on default credentials** (`admin` / `SecretPassword`) — real security gap, flagged, not yet closed. Next session picks up exactly here: log out (done), then `docker compose down` → hash → edit `internal_users.yml` → edit `docker-compose.yml` → `docker compose up -d` → `securityadmin.sh`.
- **Phase B Step 5 is functionally deployed but not complete** until the password change lands. Steps 6–9 (move Win11-LTSC-victim and Kali to Range VLAN, install Sysmon + Wazuh agent, re-test isolation, acceptance check) remain after that.

### Lessons
1. **Wazuh's `admin` indexer user can't have its password changed through the dashboard UI or API** — it's deliberately `reserved: true`. The only correct path is editing `internal_users.yml`'s hash directly and keeping `docker-compose.yml`'s plaintext copy in sync, then re-applying via `securityadmin.sh`. A UI "Forbidden" error here doesn't mean something's broken — it means you're on the wrong tool for this specific account.
2. **The certs-generator image (`wazuh-certs-generator:0.0.4`) throws a harmless `find: command not found`** during a permissions-cleanup step — cosmetic, doesn't stop cert generation, confirmed by checking the actual files on disk rather than trusting the log text alone.
3. **`docker compose ps` showing `Up` doesn't confirm a service is actually working** — worth a direct check (curl the indexer's own API, tail the manager's logs) before assuming a multi-container stack is genuinely healthy, not just running.
4. **Password/credential changes on multi-container stacks with shared secrets need every reference updated in lockstep** — the indexer's hash, and every other container's plaintext copy of the same password, or authentication between components breaks. Same "verify every field, not just that a command succeeded" discipline as the pfSense trunk-VLAN gotcha from Phase A.
5. **Claude in Chrome's own tab group is separate from your regular browsing tabs** — it can only see/control tabs it creates itself, not an existing tab you already have open elsewhere. Worth remembering for future GUI-driven steps: expect a fresh tab, not reuse of one you're already looking at.

## 2026-08-04 — Wazuh admin password change completed; securityadmin.sh path/JAVA_HOME troubleshooting (Phase B Step 5 complete)

**Phase:**       B (Step 5, closeout)
**Goal:**        Finish the deferred Wazuh admin password change from the prior session — apply `securityadmin.sh` to push the updated `internal_users.yml` hash into the running indexer security config, confirm login with the new credentials.
**Rollback:**    Not needed — continuation of the prior session's in-progress change. Docker Compose stack briefly recreated (`docker compose up -d`) with updated environment variables, not stopped in a data-losing way.
**Transcript:**  session log today — SSH on `wazuh-host` + an interactive `docker exec` shell inside the indexer container

### What happened

Confirmed containers were stopped from the prior night's clean shutdown (`docker compose ps` returned an empty table).

**Generated the bcrypt hash** for the new admin password via `docker run --rm -ti wazuh/wazuh-indexer:4.14.6 bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/hash.sh`.

**Edited `config/wazuh_indexer/internal_users.yml`** — viewed the file first to confirm exact structure, then edited only the `admin` block's `hash` field via `nano`, verified via `cat` that no other fields or users were disturbed.

**Edited `docker-compose.yml`** — checked via `grep -n "SecretPassword\|INDEXER_PASSWORD"` first, found exactly 2 occurrences (`wazuh.manager` block, `wazuh.dashboard` block). Edited both via `nano` to the new plaintext password, verified via `grep` afterward that both changed and no `SecretPassword` remained.

**Brought the stack back up** (`docker compose up -d`) — fast since images were already cached locally, all three containers started clean.

**`securityadmin.sh` troubleshooting — several rounds, root-caused properly rather than guessed around:**
1. First attempt ran the tool's bare filename directly on `wazuh-host`'s own shell instead of inside the container — `command not found`, corrected to a full `docker exec` invocation.
2. Second attempt (correctly inside the container) printed only a `JAVA_HOME`/`OPENSEARCH_JAVA_HOME` warning and returned silently to prompt — no success or failure message. Opened an interactive shell into the indexer container (`docker exec -it ... bash`) to investigate directly instead of continuing to guess flags from outside.
3. The generic cert path from `wazuh-docker`'s own documentation (`/usr/share/wazuh-indexer/certs/`) didn't exist. `find / -iname "*.pem" 2>/dev/null` located the real path: **`/usr/share/wazuh-indexer/config/certs/`** (`admin.pem`, `admin-key.pem`, `root-ca.pem`, `indexer.pem`, `indexer-key.pem`).
4. The documented config directory (`/usr/share/wazuh-indexer/opensearch-security/`) didn't exist either. `find / -iname "internal_users.yml" 2>/dev/null` located the real path: **`/usr/share/wazuh-indexer/config/opensearch-security/`**. Confirmed via `grep` on this file that it correctly reflected the host-edited hash — the volume mount was working fine; only the assumed paths were wrong.
5. Re-ran `securityadmin.sh` with the corrected paths and the **admin** cert (not the indexer's own service cert, since this is an administrative action, not a service-to-service connection) — still returned silently after the Java warning, no other output.
6. Read the script itself (`cat -n securityadmin.sh`) and found the real cause: its final line pipes the actual Java invocation through `2>/dev/null`, silently discarding any real error. The `JAVA_HOME` warning was the *only* thing this script was ever going to show, success or failure.
7. Manually reconstructed and ran the same Java invocation the script builds internally, without the `2>/dev/null` suppression — got a clean `java: command not found`, revealing that Java genuinely isn't on `$PATH` inside this container image.
8. `find / -iname "java" -type f 2>/dev/null` located the real binary: **`/usr/share/wazuh-indexer/jdk/bin/java`** — the image ships its own bundled JDK (expected for an OpenSearch-based container); the script's `JAVA_HOME`/`OPENSEARCH_JAVA_HOME` auto-detect just isn't populated in this image.
9. Ran the full `SecurityAdmin` invocation directly against that Java binary, with the correct config/cert paths — completed successfully: connected to the cluster (`GREEN`, `wazuh-cluster`, 1 node), all 10 config types updated including `internalusers`, ending in `Done with success`.

**Logged into the dashboard with the new password** — confirmed working end to end.

### Outcome
- **Phase B Step 5 is now fully complete.** Wazuh manager, indexer, and dashboard all deployed, verified healthy, and no longer on default credentials.
- **A real, version-specific procedure is now documented**, since the generic `wazuh-docker` doc examples didn't match this container's actual layout:
  - Real cert path: `/usr/share/wazuh-indexer/config/certs/`
  - Real security config path: `/usr/share/wazuh-indexer/config/opensearch-security/`
  - Real Java binary: `/usr/share/wazuh-indexer/jdk/bin/java` (not on `$PATH` — call by full path, or `export JAVA_HOME=/usr/share/wazuh-indexer/jdk` first)
  - `securityadmin.sh`'s own output is suppressed (`2>/dev/null` baked into the script) — a silent return to prompt is not evidence of success *or* failure; reconstruct and run the underlying `java` command directly to see the real result.
- **Phase B Step 6 is next**: move Win11-LTSC-victim to Range VLAN, install Sysmon + Wazuh agent. Not started — VM's current power state not yet confirmed before the session ended.

### Lessons
1. **Generic vendor documentation's example file paths may not match the actual layout inside a specific image/version.** Verify with `find` rather than trusting docs literally — same discipline used everywhere else in this build (switch IOS syntax, package names, etc.), now extended to container-internal paths.
2. **A shell script silently returning to prompt with no success/failure message is not evidence of success.** Check the script's own source for output suppression (`2>/dev/null`, redirected logging) before assuming a step completed — or failed.
3. **Container images that bundle their own JDK don't always populate `JAVA_HOME`/`$PATH` to point at it.** If a script relies on `which java` or `$PATH` and that's unset, call the bundled binary by its full, discovered path instead of assuming Java is missing entirely.
4. **The correct cert/key for an administrative security action (`securityadmin.sh`) is the dedicated admin cert** (`admin.pem`/`admin-key.pem`), not a service's own operational cert (e.g., the indexer's TLS cert) — the same least-privilege-of-purpose principle used elsewhere in this build, applied to certificate roles rather than user accounts.

## 2026-08-04 — Win11-LTSC-Victim moved to Range VLAN, isolation proven live, Sysmon + Wazuh agent deployed (Phase B Step 6 complete — Phase B done)

**Phase:**       B (Step 6, closeout)
**Goal:**        Move Win11-LTSC-Victim onto RANGE30, prove the Phase A.5 isolation rule against a real VM for the first time, then instrument it with Sysmon and the Wazuh agent so it reports to the manager.
**Rollback:**    No Proxmox snapshot for the VLAN move (VM was off; Hardware tab edit is trivially reversible). pfSense config exported before the new 1515 rule, per standing practice, though this was a purely additive change with no risk to MGMT10.
**Transcript:**  session logs today — Proxmox web UI via Claude in Chrome, PowerShell 7 and Windows PowerShell on the management PC, PowerShell on Win11-LTSC-Victim's own console

### What happened

**VLAN move.** Confirmed Win11-LTSC-Victim (VM 101) was powered off. Via Proxmox's Hardware tab (Claude in Chrome, point-and-describe — Michael clicked, Claude confirmed each field via screenshot before commit), changed `net0` from `bridge=vmbr1` (flat, untagged) to `bridge=vmbr3,tag=30` (Range VLAN). MAC address (`BC:24:11:5B:13:31`) and NIC model left untouched — confirmed via the Hardware tab listing before and after.

**Isolation proven live — the real Phase A.5 acceptance check, finally run.** Booted the VM. `ipconfig` showed a clean DHCP lease on the Range subnet: `10.10.30.100/24`, gateway `10.10.30.1` — confirms pfSense's RANGE30 DHCP scope working correctly. From inside the VM:
- `ping 8.8.8.8` → 100% loss (no internet) ✅ expected
- `ping 192.168.0.1` → 100% loss (no home network) ✅ expected
- `Test-NetConnection -ComputerName 10.10.20.100 -Port 1514` → `TcpTestSucceeded: True` ✅ expected (Wazuh ingest port specifically allowed; plain ICMP to the same host would have failed since the rule is TCP/1514-only, which is why `Test-NetConnection` was used instead of `ping` for this one)

This closes the caveat that's been sitting in the log since 2026-07-30 — the isolation rule was correctly configured but never actually tested against a live VM until tonight.

**Staging tools onto an intentionally internet-less VM.** Since RANGE30 has zero outbound internet by design, Sysmon, the SwiftOnSecurity config, and the Wazuh agent MSI all had to be downloaded on the **management PC** (which has internet via MGMT10's default allow) and transferred in via virtual media rather than a direct download inside the VM — the general pattern for any airgapped/isolated system, and one that will likely recur in Phase C for Atomic Red Team.

- Confirmed current versions live rather than from memory: Sysmon **v15.20/15.21** (Microsoft's Sysinternals page, updated mid-June 2026), Wazuh agent pinned to **4.14.6** specifically — matching the manager's version exactly, since a search turned up an explicit compatibility note that agent version must be ≤ manager version, and generic doc examples showing `4.14.7` would have been *higher* than our manager.
- Downloaded all three (`Sysmon.zip`, `sysmonconfig-export.xml`, `wazuh-agent-4.14.6-1.msi`) to `C:\Users\micha\SOC-Lab\Staging` via `Invoke-WebRequest`. First attempt was accidentally run in **cmd.exe**, not PowerShell (`Invoke-WebRequest` doesn't exist there) — caught by checking the prompt string (`C:\...>` vs `PS C:\...>`), corrected by opening real PowerShell 7.
- **Built an ISO from the staging folder to attach as virtual media** — this took several failed attempts before landing on a working approach:
  1. First script used `$stream.Read()` directly on the COM `ImageStream` object — failed with "does not contain a method named 'Read'" in PowerShell 7.
  2. Suspected a PS7-vs-PS5.1 COM interop difference; switched to Windows PowerShell 5.1 — **same error**, ruling that theory out.
  3. Root cause (confirmed via web search): this is a **long-standing, version-independent limitation** — `IStream` (what `ImageStream` returns) is a low-level COM interface with no "dispatch" layer, so PowerShell fundamentally cannot call `.Read()` on it directly, in any version. The standard, widely-documented fix is a small inline C# helper class, compiled at runtime via `Add-Type` with `/unsafe`, that does the raw marshaling correctly.
  4. Used that pattern (`ISOFile::Create`) — worked immediately, no further errors. ISO built at `C:\Users\micha\SOC-Lab\Staging\soclab-tools.iso`, ~9.7MB, verified via `Get-ChildItem`.
- Uploaded the ISO to Proxmox's `local` storage (System > Node > local > ISO Images > Upload) — this specific step had to be done by Michael directly, since selecting a local file requires the OS's native file picker, which sits outside anything the browser extension can see or interact with even in principle.
- Attached the ISO to VM 101's `ide0` CD/DVD drive (Storage: `local`, ISO image: `soclab-tools.iso`) via the Hardware tab — confirmed via the resulting device string: `local:iso/soclab-tools.iso,media=cdrom,size=9920K`.
- Confirmed inside the VM: all three files visible on the new CD drive, landed at **`E:`** (not the assumed `D:` — checked live via `Get-Volume` rather than guessing a second time).

**Sysmon installed.** Copied files from `E:\` to `C:\SOC-Tools`, extracted `Sysmon.zip`. Installed with `Sysmon64.exe -accepteula -i C:\SOC-Tools\sysmonconfig-export.xml` in an **elevated** PowerShell window (required for the driver install) — clean install, schema validated (config schema 4.50 against Sysmon's own 4.91, handled correctly), driver and service both installed and started with no errors. Verified real events flowing via `Get-WinEvent` — DNS queries (ID 22), process creation with full hash/parent-chain detail (ID 1), even a `CreateRemoteThread` event (ID 8) from normal DWM background activity — confirming SwiftOnSecurity's config is actually capturing the intended depth of telemetry, not just that the service reports "running."

**Wazuh agent installed, hit a real gap, fixed it.** Installed via `msiexec.exe /i wazuh-agent-4.14.6-1.msi /q WAZUH_MANAGER="10.10.20.100" WAZUH_AGENT_NAME="win11-ltsc-victim"` — silent install returned cleanly but the `WazuhSvc` service was `Stopped`, not auto-started. Started it manually; log showed a repeating cycle: `Requesting a key from server` → `ERROR: Unable to connect to enrollment service at '[10.10.20.100]:1515'`. `client.keys` was empty, confirming enrollment never completed.

**Root cause: the same "OPT/segment interfaces get zero rules by default" pattern that hit INFRA20 three times during Phase B Steps 1 and 4 — this time on RANGE30.** The existing Phase A.5 rule only covers port 1514 (ongoing agent-to-manager data traffic); a Windows agent's *first-time enrollment* handshake needs a separate one-time connection to port **1515** (the enrollment service), which RANGE30 had no rule for at all.

Wrote and applied a new pfSense rule on RANGE30 (Michael wrote the final rule per the two-tier standing rule, reviewed against a Claude-drafted reference first): **Pass, RANGE30 subnets → INFRA20 subnets, TCP, port 1515, description `allow RANGE30 -> Wazuh enrollment (1515)`** — placed above the existing rules, no reordering needed since order between two Pass rules above a single Block doesn't matter. Applied cleanly (`The changes have been applied successfully`), confirmed via screenshot showing all three rules (1515 Pass, 1514 Pass, default Block) in the list.

**One process note:** several browser tabs controlled by Claude in Chrome closed unexpectedly mid-session tonight for unclear reasons, disrupting the point-and-describe workflow a few times. When it kept happening around this specific rule, switched to Michael building the rule directly in pfSense's GUI himself and sending a screenshot for Claude to verify field-by-field afterward — worked cleanly and is a reasonable fallback pattern if the extension misbehaves again.

**Retried enrollment** — `Restart-Service WazuhSvc`, waited 15 seconds, checked `client.keys`: now populated with **agent ID `001`, name `win11-ltsc-victim`, and a real registration key**. Log confirmed `Connected to the server ([10.10.20.100]:1514/tcp)` and `Agent is now online`.

**Confirmed in the Wazuh dashboard directly** (Michael checked himself, since this required his new password, which Claude does not have and should never be asked to enter): agent `001`, name `win11-ltsc-victim`, IP `10.10.30.100`, OS correctly identified as Windows 11 Enterprise LTSC 2024, status **active**, agent version `4.14.6` matching the manager exactly, group `default`.

### Outcome
- **Phase B Step 6 is complete.** Win11-LTSC-Victim is on RANGE30, isolation is proven live (not just configured), Sysmon is capturing detailed telemetry with the SwiftOnSecurity ruleset, and the Wazuh agent is enrolled, connected, and reporting.
- **Phase B is now fully complete** — Docker/Git substrate, Wazuh deployed and password-secured, first real endpoint instrumented and confirmed reporting end to end.
- **RANGE30 now has 3 firewall rules**: Pass (1515, Wazuh enrollment), Pass (1514, Wazuh ingest), Block (default deny, everything else) — same "explicit allow, then deny" shape as INFRA20's rule growth during Phase B.
- **Reusable technique captured**: the ISO-staging pattern (download on MGMT PC → build ISO via the `IStream`/C# `Add-Type` workaround → upload to Proxmox → attach as virtual CD) is now a proven path for getting tools onto any isolated Range VM — directly relevant to Phase C's Atomic Red Team install, which will hit the exact same "no internet on this VM" constraint.
- Kali is still on `vmbr1`, not yet moved to Range VLAN — that was never actually in Phase B's scope (only Win11-LTSC-Victim was), worth confirming against `LAB-BLUEPRINT.md` before Phase C assumes Kali's placement.
- **Phase C (Detection Engineering) is next** — not started tonight.

### Lessons
1. **A firewall rule can be correctly configured and still not be "proven"** — Phase A.5's isolation rule sat untested against a real VM for five days between being written and actually being validated live. Don't mark an acceptance check complete until it's actually been run against real traffic, not just reasoned about.
2. **Every new VLAN/segment's "zero rules by default" gap can bite more than once, in different directions.** INFRA20 needed five separate explicit rules discovered one at a time (DNS, HTTP, HTTPS, ICMP, SSH); RANGE30's single Wazuh-ingest rule (1514) wasn't enough either — first-time agent *enrollment* uses a completely different port (1515) than ongoing data traffic. Check both when standing up any new agent-based service on a segmented network.
3. **`IStream.Read()` cannot be called directly from PowerShell, in any version** — this is a fundamental COM interop limitation (no IDispatch layer), not a PS5.1-vs-PS7 issue. The fix is always a small compiled C# helper via `Add-Type`, not a different PowerShell version.
4. **Silent `msiexec /q` success doesn't mean the installed service actually started or connected** — check `Get-Service` and the application's own log, not just the installer's exit behavior.
5. **When picking a package version for a new tool, check compatibility constraints explicitly** (agent ≤ manager, in this case) rather than grabbing whatever a generic doc example shows — a newer-looking version number isn't automatically the right one.
6. **The Claude in Chrome extension's controlled tabs closing unexpectedly is a real, if intermittent, failure mode** — when point-and-describe workflow breaks down repeatedly, falling back to "user builds it directly, Claude verifies via screenshot afterward" is a reasonable and fast recovery pattern rather than continuing to fight the tooling.
7. **A password typed during a session lives in that session's raw transcript regardless of care taken afterward** — the fix is discipline about never repeating it back in chat or writing it into any tracked documentation, not pretending it can be scrubbed retroactively. Rotate if full removal from history is ever actually required.
