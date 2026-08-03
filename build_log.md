# build_log.md
### Home SOC Lab — running record

Every change, its **actual** output, the decision behind it, and every rollback point. Appended as it happens — never reconstructed afterward.

**Why this file exists:** a previous attempt at Cisco + pfSense configuration (via a different AI tool) caused a lockout. There was no record of what had been changed, so there was no way to reverse it — which is what forced a full Proxmox reinstall. This file is the answer to "what exactly did we change?" before you need to ask it.

**What goes where:**
- **Raw terminal output** → session transcripts in `C:\Users\micha\SOC-Lab\Updated 7-16-2026\logs` (PuTTY logging, `Start-Transcript`, `script`). Everything, including typos and dead ends.
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
**Transcript:** `C:\Users\micha\SOC-Lab\Updated 7-16-2026\logs` (PuTTY → Session → Logging → All session output, set BEFORE connecting)

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
