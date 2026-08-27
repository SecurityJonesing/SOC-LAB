# build_log.md
### Home SOC Lab — running record

Every change, its **actual** output, the decision behind it, and every rollback point. Appended as it happens — never reconstructed afterward.

**Why this file exists:** a previous attempt at Cisco + pfSense configuration (via a different AI tool) caused a lockout. There was no record of what had been changed, so there was no way to reverse it — which is what forced a full Proxmox reinstall. This file is the answer to "what exactly did I change?" before I need to ask it.

**Note on scope (2026-08-06):** this file now also incorporates the build history of the `pve-ai`/`ai-vm` AI inference node, formerly tracked as a separate project (`hybrid_ai_node_build_plan.md` and its own `build_log.md`). That work chronologically predates the SOC lab build itself (2026-07-12/13, vs. the lab's own start on 2026-07-16), but this file is organized by **phase**, not strictly by date — so those entries are grouped under Phase E below, where that work actually belongs in `LAB-BLUEPRINT.md`, with their original dates retained. The original standalone files remain as historical source material but are no longer separately maintained.

**Note on reconstruction (2026-08-24):** this file was rebuilt from GitHub commit history after the working copy in this Claude project was found to be missing all entries prior to 2026-08-18. Recovered from commit `17c862c` ("Phase C Step 3 Continued") via GitHub's file history, then extended with the two entries below (commits `932da5b` and `3c4fdf7`), which were already correctly present in the project's working copy and independently confirmed accurate. Content is otherwise unedited from the recovered source.

**What goes where:**
- **Raw terminal output** → session transcripts in a local logs folder (PuTTY logging, `Start-Transcript`, `script`). Everything, including typos and dead ends.
- **This file** → the narrative. What I tried, what actually happened, what I decided, and how to undo it. Readable six months from now.

**Rules:**
- Record **actual** output, not what I expected. "It worked" is not an entry.
- Log dead ends too. The thing that *didn't* work is often the most useful line in the file.
- Every rollback point gets recorded when it's created, not when I need it.
- Append as I go. If I'm writing it up at the end of the session, it's already too late to be accurate.

**Entry format:**

```
## YYYY-MM-DD — <what changed>
**Phase:**       <A / A.5 / B / ... / pre-A / E>
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

**What actually worked — `Ctrl+E` during POST.** The R710 shows a five-second window at POST offering "Remote Access Setup." That opens the iDRAC6 Configuration Utility, separate from F2's System Setup. Reset iDRAC configuration to defaults from there.

**After the reset:**
- Set iDRAC to DHCP; created a DHCP reservation on the home router → `192.168.0.100`
- Logged in with factory defaults, changed the password immediately — a freshly-reset iDRAC on factory credentials is an open door on the network
- Recorded IP + username + password together in one place (Keeper)

### Outcome
iDRAC confirmed reachable and working at **`192.168.0.100`**. Console access to `pve01` now exists independently of network config — the prerequisite for touching anything network-related.

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

**Created `vmbr1` on NIC1 (`eno1`) via the Proxmox GUI.** First attempt failed: Proxmox rejected the Create with *"default gateway already exists on interface vmbr0."* Proxmox allows only one default gateway per host. Left the Gateway field blank — and the IP field blank too, deliberately, to avoid a duplicate-IP conflict before the cable had moved.

**Moved the physical cable from NIC2's port to NIC1's port.**

**`vmbr1` initially showed state DOWN.** Checked `ip link show` — `eno1` itself came up, and `vmbr1` followed once its member interface had carrier. Confirmed `eno1` was genuinely NIC1 rather than assuming the chassis label matched the Linux name.

**Assigned the IP via the local console** (monitor + keyboard, since neither bridge was reachable at that moment): edited `/etc/network/interfaces` to add the address/gateway to `vmbr1`, `ifreload -a`, confirmed via `ip addr show vmbr1` and `ip link show vmbr1`.

**Then: ping to 192.168.0.201 timed out anyway.** Everything looked correct — bridge UP, IP assigned, same subnet, cable in the same dumb switch.

**Root cause: `vmbr0` still held `192.168.0.201` too.** The earlier edit to comment out its address hadn't taken. Two bridges on the same host claiming the same static IP creates ARP ambiguity — the kernel doesn't know which interface should answer, so replies get dropped or sent from the disconnected bridge. **Silent failure, no error message anywhere.**

**Fix:** commented out `vmbr0`'s address line, `ip addr flush dev vmbr0` to clear it at the kernel level immediately, `ifreload -a`. Ping replied. Proxmox web UI loaded at `192.168.0.201:8006`.

**Second problem, immediately after: both existing VMs reported "network unreachable."** Root cause: both VMs' network devices were still attached to `vmbr0`, which now had no IP *and* no cable. Moving the physical cable did not move the VMs. Fix: shut down each VM, changed Bridge from `vmbr0` to `vmbr1` in the Hardware tab, booted. Both reached the internet again.

### Outcome
- **NIC1 / `vmbr1`** = Proxmox management, `192.168.0.201`. Confirmed working.
- **NIC2 / `vmbr0`** = no IP, no VMs attached. **Free — becomes pfSense WAN in Phase A.**
- Both existing VMs on `vmbr1`, internet-reachable, **not yet isolated**.
- NIC3 (pfSense LAN trunk) and NIC4 (spare/SPAN fallback) unconfigured.

### Lessons
1. **Duplicate IP across bridges = silent ARP failure.** Proxmox permits two bridges to hold the same static IP even with one physically disconnected. No error, no warning, pings just vanish. **Always check `ip addr show` on BOTH bridges after any reassignment.**
2. **Moving a cable or bridge orphans every VM still pointed at the old one.** The VM reports "network unreachable" with no obvious cause, because nothing about the VM changed. **Check every VM's Hardware tab after any bridge change.**
3. Proxmox allows exactly one default gateway per host — a second bridge on the same subnet doesn't need its own.
4. Create the bridge with **no IP**, move the cable, *then* assign the IP. Ordering avoids the duplicate-IP window entirely.
5. Don't assume the chassis port label matches the Linux interface name. Confirm with `ip link show`/`ethtool <iface>` and watch which one loses carrier when you unplug it.
6. **Both of these were found by console access, not by SSH** — exactly the situation the recovery ladder exists for. The iDRAC recovery the day before is what made this change safe to attempt.

---

## 2026-07-17 — Identifying the Cisco switch

**Phase:** A
**Goal:** Identify the Cisco switch before writing any command for it — model, IOS version, real interface names.
**Rollback:** None needed — `show` commands are read-only.
**Transcript:** PuTTY logging, set before connecting.

### What happened
`enable`, `show version`, `show interfaces status`.

### Outcome
Model: WS-C2960X-48FPS-L, IOS: 15.2(7)E9. Interface naming: `GigabitEthernet1/0/1` through `1/0/52` (shorthand `Gi1/0/1`), plus a separate `Fa0` management port.

**Port summary** (pre-config baseline):
- `Gi1/0/1`, `Gi1/0/48-52`: VLAN 1 (default)
- `Gi1/0/2-47`: VLAN 10 (leftover from a prior config, wiped in the factory reset below)
- `Gi1/0/49-50`: err-disabled (later confirmed hardware-faulty)
- `Gi1/0/51-52`: "Not Present" (unpopulated SFP/stacking ports)
- All ports notconnect — nothing cabled yet.

### Lessons
*(none — read-only identification step; see the factory-reset entry below for the actual config change)*

---

## 2026-07-17 — Cisco switch factory reset

**Phase:** A
**Goal:** Wipe the switch to factory defaults, confirm clean console access, before any VLAN/trunk config.
**Rollback:** None needed — intentional, planned reset.
**Transcript:** PuTTY, COM14.

### What happened
- Identified the switch first (see above): WS-C2960X-48FPS-L, IOS 15.2(7)E9.
- Pre-reset `show interfaces status` showed prior config: hostname Lab-Switch, most ports in VLAN 10, ports `Gi1/0/49-50` err-disabled.
- `write erase` completed cleanly.
- First `reload` attempt: answered `[confirm]` with "n" instead of Enter, which **canceled** the reload — NVRAM was erased but the switch kept running the old config in RAM. Caught this before assuming the reload had happened.
- Re-ran `reload`, pressed Enter at `[confirm]` this time. Reboot completed. Declined the initial config dialog.
- Session log has garbage lines from an accidental clipboard paste (right-click in PuTTY pastes clipboard directly; a chat response was on the clipboard at the time) — the switch correctly rejected all of it as invalid input, no actual config impact. Confirmed via `show run`.
- Verified clean state: `enable` required no password, `show run` showed hostname "Switch" (default), no VLAN assignments, no err-disabled state, all 52 Gi ports at default, no aaa/line passwords configured.
- Ran `write memory`, confirmed saved config matched the verified clean running-config exactly.
- Powered off the switch normally at an idle prompt.

### Outcome
Switch factory reset confirmed clean, saved to NVRAM, powered down safely. Console access confirmed working post-reset.

### Lessons
1. On Cisco, `[confirm]` prompts want Enter — any other keystroke (including "n") **cancels** the action rather than confirming "no." Different from a `[yes/no]` prompt, which takes a typed answer.
2. Right-click in PuTTY pastes the OS clipboard directly as keystrokes with no preview. Check clipboard contents before right-clicking, or use left-click-drag to select+copy just the intended text.

---

## 2026-07-21 — NIC troubleshooting + `vmbr3` creation (Phase A step 3)

**Phase:** A
**Goal:** Identify NIC3 physically, create a trunk-capable bridge for pfSense LAN.
**Rollback:** None needed — no destructive changes; only bridge creation (additive) and interface up/down toggles.
**Transcript:** console + SSH, this date.

### What happened
- Chassis port labels were confirmed NOT self-evident from `ip link show` alone — had to physically trace cables to labels. Confirmed the management NIC both by physical inspection and behaviorally (unplugging it dropped the SSH/management session).
- The other three NICs all showed `state DOWN`/`Link detected: no` across multiple cables, multiple switch ports, and even a direct PC-to-server connection — appeared to rule out cabling entirely.
- Extensive troubleshooting followed: `lspci`/`dmesg` (confirmed all 4 NICs detected cleanly by the kernel, no driver errors), BIOS Integrated Devices check (all 4 ports shown Enabled), iDRAC Lifecycle Log/SEL review (no explicit NIC fault logged, though real historical CPU/memory/IO fault entries exist from earlier in the year — unrelated to this issue as it turned out).
- **Root cause, finally found:** the other three NICs were simply never administratively brought up (`ip link show` showed no `UP` flag). Physical hardware was never at fault. Fix: `ip link set <iface> up` on each — all three immediately showed `LOWER_UP` and clean `1000Mb/s Full duplex` link once brought up.
- Created `vmbr3` via the Proxmox GUI: bridge-port on NIC3, no IP, VLAN-aware enabled (`bridge-vids 2-4094`). Applied via GUI, took effect live, no reboot needed. Confirmed written to `/etc/network/interfaces` (persistent).
- Naming: chose `vmbr3` (matching NIC3) instead of the "next sequential" `vmbr2`, a deliberate one-off exception to keep NIC-to-bridge naming legible, despite `vmbr0`/`vmbr1` already being a mismatched, unfixable-without-risk precedent from earlier setup.

### Outcome
All 4 physical NICs confirmed healthy. `vmbr3` created, VLAN-aware, ready for pfSense's LAN interface in the next step.

### Lessons
1. **Check administrative interface state (`ip link set <iface> up`) BEFORE any hardware-level troubleshooting.** A `DOWN` interface with no `UP` flag looks identical to a real hardware fault and cost significant time to diagnose here. This is now the FIRST thing to check on any "port shows no link" problem.
2. Chassis port labels are not reliable without physical confirmation — verify by unplug/replug + `ip link show`, not by assuming left-to-right ordering matches interface numbering.
3. Bridge names don't have to match NIC numbers, and neither convention is objectively correct — but once one convention exists, decide deliberately rather than defaulting to "next sequential number" without thinking about it.

---

## 2026-07-21 — `vmbr0`→`vmbr2` swap, missing gateway fix, package upgrade + reboot

**Phase:** A
**Goal:** Rename NIC2's bridge to match the NIC-number convention; discovered and fixed a missing default gateway in the process; upgraded and rebooted `pve01`.
**Rollback:** None needed — `vmbr0` had no IP/no VMs attached (safe to delete); gateway addition tested live before persisting; package upgrade left prior kernels installed and selectable in GRUB as fallback.
**Transcript:** console + SSH, this date.

### What happened
- Deleted `vmbr0` via the Proxmox GUI. Confirmed removed both live and in `/etc/network/interfaces`.
- Created `vmbr2` on NIC2: no IP, VLAN-aware left unchecked (this is pfSense's planned WAN side, not a VLAN trunk). Confirmed UP, no IPv4, present in the config file.
- While troubleshooting `apt update` failures (all repos "Network is unreachable"), found `ip route show` had no default route — only the local subnet route via `vmbr1`. `/etc/network/interfaces` confirmed no `gateway` line existed anywhere, despite an earlier log entry stating one was added. Root cause of the discrepancy not determined — either removed at some point, or the original apply never fully completed.
- Before editing `vmbr1` (the live management interface): confirmed iDRAC and monitor+keyboard fallback access, confirmed session recording on.
- Fix: added `gateway 192.168.0.1` under `vmbr1`'s `iface` stanza, `ifreload -a`. Verified live: `ip route show` showed the new default route, `ping 192.168.0.1` and `ping 8.8.8.8` both succeeded, SSH session stayed connected throughout.
- Ran `apt update` — succeeded once the gateway was fixed. Reviewed `apt list --upgradable` before proceeding: two kernel packages, `pve-manager`, `qemu-server`, `pve-firewall`, `pve-container`, ZFS packages, standard Debian libs — no unexpected removals.
- Chose not to snapshot before the upgrade (`pve01` is the bare-metal host, not a VM — `qm snapshot` doesn't apply to the host itself; confirmed prior kernels would remain selectable in GRUB as the actual fallback).
- Ran `apt upgrade`, confirmed at the prompt. All packages + 2 new kernel packages installed cleanly, no errors, no interactive config-conflict prompts. GRUB regenerated with 5 kernel entries available.
- Confirmed both existing VMs were `stopped` before reboot — no running sessions to interrupt.
- Rebooted. Confirmed post-reboot: new kernel active, `vmbr1`/`192.168.0.201` up, default route persisted, ping to `8.8.8.8` succeeded — gateway fix and bridge changes both survived reboot cleanly.

### Outcome
- NIC2/`vmbr2` = no IP, no VMs, ready for pfSense WAN.
- NIC1/`vmbr1` = management, `192.168.0.201/24`, now with a working default gateway, internet-reachable, confirmed persistent across reboot.
- `pve01` fully patched.
- Current bridge state: `vmbr1` (NIC1, management), `vmbr2` (NIC2, pfSense WAN, unconfigured), `vmbr3` (NIC3, pfSense LAN trunk, VLAN-aware, unconfigured), NIC4 unbridged.

### Lessons
1. **A "no default gateway" symptom can look identical to a DNS or firewall problem** (`apt update` failing on every repo) — check `ip route show` for a default route early when package-manager connectivity fails entirely, not just DNS resolution.
2. **A prior log entry stating a change was made is not the same as confirming it's still true.** An earlier entry documented adding a gateway to `vmbr1`, but it was absent from the live config weeks later with no record of removal. Verify current state against the live system before trusting a past log entry.
3. `qm snapshot` only covers VMs, not the bare-metal host — there's no built-in Proxmox mechanism to snapshot the host's own OS/filesystem. The actual fallback for a host-level package upgrade is confirming prior kernels remain selectable in GRUB, plus the standard recovery ladder.
4. Before any host-level `apt upgrade` involving a kernel: confirm all VMs' running state first — a reboot's impact scope depends entirely on what's actually running, not on what's configured to autostart.

---

## 2026-07-23 — VLAN creation, trunk to pfSense, management access port (Phase A Steps 4-6)

**Phase:** A
**Goal:** Create the three VLANs, configure the trunk port to pfSense (NIC3/`Gi1/0/3`), and put the management PC's port into the Management VLAN.
**Rollback:** None needed for VLAN creation (additive). Config saved to NVRAM only after each step's output was confirmed correct.
**Transcript:** PuTTY (switch, COM14), this date.

### What happened

**Pre-check — `show run` reviewed before touching anything.** Confirmed the switch still matched the clean, factory-reset baseline: hostname `Switch`, no VLANs beyond default `Vlan1`, no `aaa new-model`, no passwords on `con 0`/`vty` lines, all 52 `Gi1/0/1`-`1/0/52` ports present and unconfigured.

**Step 4 — created VLANs 10 (MGMT), 20 (INFRA), 30 (RANGE).** `show vlan brief` confirmed all three as `active`, no ports assigned yet. Clean on the first attempt.

**Step 5 — trunk port to pfSense (`Gi1/0/3`).** Identified which physical port NIC3 was actually cabled into via `show interfaces status` — `Gi1/0/3` was the only port showing `connected`, matching NIC3. Configured trunk mode with allowed VLANs restricted to 10,20,30.
- `switchport trunk encapsulation dot1q` was rejected (`% Invalid input detected`) — expected, this 2960X only supports 802.1Q natively.
- **First pass missed the `switchport trunk allowed vlan 10,20,30` line** — appears to have been dropped from a paste. `show interfaces trunk` still showed `trunking`/`802.1q` correctly even with it missing, but "Vlans allowed on trunk" read `1-4094`, not `10,20,30` — caught on closer inspection. Re-ran the missing line, confirmed correct afterward.
- Interface flapped twice — transient STP renegotiation, self-resolved within ~30 seconds each time. No action needed.

**Step 6 — management PC's access port.** Original plan was to use `Gi1/0/48` directly, but investigation revealed the PC's cable was actually running through the home dumb switch as a shared uplink, not a dedicated line. Putting `Gi1/0/48` into VLAN 10 access mode as planned would have pulled the entire dumb switch (and everything behind it, including possibly the home router) onto VLAN 10. Decided against consolidating the dumb switch into the managed switch for now, to avoid stacking a second, larger network change on top of an unverified one.

**Fix:** used one of the two USB-to-Ethernet adapters (purchased for later SPAN duty, borrowed for this in the meantime) to run a dedicated point-to-point link directly into `Gi1/0/48`, bypassing the dumb switch entirely. The PC's onboard NIC stays on the dumb switch/home network for normal internet; the USB adapter carries only lab management traffic.

Also discovered during port selection: `Gi1/0/49` and `Gi1/0/50` still showed `err-disabled` even after the full factory reset. Since `err-disabled` normally requires an active runtime trigger (port security, BPDU guard, storm-control) that a factory erase wipes clean, both coming back err-disabled with no config present points to a **hardware-level fault**, not leftover config. Decided to avoid these two ports going forward.

With the dedicated link in place: configured `Gi1/0/48` as access mode, VLAN 10. `show vlan brief` confirmed it listed under VLAN 10.

**Saved config:** `copy running-config startup-config` → confirmed written to NVRAM.

### Outcome
- VLANs 10/20/30 created and active.
- `Gi1/0/3` trunking correctly scoped to `10,20,30` only.
- `Gi1/0/48` in VLAN 10 access mode, a dedicated point-to-point link from the PC (via USB-Ethernet adapter).
- Switch config saved to NVRAM.
- `Gi1/0/49`/`Gi1/0/50` flagged as likely hardware-faulty.
- One USB-to-Ethernet adapter now in use for management access — remember to free it or confirm a second unit before SPAN duty needs it.
- The pfSense VM itself still not built — bridges exist, but no VM attached to them. Next step, ahead of creating pfSense's own VLAN interfaces.

### Lessons
1. **A missing line in a multi-line paste can silently succeed** — the trunk showed success even with a critical line missing, because "no errors" and "matches the intended config" are not the same check. Always compare *every* relevant field, not just whether the command was accepted.
2. **`end` typed at the top-level EXEC prompt (not inside a config sub-mode) is not a no-op** — IOS tries to resolve it as a hostname/telnet target instead, producing a DNS-lookup failure. Harmless, but confusing. Only use `end` to exit a config sub-mode.
3. **A "management port" assumption should be physically verified, not taken on faith** — the PC's cable was assumed dedicated but was actually a shared dumb-switch uplink. Confirmed before applying the VLAN assignment, avoiding an accidental home-network-wide VLAN 10 membership.
4. **Ports staying `err-disabled` after a full factory erase is a hardware-fault signal**, not a config leftover — `err-disabled` requires an active trigger condition to persist, and factory erase removes all such conditions.

---

## 2026-07-24 — pfSense VM built and installed (Phase A Step 3)

**Phase:** A
**Goal:** Build the pfSense VM on `pve01`, attach it to the two existing bridges (WAN, LAN trunk), and get pfSense CE actually installed and booted.
**Rollback:** None needed — new VM, no prior state. ISO confirmed present before starting; bridges re-verified live rather than trusted from the log.
**Transcript:** console/web UI + PuTTY (switch, for pre-checks), this date.

### What happened

**Pre-checks before touching Proxmox:**
- Confirmed the Netgate Installer ISO was present. Note: Netgate no longer ships version-pinned pfSense ISOs directly — the "Netgate Installer" is a bootstrap image whose own version is unrelated to the pfSense version installed; the actual release is chosen later, inside the installer, once it has internet access.
- Re-verified the WAN/LAN-trunk bridges live rather than trusting the prior log entry: both initially showed `NO-CARRIER`/`DOWN` — traced to the Cisco switch simply being powered off since last session, not a real fault. Powered the switch back on; both bridges came up correctly, matching the switch's `connected` status. Confirmed the LAN-trunk bridge's VLAN-aware flag genuinely active at the kernel level. Both bridges confirmed no IPv4 address.

**VM creation (VM 102, named `pfsense`):**
- OS: "Other" guest type, ISO = the Netgate installer bootstrap image.
- Disk: 32GB, VirtIO SCSI.
- CPU/RAM: started at 2 cores/2GB; **bumped to 3 cores/4GB mid-install** because first-boot resource usage pegged out and installation was crawling. Flagged as a TODO to revisit once past first boot.
- Network: `net0` → WAN bridge, `net1` → LAN-trunk bridge — deliberately not left on the wizard's default management bridge. Both NICs VirtIO, no VLAN tag (the trunk needs to pass VLANs untagged at this layer; pfSense creates the actual VLAN sub-interfaces later), Proxmox firewall checkbox left unchecked on both (pfSense is meant to be the sole firewall).
- Confirmed MAC pairing via the Hardware tab before booting, for an authoritative cross-check during the installer's interface-selection screens.

**Installer walkthrough:**
- WAN interface confirmed via MAC match → DHCP client, VLAN tagging disabled (correct, WAN isn't a trunk).
- LAN interface confirmed via MAC match and independently corroborated by the switch already showing `Gi1/0/3` connected → left at factory-default static, DHCPD enabled, VLAN tagging disabled on this parent interface (correct, VLAN sub-interfaces get created on top later).
- **Internet connectivity check failed** — WAN had no physical cable yet. The Netgate Installer requires live internet to fetch pfSense packages (not bundled in the small bootstrap ISO). Fix: plugged NIC2's physical port into the dumb switch, confirmed link. Retried — WAN pulled a DHCP lease, connectivity check passed.
- Subscription validation: device has no active Plus subscription, as expected → selected Install CE.
- Software version: 2.8.1 (current stable) — matched what was confirmed via web search beforehand.
- Filesystem: ZFS, Stripe (only option for a single virtual disk), GPT partitioning. All defaults, all appropriate.
- Disk selection: single 32GB virtual disk, confirmed, destroyed/formatted (empty disk, no data at risk).
- Installation completed cleanly.
- **Halted (not rebooted) before removing installer media** — detached the ISO from the CD/DVD Drive to avoid booting back into the installer. Proxmox needed a manual Stop after the guest's internal halt completed, since a FreeBSD-based halt doesn't power off the VM at the hypervisor level automatically.
- **First real boot succeeded:** pfSense 2.8.1-RELEASE amd64, "Bootup complete." Console menu confirmed both interfaces live: WAN via DHCP from the dumb switch, LAN at the factory-default static address (to be replaced by VLAN interfaces next).

### Outcome
- pfSense CE 2.8.1 installed and running, attached to both bridges correctly.
- WAN's physical cable placement matches the blueprint's intended final destination — no cable move needed later.
- **Outstanding TODO:** VM resources bumped to 3 cores/4GB mid-install to push through a slow first boot — revisit once steady-state load is known.
- pfSense's web GUI not yet reachable from the management PC — LAN currently only knows the factory-default subnet, not the real VLAN subnets. That gap closes next.

### Lessons
1. **The Netgate Installer needs internet access on WAN to complete installation** — it's a bootstrap image that fetches the actual pfSense packages live. Plan for WAN to have real connectivity before starting a pfSense install.
2. **A build_log entry is a snapshot, not a guarantee** — the bridges showing down today wasn't a config regression, just the switch being powered off since last session. Re-verifying live state before trusting it caught this immediately.
3. **VM resource allocation may need temporary headroom for first boot/install**, separate from steady-state requirements.
4. **MAC address cross-checking between Proxmox's Hardware tab and the guest OS's interface-selection screen is a reliable way to confirm mapping**, more trustworthy than assuming numbering order alone.

---

## 2026-07-25 — pfSense VLAN interfaces, setup wizard, first snapshot (Phase A Step 5, acceptance check passed)

**Phase:** A
**Goal:** Create the three VLAN sub-interfaces inside pfSense, assign them real subnets, confirm the web GUI is reachable from the Management VLAN, and take a clean rollback snapshot before any firewall rules exist.
**Rollback:** Snapshot `pfsense-clean-install` taken at the end of this session, VM running (RAM not included) — first real rollback point for this VM.
**Transcript:** Proxmox VM console (pfSense) + browser (pfSense GUI), this date.

### What happened

**VLAN sub-interfaces created** via the console menu, configuring VLANs on the LAN parent interface: tag 10 (Management), tag 20 (Infra), tag 30 (Range). Confirmed WAN/LAN/OPT1/OPT2 mapping before applying.

**IP assignment** via the console menu, once per interface — same pattern each time (static, no upstream gateway on any of the three, IPv6 skipped, DHCP server enabled, kept HTTPS for the web configurator):
- LAN (Management): `10.10.10.1/24`, DHCP range `10.10.10.100`–`10.10.10.199`
- OPT1 (Infra): `10.10.20.1/24`, DHCP range `10.10.20.100`–`10.10.20.199`
- OPT2 (Range): `10.10.30.1/24`, DHCP range `10.10.30.100`–`10.10.30.199`

**Client-side confirmation:** the management PC's USB-Ethernet adapter picked up a DHCP lease on the first attempt — `10.10.10.100/24`, gateway `10.10.10.1`. Browsed to `https://10.10.10.1/` and reached pfSense's login page (self-signed cert warning, expected/accepted). **This is Phase A's acceptance check, passed.**

**Logged in** with default credentials, forced password change immediately.

**Ran the pfSense setup wizard** (auto-prompted on first login):
- General info: hostname/domain left as the already-correct defaults.
- NTP: kept the default time server.
- **WAN configuration — one real fix required here:** "Block RFC1918 Private Networks" was checked by default. **Unchecked it.** This build's WAN doesn't face a real ISP — it faces the home network, itself an RFC1918 range — so leaving this checked would have caused pfSense to block its own WAN's legitimate upstream traffic as if it were spoofed. Left "Block bogon networks" checked, still correct regardless of WAN's private/public nature.
- LAN configuration screen: showed the already-configured Management subnet correctly, confirming the wizard picked up existing state rather than overwriting it.
- Reloaded configuration, wizard completed cleanly.

**Snapshot taken:** `pfsense-clean-install`, VM running, RAM not included (unnecessary for a config checkpoint). Confirmed via the Snapshots tab.

### Outcome
- **Phase A is now fully complete.** All six steps done: VLANs created, trunk configured and scoped correctly, management PC's access isolated via a dedicated USB-adapter link, pfSense VM built and installed, VLAN interfaces created and IP-addressed, GUI reachability confirmed from the Management VLAN.
- pfSense reachable at `https://10.10.10.1/` from any device on the Management VLAN.
- **First rollback point for the pfSense VM exists** (`pfsense-clean-install`).

### Lessons
1. **"Block RFC1918 Private Networks" on WAN is the wrong default for any home-lab topology where WAN's upstream is itself a private network** — a genuine pfSense wizard default that needs deliberate correction here.
2. **The pfSense setup wizard re-displaying already-configured values rather than blank defaults is a good sign**, not a coincidence — confirms the wizard reads and preserves existing config rather than silently resetting it.
3. **A live VM snapshot (no RAM) is sufficient for a configuration checkpoint** — no need to stop the VM or capture memory state when the goal is "return to this config later."

---

## 2026-07-30 — Range isolation rule written and applied (Phase A.5)

**Phase:** A.5
**Goal:** Write and apply the pfSense Range-VLAN isolation rule — deny all outbound from Range, with one explicit allow to Infra's Wazuh ingest port.
**Rollback:** Proxmox snapshot `pre-isolation-rule` (VM 102) taken from the `pve01` shell, confirmed via the logical volume creation message. pfSense config also exported as XML and saved locally.
**Transcript:** session recording on throughout.

### What happened

**Gate items confirmed before any firewall change:**
1. Fresh Proxmox snapshot `pre-isolation-rule` taken on VM 102 — succeeded cleanly.
2. pfSense XML config exported and saved locally, opened to confirm it wasn't empty.
3. Session recording confirmed on.

**Interface labels renamed for clarity** (cosmetic only): LAN → MGMT10, OPT1 → INFRA20, OPT2 → RANGE30, applied once after all three renames. Firewall > Rules tabs picked up the new names automatically.

**Rule 1 — the allow, built on the RANGE30 tab:** Pass, IPv4, TCP, source RANGE30 subnets, destination INFRA20 subnets (placeholder — Wazuh doesn't exist until Phase B; to be tightened to a single host IP once built), port 1514, description `allow range -> wazuh agent ingest ONLY`.

**Rule 2 — the default deny, built second:** Block, RANGE30 subnets → any, description `default deny - range is isolated`.

Rules were built and read back field-by-field using the Claude in Chrome extension in a strict "point/describe only, never click or type" mode — the extension highlighted each field and stated the value to enter; every field was typed in by hand and confirmed via screenshot before Save.

**Reordering gotcha, caught live:** dragged Rule 2 to confirm its position relative to Rule 1. Discovered that a drag-to-reorder is **not committed by the drag alone** — clicking Apply Changes immediately after a drag, without an intervening Save on the rule list, triggered pfSense's own "unsaved changes" warning. Canceled out of that warning, clicked Save first (which committed the new order), *then* clicked Apply Changes. Confirmed the browser-level warning is a real safety net here, not just a mechanical prompt.

**Applied successfully:** pfSense confirmed the rules were reloading, order preserved on reload.

**Post-apply sanity check:** confirmed continued GUI access from MGMT10 immediately after applying — the rule set only targets RANGE30, but verified rather than assumed.

### Outcome
- Both rules live on the RANGE30 interface: Pass (RANGE30→INFRA20 subnets, TCP 1514) above Block (RANGE30→any, any).
- pfSense GUI access from MGMT10 confirmed unaffected.
- **Rule exists and is correctly scoped, but isolation is not yet acceptance-tested live** — the target VMs are still on the flat network, not yet moved to the Range VLAN. That move is Phase B. The real "no internet, no home net, no MGMT, only Wazuh:1514" test can't run until a VM actually sits on RANGE30.
- Destination on Rule 1 (INFRA20 subnets) is intentionally broader than the final design (a single Wazuh host) — revisit once Wazuh is deployed.

### Lessons
1. **A drag-to-reorder in pfSense's rule list is not committed until Save on the rule list itself is clicked** — sequence going forward: drag → Save → Apply Changes.
2. **Renaming interface descriptions is purely cosmetic** but meaningfully reduces the chance of picking the wrong tab under pressure.
3. **A rule being applied cleanly is not the same as the isolation being proven.** Don't mark this phase fully complete until the acceptance check has actually run against a real VM.

---

## 2026-07-31 — Ubuntu Server VM built (Phase B Step 1), Infra outbound rules written and live-tested

**Phase:** B (Step 1)
**Goal:** Build the Ubuntu Server VM on `pve01`/Infra VLAN for the Docker/Wazuh substrate; enable OpenSSH. Along the way, write and verify the Infra outbound internet rule that had been sitting unresolved.
**Rollback:** pfSense config exported as XML before any firewall rule changes. No Proxmox snapshot taken — additive rule changes only, no risk to MGMT10 access. VM 103 itself is new-build; no rollback needed pre-install.
**Transcript:** console (VM 103) + pfSense GUI via Claude in Chrome + PowerShell (SSH), this date.

### What happened

**ISO selection.** Confirmed Ubuntu 26.04 LTS was current, but deliberately chose **24.04.4 LTS** — most Wazuh/Docker Compose guides are still written against 24.04, and 26.04 is only ~3 months old. First upload attempt was actually the Desktop ISO by mistake — caught before VM creation; the correct Server ISO uploaded and confirmed.

**VM 103 (`ubuntu-soc-host`) created:** 8GB RAM (ballooning disabled, min=max), 100GB disk (VirtIO SCSI, `local-lvm`), network tagged VLAN 20 (Infra). Two confirm-screen catches before Finish: cores/sockets initially inverted, fixed; CPU type defaulted to a specific x86-64 variant instead of `host`, caught and re-applied correctly on retry.

**Installer walkthrough:** "Try or Install Ubuntu Server" (GA kernel), full "Ubuntu Server" install (not minimized), no third-party drivers, no Ubuntu Pro, no featured server snaps (Docker via the apt-repo method later per plan, not snap/curl|sh).

**Network config initially failed** ("autoconfiguration failed") — root cause was simply that pfSense wasn't powered on. Confirmed the nightly full shutdown of `pve01` + the switch is a current standing habit; flagged (not resolved) that this conflicts with the "always-on service host" design intent for later continuous-monitoring phases. Powered switch → `pve01` → pfSense back up in order; DHCP succeeded.

**Mirror test failed** next: DNS resolution failure to the Ubuntu mirror. Root cause: INFRA20 (OPT1) had zero firewall rules — unlike RANGE30 (explicit rules from Phase A.5), OPT interfaces get no automatic allow rule the way the LAN-role interface does. Implicit deny blocked everything on that interface, including DNS queries to pfSense's own resolver — DHCP worked because it's handled below the packet-filter layer, but nothing else was. This is exactly the "Infra VLAN outbound internet rule" item that had been sitting open.

**Wrote and applied 3 Pass rules on INFRA20** (config exported first): DNS (TCP/UDP 53), HTTP (TCP 80), HTTPS (TCP 443), all scoped INFRA20 subnets → any. Chose scoped-by-port over blanket-allow, consistent with the least-privilege pattern used elsewhere. Built via the Claude in Chrome point-and-describe workflow, one rule at a time. Two description-field mixups along the way — caught on screenshot review and fixed manually. New standing convention adopted from here forward: rule descriptions read `allow <INTERFACE_NAME> -> <PURPOSE> (<PORT>)`.

**Retried the mirror test — hung twice**, ~24 minutes unresponsive each time, right after a small file fetch succeeded but before the actual larger package index came through. Diagnosed via the installer's built-in debug shell: DNS resolved fine, a direct HTTP request succeeded instantly, but ping produced no result. Root cause: ICMP was completely unhandled on INFRA20. The three rules written covered TCP/UDP 53/80/443 only, no ICMP — meaning Path MTU Discovery had no way to signal "fragmentation needed" back to the client. Small single-packet transfers worked fine; anything requiring an oversized packet along the path hung indefinitely.

**Added a 4th rule — first attempt set Protocol to IGMP by mistake** (not ICMP) — caught reviewing the full rule list screenshot post-apply, corrected, re-applied. Confirmed working: the mirror test passed on retry with a full package-index fetch.

**Console input became unreliable** immediately after (a browser console session split every keystroke into its own line) — unable to get a clean confirmation in-shell. Rather than keep fighting it, did a hard Stop/Start power cycle on VM 103 (no data at risk — pre-storage-configuration). Came back up cleanly at the same point.

**Storage:** "Use an entire disk" + LVM (chosen over plain partitioning for future resize flexibility). LUKS encryption explicitly skipped — this VM's operating pattern (nightly full power-cycle) would mean manually entering a passphrase every morning before Docker/Wazuh could even start; the threat model doesn't justify that recurring friction for a home lab. Caught a default-guided-partitioning gap: the installer's guided layout only allocated about half of the volume group to the root filesystem, leaving the rest stranded as unallocated free space. Manually edited the logical volume to the full size, confirmed zero free space remaining before proceeding.

**Profile:** hostname `wazuh-host`, username matching the existing convention. OpenSSH server install checked, password auth left enabled (no key imported at install time; hardening to key-only auth is a follow-up, not done tonight).

**Install completed** — `openssh-server` installed, security updates pulled successfully (a second real-world confirmation the INFRA20 rules work, this time for actual package traffic rather than just the mirror test).

**Mistake, mine:** instructed ejecting the ISO from the VM's CD/DVD drive before the installer's final "Reboot Now" prompt actually appeared, rather than after. The live installer environment's own root filesystem was mounted from that ISO — pulling it while still live caused unmount failures and a stuck shutdown (~15 minutes), followed by a second failed reboot attempt, since the live environment had no working root filesystem left to shut down properly. No actual damage — partitioning, bootloader install, and package installs had already fully completed before this happened; only the dying live environment was affected. Recovered with a hard Stop/Start (CD/DVD already set to no media) — booted cleanly straight into the installed OS.

**Confirmed working:** console login succeeded; the expected IP showed on the interface as expected; SSH from PowerShell connected without the KexAlgorithms workaround `pve01` needed. Host key fingerprint manually verified against the one printed in the VM's own boot log before accepting (matched exactly). Session ended with a graceful shutdown (not a hard stop, since this is now a real installed OS).

### Outcome
- **VM 103 (`ubuntu-soc-host`/hostname `wazuh-host`)** built and fully installed: Ubuntu Server 24.04.4 LTS, 8GB RAM/4 cores/host CPU, 100GB disk (single LVM volume, no leftover free space, no encryption), on the Infra VLAN.
- **SSH confirmed working**: password auth. Key-only auth hardening still open for later.
- **INFRA20 now has 4 firewall rules**: DNS (53), HTTP (80), HTTPS (443), ICMP (any subtype) — all INFRA20 subnets → any, scoped rather than blanket-allow. This closes the "Infra VLAN outbound internet rule" item, now live-tested by a real OS install rather than just theoretical.
- **Phase B Step 1 is complete.** Step 2 (host prep) is next.
- **Still open, unchanged:** `pve01`/switch nightly shutdown vs. the blueprint's always-on-service-host design — flagged, not resolved.

### Lessons
1. **OPT interfaces in pfSense get zero rules by default** — only the LAN-role interface gets an automatic allow-all. Any new VLAN/segment beyond the original three needs its outbound rule written explicitly before assuming internet reachability.
2. **DNS-resolution-only failures and full-hang-after-partial-success are different symptoms pointing at different rule gaps.** The first meant no rules existed at all; the second — succeeding on small transfers, hanging on larger ones — specifically means ICMP/PMTUD is missing, not DNS or the main port itself.
3. **A live installer's root filesystem may still be the mounted ISO, even well into the install process.** Don't eject virtual media until the installer's own explicit "Reboot Now" (or equivalent) prompt appears.
4. **Guided "entire disk" LVM layouts don't always use the full disk by default** — they can conservatively split it, leaving real leftover space unallocated unless manually resized up to the max before finishing storage configuration.
5. **A stuck/glitchy remote console is often better solved by a clean power-cycle than by fighting the session** — especially pre-storage-configuration, where there's nothing to lose.

---

## 2026-08-03 — Phase B Steps 2-4 complete: host prep, Docker, Git/SSH + repo clone; INFRA20 gains a 5th rule; docx workbooks retired

**Phase:** B (Steps 2, 3, 4)
**Goal:** Finish host prep on `wazuh-host` for Wazuh's indexer, install Docker + Compose, install Git and get the repo cloned onto the box with its own SSH deploy key. Along the way: fix a new INFRA20 gap (port 22), retire the docx workbooks in favor of the `.md` files as sole source of truth, and rewrite `PROJECT-INSTRUCTIONS.md` in first person.
**Rollback:** No Proxmox snapshot — all changes are additive. No risk to MGMT10 or existing services.
**Transcript:** SSH (PowerShell → `wazuh-host`), pfSense GUI via Claude in Chrome, GitHub web UI, PowerShell (repo work on the management PC).

### What happened

**Step 2 — host prep.** Set `vm.max_map_count=262144` live, then persisted it to `/etc/sysctl.conf` via `echo "..." | sudo tee -a` — confirmed necessary because a plain `sudo echo ... >> file` would fail: the shell sets up `>>` redirection using the calling user's own permissions *before* `sudo` ever runs, so only `tee` (itself elevated by `sudo`) can actually write to a root-owned file this way. Verified the line landed in the file, not just echoed to the terminal.

**Step 3 — Docker + Compose, official apt method.** Ran the defensive removal of six potentially-conflicting packages — all six came back "not installed," a clean baseline. Added Docker's GPG key and the official repo, scoped to the correct architecture/codename rather than hardcoding either. `apt-get update` confirmed the new repo was actually being read. Installed the Docker CE package set — clean install, `docker.service`/`docker.socket` both enabled via systemd on install. Verified via `systemctl status docker` and `docker run hello-world` (a real test of INFRA20's HTTPS rule against Docker Hub, not just apt mirrors). Added the primary user to the `docker` group for non-root usage — flagged explicitly that this is functionally equivalent to root access on this box, acceptable here since it's a single-user lab box with existing sudo access. Required a full SSH session restart (group membership is read at login, not live) — confirmed via `groups` and a passwordless `docker run hello-world` retry.

**Step 4 — Git, identity, SSH, clone.** Git was already present (2.43.0, stock on 24.04). Set the Git identity — flagged the choice between a real email vs. GitHub's noreply alias before setting it, given this repo is public and framed for employers to browse; decided to keep the real email for this git identity specifically (kept out of this documentation set per the no-name/no-identifying-info rule adopted since).

Generated a fresh ED25519 keypair on `wazuh-host` itself, deliberately separate from `pve01`'s existing key. Added the public key to GitHub as a **repo-scoped Deploy Key** (not an account-level key) with write access enabled, since this box will eventually push Wazuh Compose files/config. Confirmed added correctly via the Deploy Keys screen.

**`ssh -T git@github.com` hung** on first attempt — diagnosed immediately as the same INFRA20 gap pattern from the DNS/ICMP issue during the OS install: port 22 (SSH) was never in the original four rules, so it fell to implicit deny with no response either way. **Added a 5th Pass rule to INFRA20** — TCP/22, description `allow INFRA20 -> SSH (22)` — same pattern as the prior four, no gate ceremony needed (purely additive, zero MGMT10 risk). One process hiccup building it: the Claude in Chrome point-and-confirm workflow briefly landed on the wrong tab and flagged the mismatch itself rather than guessing — caught before any wrong values were entered.

Retried `ssh -T git@github.com` — connected, host key fingerprint manually verified against GitHub's officially published fingerprint before accepting. Got the expected success message.

**Cloned the repo** via SSH — clean, no errors. Confirmed the file set matched the current repo state exactly.

**Docx workbook retirement (documentation cleanup, not lab infrastructure).** Discovered the repo's working folder is actually the repo root itself, not a subfolder holding old transcripts as previously assumed — confirmed via `git remote -v`. The rename itself caused zero git issues (git doesn't track its own containing folder's name). This corrected a standing misunderstanding about the repo layout carried since early sessions.

Reviewed all five `.docx` files in the workbook folder — decided explicitly, to reduce ongoing maintenance burden of keeping two documentation formats in sync, to **delete all five** and treat the four `.md` files as the sole source of truth going forward. Deleted the folder locally, used `git add -A` (correct tool for already-deleted-outside-of-git files) after confirming via `git status` that exactly the five expected files showed as deleted and nothing else. Committed and pushed.

**`agent-registry.md` replaced** with a real governance scaffold (previously a stale, mislabeled duplicate of `build_log.md` itself) — states its purpose, lays out planned scope for the future triage/routing agents ahead of when they're actually built, plus governing principles that apply to any future agent.

**`PROJECT-INSTRUCTIONS.md` rewritten in first person** throughout. Folded in several facts that had drifted out of sync: `wazuh-host` added to the VM list, INFRA20's firewall rules reflected, the `sudo tee` redirection gotcha, the pfSense rule-description naming convention, and the nightly-shutdown-vs-always-on-host tension moved into open decisions. Committed alongside the `agent-registry.md` replacement, message corrected via an amend after the first pass had a mismatched description.

### Outcome
- **Phase B Steps 2, 3, and 4 are all complete.**
- **INFRA20 now has 5 firewall rules**: DNS (53), HTTP (80), HTTPS (443), ICMP (any), and SSH (22) — all INFRA20 subnets → any.
- **Documentation set simplified**: `LAB-BLUEPRINT.md`, `PROJECT-INSTRUCTIONS.md`, `build_log.md`, `agent-registry.md` are now the entire documentation set — no more `.docx` companions.
- **Repo layout corrected** — the working folder is the actual repo root, not a subfolder — all future path references reflect this.
- **Phase B Step 5 is next** — deploying Wazuh via Docker Compose.

### Lessons
1. **Any new port/protocol beyond the original four INFRA20 rules needs its own explicit Pass rule** — same implicit-deny-on-OPT-interfaces story as before, this time SSH (22).
2. **`sudo command >> file` silently fails on root-owned files even though the command itself is elevated** — the shell sets up the redirect using the calling user's permissions before `sudo` ever runs. `echo "..." | sudo tee -a file` is the correct pattern.
3. **A folder's name inside a git repo, including the folder holding `.git` itself, is not tracked by git and can be renamed freely with zero repo-side consequences** — confirmed by testing directly rather than assuming.
4. **Deploy keys, not account-level SSH keys, are the right scope for a single-repo, single-purpose credential** — same least-privilege pattern used throughout this build.
5. **Maintaining parallel documentation formats (markdown + docx) doubles the update burden for no real information gain**, when the markdown files are already the actual source of truth being edited live.

---

## 2026-08-03 — Wazuh deployed via Docker Compose (Phase B Step 5, in progress)

**Phase:** B (Step 5)
**Goal:** Deploy Wazuh (manager + indexer + dashboard) via Docker Compose on `wazuh-host`, pinned to a known-current release tag; confirm all three containers healthy and talking to each other; change the default admin password.
**Rollback:** Not needed — new deployment, no prior state. The upstream `wazuh-docker` repo kept as a separate local-only clone, deliberately not folded into the tracked project repo.
**Transcript:** PowerShell + SSH session on `wazuh-host`, this date.

### What happened

**Version selection.** Confirmed via web search and cross-checked live against the repo's own tag list that `v4.14.6` was the current stable release — verified rather than trusted from the search result alone.

**Cloned `wazuh-docker`** to a local-only path on `wazuh-host` (deliberately separate from the tracked project repo — third-party deployment tooling, not something being authored). Checked out cleanly at the `v4.14.6` tag.

**Generated TLS certificates** via the certs-generator Compose file — root CA, admin, indexer, dashboard, and manager certs all created successfully. One cosmetic, non-fatal error in the script (a known quirk in the certs-generator image) — didn't stop cert generation, verified all 10 expected cert files landed on disk.

**Brought up the full stack** via `docker compose up -d` — all three images pulled cleanly, 14 volumes created, all three containers started with no errors.

**Verified health beyond "container started":**
- Indexer: a direct authenticated API call returned a valid cluster identity, confirming it was genuinely responding, not just running.
- Manager: logs showed a clean connection to the indexer, alert templates loaded, index-connector streams initialized, vulnerability scanner started — no errors, no restart loop.

**Logged into the dashboard** via Claude in Chrome in point-and-describe mode. Default credentials worked on the first try and landed directly on the Overview dashboard (0 agents registered, as expected). Alerts already showing with zero agents connected — these are Wazuh's own internal self-monitoring alerts, not real detections.

**Password change attempt failed as expected, for a real reason.** The dashboard's own "Reset password" dialog returned a "Resource reserved" error. Looked this up against current Wazuh docs: the `admin` indexer user is flagged `reserved: true` specifically to prevent password changes through the UI/API — the real password lives as a hash in `internal_users.yml`, and the Compose file carries a matching plaintext copy for the manager/dashboard containers to authenticate with. Changing one without the other breaks inter-container auth.

Confirmed the correct multi-step procedure from current docs (not yet executed — scoped out for next session): log out of the dashboard, `docker compose down`, generate a password hash via the indexer image's own tool (matched to the deployed `4.14.6` tag, not a doc example's different tag, to avoid a tool/runtime mismatch), edit `internal_users.yml`, edit the Compose file's `INDEXER_PASSWORD` in both the manager and dashboard service blocks, `docker compose up -d`, then run `securityadmin.sh` inside the indexer container to push the updated security config live, then log in with the new password to confirm.

Session ended here — logged out of the dashboard as step 1, remaining steps deferred to next session. Standard nightly shutdown followed.

### Outcome
- **Wazuh 4.14.6 deployed and confirmed healthy**: manager, indexer, and dashboard all running, verified via direct API/log checks rather than container status alone.
- **Dashboard reachable and logging in successfully** from MGMT10 — no new pfSense rule needed, MGMT10's default allow-all covered it.
- **Still on default credentials** — real security gap, flagged, not yet closed. Next session picks up exactly here.
- **Phase B Step 5 is functionally deployed but not complete** until the password change lands. Steps 6–9 (move the victim VM to Range VLAN, install Sysmon + Wazuh agent, re-test isolation, acceptance check) remain after that.

### Lessons
1. **Wazuh's `admin` indexer user can't have its password changed through the dashboard UI or API** — it's deliberately `reserved: true`. The only correct path is editing `internal_users.yml`'s hash directly and keeping the Compose file's plaintext copy in sync, then re-applying via `securityadmin.sh`. A UI "Forbidden" error here doesn't mean something's broken — it means the wrong tool is being used for this specific account.
2. **The certs-generator image throws a harmless error** during a permissions-cleanup step — cosmetic, doesn't stop cert generation, confirmed by checking the actual files on disk rather than trusting the log text alone.
3. **`docker compose ps` showing `Up` doesn't confirm a service is actually working** — worth a direct check before assuming a multi-container stack is genuinely healthy.
4. **Password/credential changes on multi-container stacks with shared secrets need every reference updated in lockstep** — or authentication between components breaks. Same "verify every field, not just that a command succeeded" discipline as the pfSense trunk-VLAN gotcha.
5. **Claude in Chrome's own tab group is separate from regular browsing tabs** — it can only see/control tabs it creates itself, not an existing tab already open elsewhere.

---

## 2026-08-04 — Wazuh admin password change completed; `securityadmin.sh` path/JAVA_HOME troubleshooting (Phase B Step 5 complete)

**Phase:** B (Step 5, closeout)
**Goal:** Finish the deferred Wazuh admin password change — apply `securityadmin.sh` to push the updated hash into the running indexer security config, confirm login with the new credentials.
**Rollback:** Not needed — continuation of the prior session's in-progress change. Docker Compose stack briefly recreated with updated environment variables, not stopped in a data-losing way.
**Transcript:** session log + SSH on `wazuh-host` + an interactive shell inside the indexer container, this date.

### What happened

Confirmed containers were stopped from the prior night's clean shutdown.

**Generated the bcrypt hash** for the new admin password via the indexer image's own tool.

**Edited `internal_users.yml`** — viewed the file first to confirm exact structure, then edited only the `admin` block's `hash` field, verified no other fields or users were disturbed.

**Edited the Compose file** — checked via `grep` first, found exactly 2 occurrences of the old password. Edited both, verified via `grep` afterward that both changed and no old value remained.

**Brought the stack back up** — fast since images were already cached, all three containers started clean.

**`securityadmin.sh` troubleshooting — several rounds, root-caused properly rather than guessed around:**
1. First attempt ran the tool's bare filename directly on `wazuh-host`'s own shell instead of inside the container — `command not found`, corrected to a full `docker exec` invocation.
2. Second attempt (correctly inside the container) printed only a `JAVA_HOME` warning and returned silently to prompt — no success or failure message. Opened an interactive shell into the indexer container to investigate directly instead of continuing to guess flags from outside.
3. The generic doc-example cert path didn't exist. `find / -iname "*.pem" 2>/dev/null` located the real path: `/usr/share/wazuh-indexer/config/certs/`.
4. The documented config directory didn't exist either. `find / -iname "internal_users.yml" 2>/dev/null` located the real path: `/usr/share/wazuh-indexer/config/opensearch-security/`. Confirmed via `grep` that this file correctly reflected the host-edited hash — the volume mount was working fine; only the assumed paths were wrong.
5. Re-ran with the corrected paths and the **admin** cert (not the indexer's own service cert, since this is an administrative action) — still returned silently after the Java warning, no other output.
6. Read the script itself and found the real cause: its final line pipes the actual Java invocation through a suppression, silently discarding any real error. The `JAVA_HOME` warning was the *only* thing this script was ever going to show, success or failure.
7. Manually reconstructed and ran the same Java invocation the script builds internally, without the suppression — got a clean "command not found" for `java`, revealing that Java genuinely wasn't on `$PATH` inside this container image.
8. `find / -iname "java" -type f 2>/dev/null` located the real binary: `/usr/share/wazuh-indexer/jdk/bin/java` — the image ships its own bundled JDK, expected for an OpenSearch-based container; the script's auto-detect just wasn't populated in this image.
9. Ran the full invocation directly against that Java binary, with the correct config/cert paths — completed successfully: connected to the cluster (GREEN, 1 node), all config types updated including internal users, ending in a success message.

**Logged into the dashboard with the new password** — confirmed working end to end.

### Outcome
- **Phase B Step 5 is now fully complete.** Wazuh manager, indexer, and dashboard all deployed, verified healthy, no longer on default credentials.
- **A real, version-specific procedure is now documented**, since the generic doc examples didn't match this container's actual layout:
  - Real cert path: `/usr/share/wazuh-indexer/config/certs/`
  - Real security config path: `/usr/share/wazuh-indexer/config/opensearch-security/`
  - Real Java binary: `/usr/share/wazuh-indexer/jdk/bin/java` (not on `$PATH` — call by full path, or set `JAVA_HOME` first)
  - `securityadmin.sh`'s own output is suppressed — a silent return to prompt is not evidence of success *or* failure; reconstruct and run the underlying Java command directly to see the real result.
- **Phase B Step 6 is next**: move the victim VM to Range VLAN, install Sysmon + Wazuh agent.

### Lessons
1. **Generic vendor documentation's example file paths may not match the actual layout inside a specific image/version.** Verify with `find` rather than trusting docs literally.
2. **A shell script silently returning to prompt with no success/failure message is not evidence of success.** Check the script's own source for output suppression before assuming a step completed — or failed.
3. **Container images that bundle their own JDK don't always populate `JAVA_HOME`/`$PATH` to point at it.** Call the bundled binary by its full, discovered path instead of assuming Java is missing entirely.
4. **The correct cert/key for an administrative security action is the dedicated admin cert**, not a service's own operational cert — the same least-privilege-of-purpose principle used elsewhere in this build, applied to certificate roles rather than user accounts.

---

## 2026-08-04 — Victim VM moved to Range VLAN, isolation proven live, Sysmon + Wazuh agent deployed (Phase B Step 6 complete — Phase B done)

**Phase:** B (Step 6, closeout)
**Goal:** Move the Windows victim VM onto RANGE30, prove the Phase A.5 isolation rule against a real VM for the first time, then instrument it with Sysmon and the Wazuh agent so it reports to the manager.
**Rollback:** No Proxmox snapshot for the VLAN move (VM was off; Hardware tab edit is trivially reversible). pfSense config exported before the new 1515 rule, per standing practice, though this was a purely additive change with no risk to MGMT10.
**Transcript:** Proxmox web UI via Claude in Chrome, PowerShell 7 and Windows PowerShell on the management PC, PowerShell on the victim VM's own console.

### What happened

**VLAN move.** Confirmed the victim VM was powered off. Via Proxmox's Hardware tab (Claude in Chrome, point-and-describe — I clicked, Claude confirmed each field via screenshot before commit), changed the network device from the flat bridge to the LAN-trunk bridge tagged VLAN 30. MAC address and NIC model left untouched — confirmed via the Hardware tab listing before and after.

**Isolation proven live — the real Phase A.5 acceptance check, finally run.** Booted the VM. Confirmed a clean DHCP lease on the Range subnet, gateway matching RANGE30's — confirms pfSense's DHCP scope working correctly. From inside the VM:
- Ping to a public IP → 100% loss (no internet) ✅ expected
- Ping to the home router → 100% loss (no home network) ✅ expected
- A TCP test to the Wazuh host on port 1514 → succeeded ✅ expected (the ingest port is specifically allowed; plain ICMP to the same host would have failed since the rule is TCP/1514-only, which is why a TCP-specific test was used instead of ping for this one)

This closes the caveat that had been sitting in the log since the isolation rule was written — the rule was correctly configured but never actually tested against a live VM until now.

**Staging tools onto an intentionally internet-less VM.** Since RANGE30 has zero outbound internet by design, Sysmon, the SwiftOnSecurity config, and the Wazuh agent MSI all had to be downloaded on the management PC (which has internet via MGMT10's default allow) and transferred in via virtual media rather than a direct download inside the VM — the general pattern for any airgapped/isolated system.

- Confirmed current versions live rather than from memory: Sysmon (current release from Microsoft's Sysinternals page), Wazuh agent pinned to **4.14.6** specifically — matching the manager's version exactly, since a search turned up an explicit compatibility note that agent version must be ≤ manager version.
- Downloaded all three to a local staging folder. First attempt was accidentally run in the wrong shell (a download command that doesn't exist there) — caught by checking the prompt string, corrected by opening real PowerShell 7.
- **Built an ISO from the staging folder to attach as virtual media** — took several failed attempts before landing on a working approach:
  1. First script used a direct `.Read()` call on the COM stream object — failed with "does not contain a method named 'Read'" in PowerShell 7.
  2. Suspected a PowerShell-version-specific COM interop difference; switched to Windows PowerShell 5.1 — **same error**, ruling that theory out.
  3. Root cause (confirmed via web search): this is a **long-standing, version-independent limitation** — the underlying COM interface is low-level with no dispatch layer, so PowerShell fundamentally cannot call `.Read()` on it directly, in any version. The standard, widely-documented fix is a small inline C# helper class, compiled at runtime, that does the raw marshaling correctly.
  4. Used that pattern — worked immediately, no further errors. ISO built and verified.
- Uploaded the ISO to Proxmox's `local` storage — this specific step had to be done directly, since selecting a local file for upload requires the OS's native file picker, which sits outside anything the browser extension can see or interact with even in principle.
- Attached the ISO to the victim VM's CD/DVD drive.
- Confirmed inside the VM: all three files visible on the new CD drive, landed at a drive letter checked live via a volume listing rather than guessed a second time.

**Sysmon installed.** Copied files to a local tools folder, extracted, installed with the SwiftOnSecurity config in an elevated PowerShell window (required for the driver install) — clean install, schema validated, driver and service both installed and started with no errors. Verified real events flowing — DNS queries, process creation with full hash/parent-chain detail, even a cross-process event from normal background activity — confirming the config was actually capturing the intended depth of telemetry, not just that the service reports "running."

**Wazuh agent installed, hit a real gap, fixed it.** Installed via a silent MSI install pointed at the manager's IP — the install returned cleanly but the service was stopped, not auto-started. Started it manually; the log showed a repeating cycle: requesting an enrollment key, unable to connect to the enrollment service on port 1515. `client.keys` was empty, confirming enrollment never completed.

**Root cause: the same "OPT/segment interfaces get zero rules by default" pattern that hit INFRA20 three times during Phase B — this time on RANGE30.** The existing isolation rule only covered port 1514 (ongoing agent-to-manager data traffic); a Windows agent's *first-time enrollment* handshake needs a separate one-time connection to port **1515** (the enrollment service), which RANGE30 had no rule for at all.

Wrote and applied a new pfSense rule on RANGE30 (I wrote the final rule per the two-tier standing rule, reviewed against a Claude-drafted reference first): Pass, RANGE30 subnets → INFRA20 subnets, TCP, port 1515, description `allow RANGE30 -> Wazuh enrollment (1515)` — placed above the existing rules, no reordering needed since order between two Pass rules above a single Block doesn't matter. Applied cleanly, confirmed via screenshot showing all three rules in the list.

**One process note:** several browser tabs controlled by Claude in Chrome closed unexpectedly mid-session, disrupting the point-and-describe workflow a few times. When it kept happening around this specific rule, switched to building the rule directly in pfSense's GUI myself and sending a screenshot for verification afterward — worked cleanly and is a reasonable fallback pattern if the extension misbehaves again.

**Retried enrollment** — restarted the service, waited, checked `client.keys`: now populated with a real agent ID, name, and registration key. Log confirmed a successful connection to the manager and "Agent is now online."

**Confirmed in the Wazuh dashboard directly** (checked myself, since this required my own dashboard credentials, which Claude does not have and should never be asked to enter): agent ID `001`, correct name, correct IP, OS correctly identified, status **active**, agent version matching the manager exactly, group `default`.

### Outcome
- **Phase B Step 6 is complete.** The victim VM is on RANGE30, isolation is proven live (not just configured), Sysmon is capturing detailed telemetry with the SwiftOnSecurity ruleset, and the Wazuh agent is enrolled, connected, and reporting.
- **Phase B is now fully complete** — Docker/Git substrate, Wazuh deployed and password-secured, first real endpoint instrumented and confirmed reporting end to end.
- **RANGE30 now has 3 firewall rules**: Pass (1515, Wazuh enrollment), Pass (1514, Wazuh ingest), Block (default deny, everything else) — same "explicit allow, then deny" shape as INFRA20's rule growth during Phase B.
- **Reusable technique captured**: the ISO-staging pattern (download on the management PC → build ISO via the COM/C# workaround → upload to Proxmox → attach as virtual CD) is now a proven path for getting tools onto any isolated Range VM.
- The attack VM (Kali) is still on the flat network, not yet moved to Range VLAN — that was never actually in Phase B's scope.
- **Phase C (Detection Engineering) is next.**

### Lessons
1. **A firewall rule can be correctly configured and still not be "proven"** — the isolation rule sat untested against a real VM for several days between being written and actually being validated live. Don't mark an acceptance check complete until it's actually been run against real traffic.
2. **Every new VLAN/segment's "zero rules by default" gap can bite more than once, in different directions.** INFRA20 needed five separate explicit rules discovered one at a time; RANGE30's single Wazuh-ingest rule wasn't enough either — first-time agent *enrollment* uses a completely different port than ongoing data traffic. Check both when standing up any new agent-based service on a segmented network.
3. **A low-level COM stream's `.Read()` method cannot be called directly from PowerShell, in any version** — a fundamental interop limitation, not a version issue. The fix is always a small compiled C# helper, not a different PowerShell version.
4. **Silent installer success doesn't mean the installed service actually started or connected** — check the service status and the application's own log, not just the installer's exit behavior.
5. **When picking a package version for a new tool, check compatibility constraints explicitly** (agent ≤ manager, in this case) rather than grabbing whatever a generic doc example shows.
6. **A browser-automation tool's controlled tabs closing unexpectedly is a real, if intermittent, failure mode** — when the automated workflow breaks down repeatedly, falling back to "build it directly, verify via screenshot afterward" is a reasonable and fast recovery pattern.
7. **A password typed during a session lives in that session's raw transcript regardless of care taken afterward** — the fix is discipline about never repeating it back in chat or writing it into any tracked documentation, not pretending it can be scrubbed retroactively. Rotate if full removal from history is ever actually required.

---

## Phase C.5, C.6, C.7, D — not started

No entries yet. Phase D (AI Triage Layer) has no work logged either — it comes after Phase C.6/C.7 in the current sequencing.

---

## 2026-08-09 — Phase C Steps 1-2: pre-atomic snapshot, Atomic Red Team staged and attached to victim VM (in progress)

**Phase:** C
**Goal:** Snapshot Win11-LTSC-Victim before any Atomic Red Team activity, then stage Atomic Red Team (engine + full atomics technique library) onto the victim VM via the proven ISO-staging pattern, since RANGE30 has no internet by design.
**Rollback:** Proxmox snapshot `pre-atomic-clean` taken on VM 101 before any changes — disk-only, no RAM state, confirmed via the Snapshots tab (`2026-08-08 20:12:31`).
**Transcript:** PowerShell 7 and Windows PowerShell 5.1 sessions on the management PC; Proxmox web UI via Claude in Chrome (point-and-confirm); pfSense web UI via Claude in Chrome (read-only diagnostics).

### What happened

**Step 1 — snapshot.** Took `pre-atomic-clean` on VM 101 via the Proxmox UI. Confirmed via the Snapshots tab: entry present, RAM: No, timestamped correctly, "NOW" marker showing current state above it.

**Step 2 — staging Atomic Red Team, several rounds of tool friction:**

1. **PowerShell 7 / PackageManagement broken.** `Install-AtomicRedTeam -getAtomics -Force` failed in PS7 with a `Get-InstalledModule` load error. Root cause found via direct diagnosis: PS7's Microsoft Store package ships a stub `PackageManagement` module under the sandboxed WindowsApps path — `ModuleType: Script`, zero exported commands (`Get-Command -Module PackageManagement` returned nothing), despite showing as "loaded." Confirmed via `Get-Command`/`.Path`/`.ModuleType` checks. Fix: switched to Windows PowerShell 5.1, which uses the real, non-sandboxed `PackageManagement 1.0.0.1` — no further issue at that layer.

2. **Execution policy blocked a dependency.** `powershell-yaml` module (a dependency of the Atomic installer) failed to load under `Restricted` policy (the effective default — all scopes showed `Undefined`, which resolves to `Restricted`). Fixed with `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser` — narrowest applicable scope, standard policy for legitimately-installed Gallery modules.

3. **Bitdefender (management PC) repeatedly interfered with hacktool-flagged atomic test payloads** — expected, since several atomics techniques (e.g., T1003.001 credential dumping) ship the same tools real attackers use. First hit: `Out-Minidump.ps1` deleted outright (not quarantined) mid-extraction. Learned the official `Install-AtomicRedTeam -getAtomics` installer's `Install-AtomicsFolder` function treats **any single extraction failure as fatal and rolls back the entire batch** — one blocked file cost the whole ~1000-folder atomics library, twice (confirmed via `atomics` directory showing 0 subfolders after each failed run, despite no top-level error). A hand-rolled `Expand-Archive`-based extraction hit the identical rollback behavior — same root cause, different code path. Ultimately resolved by getting Bitdefender's Exclusions UI working (it had been silently non-functional running non-elevated; re-opening Bitdefender **as Administrator** unlocked "Add an Exception") and excluding `C:\AtomicRedTeam` (the tool's actual default install path, not the `SOC-Lab\Staging` folder originally assumed). Bitdefender's behavioral engine separately flagged and blocked the `explorer.exe → powershell.exe` process chain once, during the earlier hand-rolled mass-extraction attempt — resolved by abandoning that approach in favor of the real installer once the exclusion was in place. One stray leftover file in an abandoned scratch folder triggered a "Trojan" detection requiring a restart to fully clean — not yet done, deferred as a non-blocking follow-up.
   - Clean re-run with the exclusion in place: `Install-AtomicRedTeam -getAtomics -Force` completed with no errors. Verified: 341 technique folders under `C:\AtomicRedTeam\atomics`, `T1059.001` present and populated (`bin`, `src`, `.md`, `.yaml`).

4. **ISO-build: three separate PowerShell/COM interop bugs, on top of the Phase B `IStream.Read()` issue already known.** Building the ISO via the `IMAPI2FS.MsftFileSystemImage` COM object at this larger scale (1351 files, 630 directories — vs. Phase B's handful of files) surfaced problems the earlier pattern never hit:
   - `AddDirectory()`'s return value doesn't marshal correctly through PowerShell COM interop — comes back `$null` even though the directory is actually created. Confirmed directly (`Is null: True` on a freshly created root's first call). Fix: call `.AddDirectory()` for its side effect only, then retrieve the real object via `.Item("name")` instead of trusting the return value.
   - `AddFile()` triggered "Exception **setting** AddFile" — PowerShell's dispatch layer was interpreting the method call as a property-set. Attempted fix via `[__ComObject].InvokeMember(..., BindingFlags.InvokeMethod, ...)` to force explicit method invocation — this compiled and ran but then threw `DISP_E_TYPEMISMATCH` on every file, a third distinct interop failure.
   - **Root cause assessment:** `IMAPI2FS` COM interop through PowerShell is fundamentally fragile at this file/folder count, not just for one specific call. Abandoned the scripted approach after three consecutive interop-layer failures rather than continuing to patch it.
   - **Fix: switched tools entirely.** Installed the Windows ADK **Deployment Tools** component only (not the full ADK) to get `oscdimg.exe` — a native command-line ISO-mastering tool with no PowerShell/COM interop involved. Single command: `oscdimg.exe -m -o -u2 -udfver102 "C:\AtomicRedTeam" "...\AtomicRedTeam.iso"`. Succeeded cleanly: 1351 files, 630 directories, 233,172,992 bytes — confirmed via `Get-Item`. (Note: ran inside PowerShell ISE, which displayed a `NativeCommandError`/`RemoteException` mid-run from oscdimg's normal stderr progress output — a known ISE display quirk, not a real failure; the tool's own "Done." / "100% complete" output and the verified file size confirm success.)

5. **Uploaded `AtomicRedTeam.iso` to Proxmox's `local` ISO storage** (manual step — native file picker, not reachable by browser automation, per established gotcha) and **attached it to Win11-LTSC-Victim's CD/DVD drive (ide0)**. Confirmed via the VM's Hardware tab: `CD/DVD Drive (ide0): local:iso/AtomicRedTeam.iso, media=cdrom, size=227708K` — matches the built file.

6. **Windows Security app on the victim VM (Win11 LTSC) opens and immediately closes** — confirmed `WinDefend` and `SecurityHealthService` are both running normally, so this isn't a backend problem; likely an LTSC-specific stripped UI issue. Worked around by adding the folder exclusion directly via `Add-MpPreference -ExclusionPath "C:\AtomicRedTeam"` instead of the GUI — **command was given but not yet confirmed successful; pick up here next session.**

7. **New blocker found at session end: management PC cannot reach `wazuh-host` (`10.10.20.100`) at all.** `Test-NetConnection` on port 443 and plain `ping` both failed (timeout, not refused). Diagnosed via Proxmox (confirmed `wazuh-host` VM is up, ~15 min uptime, matching a recent `pve01` power-cycle) and pfSense (confirmed reachable, logged in fine). pfSense firewall log showed **zero entries at all** — neither pass nor block — for traffic between `10.10.10.100` and `10.10.20.100`. Checked pfSense's live state table directly: confirmed a healthy, active `RANGE30 → wazuh-host:1514` connection (the victim VM's Wazuh agent, proving `wazuh-host` itself is genuinely up and listening), but **zero states of any kind** for MGMT10 → INFRA20. Working theory, not yet confirmed: the management PC has three simultaneously active network adapters/gateways (`10.10.10.1` MGMT10, `192.168.0.1` home network, plus a third unrouted `192.168.56.1` adapter) — traffic to `10.10.20.0/24` (a subnet with no direct route) likely resolves to whichever default gateway has the lower metric, and if that's the home-network adapter instead of MGMT10, the packets never reach pfSense at all, explaining the total absence of any log/state entry. `Get-NetRoute` failed with a broken CIM/WMI subsystem error on this machine; next step is `route print -4` (classic non-CIM method) to check the two default routes' metrics directly. **Not yet resolved — first item next session.**

### Outcome
- Snapshot `pre-atomic-clean` in place on VM 101.
- Atomic Red Team fully staged on the management PC: `invoke-atomicredteam` engine installed and functional, `atomics` library with 341 technique folders including `T1059.001` (needed for Step 3).
- `AtomicRedTeam.iso` (233MB) built, uploaded to Proxmox, and attached to Win11-LTSC-Victim as `ide0`.
- **Not yet done:** confirm the Defender exclusion on the victim VM took effect; copy the ISO's contents to a local folder on the victim VM; resolve the management-PC-to-`wazuh-host` routing problem (needed to view the Wazuh dashboard and confirm alerts in Step 3); run `T1059.001`; write the first custom detection rule.
- **Deferred, non-blocking cleanup:** restart the management PC to finish a Bitdefender disinfection that required it; remove the `C:\AtomicRedTeam` Bitdefender exclusion on the management PC once all staging there is complete.

### Lessons
1. **PowerShell 7's Microsoft Store package can ship a broken, sandboxed `PackageManagement` stub** that reports as loaded but exports zero commands — a silent, hard-to-diagnose failure mode distinct from a version mismatch. Windows PowerShell 5.1's non-Store `PackageManagement` doesn't have this problem. Worth defaulting to PS 5.1 for any one-off module-installer script rather than assuming PS7 is always the safer choice.
2. **The official Atomic Red Team installer's `Install-AtomicsFolder` function rolls back the entire extraction on a single file failure** — a real-world AV block (expected and common, given the library ships actual hacktool binaries) can silently cost the whole ~1000-folder library, not just the flagged item. A hand-rolled `Expand-Archive` extraction hits the same all-or-nothing behavior via a different code path. This is a structural fragility of both approaches, not a one-off bug — assume any AV interruption during Atomic Red Team staging requires either a pre-emptive AV exclusion or a fully custom per-file extraction loop.
3. **Bitdefender's exclusion-management UI can silently fail to respond (no error, just no effect) when the app isn't running elevated** — reopening as Administrator was the actual fix after several other theories (Central-managed policy, UI bug) were investigated and ruled out.
4. **`IMAPI2FS.MsftFileSystemImage`'s COM interface degrades badly through PowerShell interop at real-world file counts** (hundreds to low thousands of files) — three distinct, different interop bugs surfaced (`AddDirectory` null-return, `AddFile` property-vs-method ambiguity, `InvokeMember` type mismatch) that never appeared in Phase B's few-file test case. `oscdimg.exe` (Windows ADK Deployment Tools) is the more reliable tool for ISO creation at this scale — no COM interop involved at all. Worth defaulting to `oscdimg` for any future ISO-staging step rather than the scripted IMAPI2FS approach, even though the latter was already proven for small cases.
5. **PowerShell ISE can misreport a native command's normal stderr progress output as a `NativeCommandError`/`RemoteException`**, even when the command actually succeeded — check the tool's own final status line and independently verify the output (file existence/size) rather than trusting ISE's error framing alone.
6. **A machine with multiple simultaneously active network adapters/default gateways can silently blackhole traffic to a VLAN subnet it has no explicit route to**, if a different adapter's default gateway wins on metric — the failure looks identical to a firewall block (timeout) from the client side, but produces zero corresponding log or state entries on the actual firewall, which is the tell that the traffic never arrived at all.
7. **A Windows edition-specific problem (e.g., the Security app on Windows 11 LTSC opening and immediately closing) can be worked around via the equivalent PowerShell cmdlets** (`Add-MpPreference`) without needing to fix or diagnose the GUI issue itself, as long as the underlying services are confirmed healthy first.

---

## 2026-08-10 — Phase C Step 2 wrap-up + Step 3: routing fix, module staging complete, first atomic test run, real detection gap found and fixed

**Phase:** C
**Goal:** Close out the two open items from the prior session (mgmt-PC routing, Defender exclusion confirmation), finish staging Atomic Red Team on the victim VM, run the first atomic test (`T1059.001-17`), and confirm whether Wazuh actually detects it.
**Rollback:** No VM-level rollback point needed for this session — all changes were either read-only diagnostics, a routing table entry on the management PC (fully reversible via `route delete`), or Wazuh manager/agent config edits (each with its own stated revert command below).
**Transcript:** PowerShell (management PC and victim VM), SSH to `wazuh-host`, Wazuh dashboard via Claude in Chrome (with two mid-session disconnects requiring manual reconnection), Claude Code (delegated: `powershell-yaml` staging, Filebeat archives config).

### What happened

**1. Diagnosed and fixed the mgmt-PC → `wazuh-host` routing problem** (carried over from the prior session). Confirmed via `route print -4` that both `0.0.0.0` default routes (home network and MGMT10) were tied at identical metric 25 — not one being lower/preferred as originally guessed. On a tie, Windows was resolving unrouted destinations (like `10.10.20.0/24`, which has no dedicated route) to the home-network adapter, which forwarded the traffic to the internet, where it silently vanished (private RFC1918 destination, no route back). This fully explained the prior session's symptom: timeout with zero pfSense log/state entries, since the traffic never reached the lab network at all.
   - **Fix chosen: one supernet static route** rather than per-subnet routes or an interface-metric change — `route -p add 10.10.0.0 mask 255.255.0.0 10.10.10.1`. Covers all current lab VLANs (10.10.10.0/24, 10.10.20.0/24, 10.10.30.0/24) and any future one added within the `10.10.x.x` scheme, without touching general/home traffic routing at all. Confirmed via `route print -4` (shows under both Active Routes and persistent). Verified fixed: `Test-NetConnection -ComputerName 10.10.20.100 -Port 443` succeeded, Wazuh dashboard loaded cleanly in-browser.
   - **Rollback:** `route delete 10.10.0.0 mask 255.255.0.0 10.10.10.1`

**2. Confirmed the Defender exclusion on Win11-LTSC-Victim actually took** (unconfirmed at the end of the prior session): `Get-MpPreference | Select-Object -ExpandProperty ExclusionPath` returned `C:\AtomicRedTeam` — confirmed active.

**3. Discovered Atomic Red Team's PowerShell module has a second dependency (`powershell-yaml`) that isn't staged on an isolated VM by default.** `Import-Module` on `Invoke-AtomicRedTeam.psd1` failed with a missing-required-module error. Since the victim VM has no internet, delegated to Claude Code (per my explicit request, citing troubleshooting fatigue) to: check if `powershell-yaml` was already present on the management PC, `Save-Module` it if not, build a small ISO via `oscdimg.exe` (the tool already proven reliable from the prior session — explicitly instructed Claude Code not to use the IMAPI2FS COM approach), and report back the ISO location for manual Proxmox upload/attach. Completed without further troubleshooting needed. Module copied from the new virtual CD (`E:\POWERSHELL-YAML`) into `C:\Program Files\WindowsPowerShell\Modules\powershell-yaml` on the victim VM. Hit the same execution-policy block seen on the management PC in the prior session (fresh VM, still at default `Restricted`) — same fix, `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser`. `Import-Module` then succeeded cleanly; confirmed `Invoke-AtomicTest` (v2.1.0) available. **Step 2 fully complete.**

**4. Ran the first atomic test: `T1059.001-17` (PowerShell Command Execution).** Selected deliberately from the 20 available T1059.001 tests as the simplest, lowest-risk option — no download cradle, no credential-dumping tooling, no internet dependency. Test ran cleanly (`Exit code: 0`).

**5. Checked Wazuh for a corresponding alert — none fired.** Confirmed via the dashboard's Threat Hunting/Events view: most recent event for the agent was ~7 minutes before the test ran, nothing in the window matching the test's timestamp, across the full 24-hour range (which fully covered it). **This is the expected first outcome of detection engineering, not a failure** — it identifies a real gap to close with a custom rule.

**6. Root-caused the missing alert — found a bigger structural gap than "no rule exists yet."** Two layers, found in sequence:
   - **Layer 1 — Wazuh only indexes events that already match a rule ("alerts"), not the raw stream.** Enabled archive logging on the manager (`<logall_json>no→yes</logall_json>` in `ossec.conf` inside the manager container, followed by `wazuh-control restart` to apply). Confirmed via `ls -la /var/ossec/logs/archives/` that `archives.json` was actively growing post-restart.
   - **Layer 2 — even with archiving on, no Sysmon data showed up in the dashboard at all.** Discovered the dashboard has no index pattern for archived data by default (`Discover`'s index pattern dropdown only listed `wazuh-alerts-*`, `wazuh-monitoring-*`, `wazuh-statistics-*`). Delegated to Claude Code (with an explicit stop/redirect: it initially proposed a script hardcoding my real SSH password in plaintext — caught and corrected before execution; I don't have an SSH key set up for automated access yet, agreed as a follow-up rather than solved tonight, so Claude Code ran commands only when directly relayed) to check Filebeat's config inside the manager container. Found `archives.enabled: false` in Filebeat's Wazuh module config — flipped to `true`. Filebeat restart inadvertently took the whole manager container down briefly (agent reconnect blip, no data loss) — flagged by Claude Code as bigger than intended. Verified via direct indexer API query: 471 archived docs already present. Created the `wazuh-archives-*` index pattern manually in the dashboard (time field: `timestamp`).
   - **Layer 3 — even with the archive index working, searching for Sysmon-channel events (`data.win.system.channel:"Microsoft-Windows-Sysmon/Operational"`) returned zero results for the victim agent, despite Sysmon being installed and (per Phase B) confirmed running.** Checked the Wazuh agent's own `ossec.conf` on the victim VM directly: only three `<localfile>` blocks existed — `Application`, `Security`, `System` — **no entry for Sysmon's event channel at all.** This was the actual root cause behind every symptom in this entire multi-session troubleshooting arc: Sysmon has been logging locally the whole time, but the Wazuh agent was never told to read its channel, so none of that data ever left the victim VM.
   - **Fix:** added a fourth `<localfile>` block for `Microsoft-Windows-Sysmon/Operational` (`eventchannel` format) to the agent's `ossec.conf`, restarted the `WazuhSvc` service. Re-ran `T1059.001-17`; a follow-up search for the Sysmon channel returned 129 hits, and specifically Event ID 1 (process creation) returned 107 hits — confirmed real, flowing data.

**7. Confirmed a real, usable example event** for building the detection rule: `powershell.exe` (image) spawned by `cmd.exe` (parentImage), with `parentCommandLine` showing `"cmd.exe" /c powershell.exe -e <base64-encoded command>` — a textbook example of the exact pattern (parent-child + encoded command) the rule is meant to catch.

**8. Set up a reusable Discover view** for future technique testing: added `data.win.eventdata.image`, `data.win.eventdata.parentImage`, and `data.win.eventdata.parentCommandLine` as table columns (raw expanded-row view was very hard to read — long wrapped inline text), saved as **"PowerShell Process Creation - T1059.001"** for quick reuse on future MITRE technique tests.

**9. Security note:** Claude Code's first attempt at connecting to `wazuh-host` for the Filebeat diagnosis embedded my real SSH password in plaintext inside a PowerShell script. Caught before execution, redirected to plain interactive `ssh` (my normal connection method). **Password should be rotated** — it was typed in this form at least once and is sitting in that chat's history. A proper dedicated SSH key for Claude Code's use was discussed as the right long-term fix but not completed this session (no `~/.ssh` key or config exists yet for this purpose).

### Outcome
- Management PC can now reach all lab VLANs reliably (persistent supernet route in place).
- Atomic Red Team fully functional end-to-end on Win11-LTSC-Victim.
- Wazuh manager now archives all events (not just alert-matching ones); `wazuh-archives-*` index pattern exists in the dashboard.
- **The actual root cause of "no Sysmon detections" — a missing `<localfile>` entry in the agent config — is now fixed.** Sysmon events (including process creation) are confirmed flowing into Wazuh.
- A real, concrete example event (PowerShell via `cmd.exe`, Base64-encoded) is captured and ready to use as the basis for the first custom detection rule.
- **Not yet done:** the custom Wazuh rule itself (reference draft from Claude, final written by me — two-tier rule applies, this is core detection logic).
- **Not yet done:** rotate the `wazuh-host` SSH password; set up a dedicated SSH key for Claude Code's use on that host.

### Lessons
1. **A tied default-route metric is a distinct failure mode from "wrong metric ordering"** — Windows' tie-breaking behavior on identical metrics isn't something to rely on being consistent or intuitive. A single supernet static route (e.g., `10.10.0.0/16` covering an entire lab's VLAN scheme) is a more durable fix than per-subnet routes, since it automatically covers future subnets added within the same addressing convention.
2. **Atomic Red Team's PowerShell module has at least two staging dependencies beyond the engine itself** (`powershell-yaml`, confirmed; possibly others not yet hit) — worth checking `Import-Module`'s required-modules list proactively before assuming a fresh isolated VM is ready after just copying the main folders over.
3. **A missing detection is not necessarily "no rule exists" — it can be several layers deeper**, in order of how this session actually unfolded: (a) no rule matches the behavior, (b) even unmatched events aren't retained at all (archive logging off), (c) even with archiving on, the dashboard has no way to *view* archived data (no index pattern), (d) even with all of that working, the agent itself was never configured to read the relevant Windows Event Log channel in the first place. Assume the gap could be at any layer and check from the bottom up (agent config) rather than assuming the top layer (missing rule) is the whole story.
4. **A fresh Wazuh Windows agent's default `ossec.conf` only watches `Application`, `Security`, and `System` — not Sysmon's own channel** — even when Sysmon is installed and running correctly. This is a required manual addition (`<localfile><location>Microsoft-Windows-Sysmon/Operational</location><log_format>eventchannel</log_format></localfile>`) for any environment using Sysmon for enhanced telemetry, and is easy to miss since Sysmon appears to be working fine from the endpoint's own perspective.
5. **Restarting Filebeat inside a Wazuh manager container can restart the whole container**, not just the Filebeat process — worth expecting a brief agent-reconnect blip from what looks like a scoped service restart, and confirming no data loss afterward rather than assuming isolation.
6. **A Kibana/OpenSearch Dashboards search bar can silently fail to accept new typed input** while still displaying as focused/editable, even after standard clear-and-retype approaches — when this happens, using the field list's own "+" filter buttons or the Table/JSON expand view is a more reliable way to interact with the data than fighting the query bar.
7. **Never let an automation tool (including Claude Code) embed real credentials in a script, even for read-only diagnostic work** — plain interactive `ssh` (or equivalent) should be the default; a dedicated key should be generated and installed for any tool that needs non-interactive access, rather than a password ever being typed into a script or agent session.
8. **Browser automation tool connections can silently drop mid-session and not reconnect on simple retry** — when this happens repeatedly, switching to a manual "you click, I read the screenshot" workflow is more time-efficient than continuing to retry the same failing tool call.

---

## 2026-08-11 — Phase C Step 3: first custom detection rule debugged and confirmed working (three distinct root causes found and fixed)

**Phase:** C
**Goal:** Get the custom Wazuh detection rule (drafted 2026-08-10, targeting PowerShell spawned by a suspicious parent process, with an escalated variant for encoded commands) actually firing against real `T1059.001-17` atomic test traffic.
**Rollback:** Each rule file rewrite was preceded by a working prior version; the manager config itself was never touched, only `/var/ossec/etc/rules/local_rules.xml` inside the container and its ownership/permissions. Claude Code backed up the pre-fix version to `local_rules.xml.bak-20260811164630` before its final edit.
**Transcript:** SSH to `wazuh-host` (management PC, plain `ssh michael@10.10.20.100`), PowerShell on the victim VM, Wazuh dashboard via Claude in Chrome (repeated mid-session tool disconnects, worked around manually), Claude Code (delegated: SSH-based diagnosis and final fix application, after setting up a dedicated key for it — see below).

### What happened

**Initial rule (drafted prior session) failed silently — three consecutive, independent bugs, found one at a time:**

**Bug 1 — wrong `decoded_as` value.** The rule specified `<decoded_as>json</decoded_as>`, but real Sysmon-sourced events are actually tagged `windows_eventchannel` by Wazuh's decoder (confirmed directly from a real archived event's `"decoder":{"name":"windows_eventchannel"}` field). Fixed by correcting the value. This alone didn't resolve the issue — rule still didn't fire.

**Bug 2 — file ownership.** After confirming (via `grep` against `ossec.log`) that Wazuh never even attempted to load `local_rules.xml`, checked `ls -la` on the rules directory: the file was owned `1000:1000` (the host user's numeric UID/GID) instead of `wazuh:wazuh`, a direct consequence of using `docker cp` to copy the file from the host into the container — the copy preserves host ownership rather than adopting the container's user. Wazuh's own process (running as `wazuh`) couldn't read a file it didn't own with restrictive permissions, and failed completely silently — no error, no log entry, just skipped. Fixed with `chown wazuh:wazuh` inside the container, followed by a manager restart. Confirmed via `wazuh-logtest -v` that the rule was now genuinely being evaluated (`Trying rule: 100002 - ...` appeared in the trace) — but still never actually matched real production events (`grep` against `archives.json` for the last several qualifying events showed only the built-in rule `92032` firing, never `100002`).

**Bug 3 — wrong field-path namespace (the real blocker).** All four `<field name="data.win.eventdata...">`-style conditions in the rule used the `data.` prefix — which is how fields are named in the *exported/indexed* alert JSON (what's visible in `archives.json` and the dashboard), but is **not** part of the internal field namespace Wazuh's rule engine matches against during actual rule evaluation. Every `<field>` condition was silently evaluating to "field not found → false" on real events, even though an isolated field-by-field simulation (run independently, using the real event's stored values) showed all four conditions would evaluate `true` if the field paths had been correct — a discrepancy that turned out to be the tell. Confirmed by comparing against the built-in rule `92032`'s actual definition, which uses bare `win.eventdata.parentImage` (no `data.` prefix). Fixed by stripping `data.` from all four field paths in both rules. Also switched the rule's top-level dispatch from a standalone `<decoded_as>windows_eventchannel</decoded_as>` condition to chaining through `<if_group>sysmon_event1</if_group>` — matching the same dispatch pattern the working base-ruleset rule (`92032`) uses, off the real Sysmon dispatch root (rule `60000` → `60004` → `61600` → `61603`), since the standalone `decoded_as` condition — while visible in `wazuh-logtest`'s verbose trace — wasn't reliably triggering rule evaluation in the live production pipeline.

**Result: confirmed firing.** Re-ran `T1059.001-17` after all three fixes; the resulting event correctly triggered both `100002` (base condition: PowerShell spawned by a suspicious parent) and `100003` (child rule, `<if_sid>100002</if_sid>`, requiring an additionally-detected encoded command) — visible in `archives.json` as `rule_id: 100003, level 14`. Per Wazuh's default alerting behavior, only the more specific matched rule (`100003`) surfaces as the dashboard-visible alert when both a parent and child rule match the same event — `100002` matching is implied by `100003` firing at all, not something that needs to show up separately. (Learned this the hard way — searched only `rule.id:"100002"` in the dashboard for a long stretch and got no results, which looked like continued failure but was actually expected, correct behavior once the underlying bug was truly fixed.)

**Set up a dedicated SSH key for Claude Code's access to `wazuh-host`**, resolving the prior session's flagged gap (Claude Code previously had no non-interactive access and had briefly hardcoded a real password before being redirected). Found Claude Code had already generated a keypair (`~/.ssh/wazuh-host_ed25519`, comment `claude-mgmt-pc-to-wazuh-host`) during an earlier, incomplete attempt at this — reused it rather than generating a duplicate. Installed the existing public key into `wazuh-host`'s `~/.ssh/authorized_keys` manually (the credential-touching step stayed with me, per standing practice); Claude Code could then connect and run diagnostic commands directly for the remainder of the session, substantially speeding up the final debugging round.

### Outcome
- Custom rules `100002` (PowerShell spawned by a suspicious parent process) and `100003` (same, with an encoded command — child rule, higher severity) are confirmed working end-to-end against real atomic-test traffic.
- **Phase C Step 3 is complete.** Wazuh now has a working, deliberately-authored detection for `T1059.001`, distinct from (and complementary to) the built-in rule (`92032`) that was already catching a related but less specific pattern.
- Claude Code now has working, key-based, non-interactive SSH access to `wazuh-host` for future sessions — no more plaintext-password risk, no more relaying every command manually through me.
- **Search habit worth carrying forward:** when checking whether a *parent* rule in an `if_sid` chain fired, search for the parent OR any of its children (e.g. `rule.id:"100002" or rule.id:"100003"`), not the parent alone — Wazuh may only surface the more specific child alert even when the parent's condition is what's actually being validated.

### Lessons
1. **`docker cp` preserves the host's file ownership (UID/GID), not the container's** — any file copied into a container this way needs its ownership explicitly corrected (`chown`) to match whatever user the container's own process runs as, or it may be silently unreadable with zero error output. This is a distinct, non-obvious failure mode from a syntax error or a decoder mismatch, and produced literally no diagnostic signal anywhere (no log entry, no `wazuh-analysisd -t` complaint) until directly checked with `ls -la`.
2. **Wazuh's exported/indexed alert JSON (`data.win.*`) and its internal rule-match field namespace (`win.*`, no `data.` prefix) are genuinely different namespaces** — a field path that's completely correct for querying the dashboard or `archives.json` will silently fail to match anything inside a `<field name="...">` rule condition. This is easy to get wrong specifically *because* the dashboard/archive view (the natural place to go look at real data while drafting a rule) uses the wrong-for-rules prefix — cross-check field paths against a working built-in rule's actual definition (e.g. `grep -A 20 'id="<known-working-rule>"' ruleset/rules/*.xml`) rather than trusting field names copied from the alert/archive view.
3. **A standalone `<decoded_as>X</decoded_as>` condition can appear to work in `wazuh-logtest`'s verbose trace (showing the rule as "tried") while still not reliably firing in the live production pipeline** — chaining through the same `<if_group>`/dispatch-root pattern an existing, known-working built-in rule uses is a more reliable pattern to copy than authoring a rule's top-level trigger condition from scratch.
4. **When a parent rule (`<if_sid>`) and its child both match the same event, Wazuh surfaces only the more specific child rule as the visible alert** — searching for the parent rule ID alone can show zero results even when the underlying detection is working completely correctly. Search for the parent and all of its known children together when validating an `if_sid`-chained rule.
5. **`wazuh-logtest`'s field-matching trace and an independent field-by-field simulation can both report "this would match" while the rule still fails to fire in production**, if the simulation is checking against the wrong namespace (as in Bug 3) — a matching simulation is evidence the *regex logic* is sound, not evidence the *field paths* are correct against the live rule-engine namespace. These are separable failure modes and both need independent verification.
6. **A dedicated SSH key generated by an automation tool in a prior, incomplete session may already exist and just need to be installed** — worth checking (`ls ~/.ssh/`) before generating a fresh duplicate keypair.

---



## Phase E — Local AI Model Integration (`pve-ai` / `ai-vm` build)

**Note on dates:** the entries below are dated 2026-07-12/13 — before the SOC lab build itself started (2026-07-16, Phase A). They're placed here, not at the front of this file, because they document Phase E's infrastructure (the GPU inference node), and this log is organized by phase rather than strict calendar order. Originally tracked under its own separate plan on a spare PC (i9-10900KF, RTX 3070) with the goal of GPU-passthrough Ollama + Open WebUI. Merged into this single project and relocated to this position in the log on 2026-08-06. Execution model for this part of the build: Claude Code (PowerShell 7) drove all remote work over SSH, proposing each command with a plain-English explanation and waiting for my confirmation before running it. This was a deliberate choice, not a shortcut around learning — I'd already done a Proxmox setup and an Ubuntu Server install once before this build, and had done GPU passthrough at work previously as well, so I already had hands-on familiarity with the concepts involved and delegated the repetitive execution to Claude Code while retaining review/confirmation authority over every step.


## 2026-07-12 — Phase 3 start (Host Setup)

**Phase:** E (Phase 3 of the original ai-node plan)
**Goal:** Establish SSH access to the new Proxmox node (`pve-ai`, `192.168.0.202`), take a pre-change config snapshot, switch to the no-subscription repo, update, verify/enable IOMMU, map IOMMU groups for the RTX 3070.
**Rollback:** Rollback Point 0 — `/etc/default/grub`, `/etc/modules`, `/etc/network/interfaces` backed up before any change (see below).
**Transcript:** PowerShell session, this date.

### What happened

**Checked basic network reachability.** `Test-NetConnection 192.168.0.202 -Port 22` → `TcpTestSucceeded: True` — SSH port open, confirming the manual Proxmox install (BIOS virtualization settings + Proxmox VE install, done by hand at the console beforehand) was up.

**Added my existing SSH public key to `pve-ai`'s trusted keys**, pasted directly into the Proxmox web console (already logged in as root from the manual install): created `~/.ssh`, appended the public key to `authorized_keys`, set permissions (700 on the directory, 600 on the file). Reused the same keypair already trusted on `pve01` rather than generating a new one.

**Connected via key auth for the first time:** `ssh -o StrictHostKeyChecking=accept-new -o ConnectTimeout=10 root@192.168.0.202 hostname` → success, no password prompt, host key trusted, hostname confirmed as `pve-ai`. Note: unlike `pve01`, this host did **not** need the `KexAlgorithms curve25519-sha256` workaround — connected fine with SSH defaults, presumably a different/older OpenSSH build than `pve01`'s.

**Verified network config matches plan:** `ip a` over SSH showed `vmbr0` with `inet 192.168.0.202/24` as expected, physical NIC bridged and up, onboard wifi present but down/unused (expected, not needed).

**Added a `pve-ai` shortcut to `~/.ssh/config`** (same pattern as the existing `pve01` entry, minus the KEX workaround since it's not needed here).

**Hit the same Windows file-permission error as during the `pve01` setup** on first testing the new config entry — `Bad owner or permissions on .../.ssh/config`, inherited access from another Windows account. Fixed with the same known remedy: `icacls` reset to remove inheritance and grant only the current user read access. First retry hung indefinitely at 0% CPU with no output — killed the stuck process and retried with `-v -o BatchMode=yes`, which succeeded cleanly. Treated the hang as a one-off, since the verbose retry showed no errors or prompts anywhere in the handshake. `ssh pve-ai hostname` confirmed working afterward.

**Phase 3, Step 1 (Establish access) — done.** Key-based SSH to `pve-ai` working via both direct IP and the `pve-ai` config shortcut.

### Rollback Point 0 — pre-change config snapshot

Backed up `/etc/default/grub`, `/etc/modules`, `/etc/network/interfaces` (as they existed before any Phase 3 changes) to two locations: a local backups folder on the Windows management PC, and `/root/backups/` on the host itself. Verified both copies present with matching file sizes (grub 1534B, interfaces 247B, modules 212B).

**Rollback path from this point:** if a later change breaks boot or networking, restore any of these three files from either backup location and reboot.

**Phase 3, Step 2 (Rollback Point 0) — done.**

### Step 3 — Switch to no-subscription repo

**Checked current repo config (read-only) first.** This Proxmox install is on Debian 13 (trixie), which uses the newer deb822 `.sources` format, not the old `.list` files — the first check attempted the old format and correctly found nothing before checking the right place.

Found: `pve-enterprise.sources` (needs a paid subscription), `ceph.sources` (also paid, not relevant — single-node build has no Ceph use), and the standard `debian.sources` (left untouched).

**Changes made:**
1. Disabled both enterprise repo files by renaming (apt only reads `.list`/`.sources` extensions, so renaming is sufficient to disable without deleting).
2. Added the free no-subscription repo pointing at `download.proxmox.com/debian/pve`, suite `trixie`, component `pve-no-subscription`.
3. Verified both changes by listing the directory and reading the new file back — contents confirmed correct.

**Note on SSH sessions this session:** individual `ssh pve-ai "<cmd>"` calls reliably returned correct output, but the underlying `ssh.exe` process then hung at 0% CPU afterward instead of exiting cleanly, requiring a manual kill before the next command each time. Command results themselves proved trustworthy every time verified — treated as a client-side process-exit quirk (Windows OpenSSH client), not a sign of anything wrong on the `pve-ai` host.

**Ran the update:** `apt update && apt -y full-upgrade` (with `DEBIAN_FRONTEND=noninteractive`) → success, exit code 0. 94 packages upgraded, 1 new installed, 0 removed, 0 failed. ~466MB downloaded in ~13s.

**Kernel changed:** `proxmox-kernel-7.0.2-6-pve` → `proxmox-kernel-7.0.14-4-pve`. GRUB regenerated automatically as part of the kernel install, listing both versions as boot entries — old kernel remains available as fallback. Also notable: `pve-manager` 9.2.2→9.2.4, `qemu-server`, `pve-qemu-kvm` 11.0.0→11.0.2 (relevant later for VM/GPU work), ZFS stack, `chrony`, various security-patched libs. Nothing upgrade-related touched networking config or IOMMU/boot parameters — those were still stock from the manual install.

**Rebooted to load the new kernel.** Rollback path stated before rebooting: old kernel still present as a GRUB fallback if the new one failed; monitor still worked normally at this pre-passthrough stage, so console recovery was straightforward if needed. Waited ~45s post-reboot, confirmed port 22 open again, then confirmed via SSH: `uname -r` → `7.0.14-4-pve`, `hostname` → `pve-ai`. New kernel loaded, host fully responsive.

**Phase 3, Step 3 (repo switch + update + reboot) — done.**

### Step 4 — Verify IOMMU readiness

Read-only check: `dmesg | grep -e DMAR -e IOMMU` → `DMAR` entries present, including Intel VT-d confirmation — confirms the BIOS virtualization settings took effect at the hardware/ACPI level. No changes made, purely a check.

**Phase 3, Step 4 (IOMMU readiness) — done.**

### Step 5 — Enable IOMMU in GRUB

**Rollback path stated before editing:** original `/etc/default/grub` already backed up at Rollback Point 0 (both locally and on host). If the boot parameters caused a problem: restore the backed-up grub file and `update-grub`, then reboot, or use the GRUB boot menu directly (monitor still available pre-passthrough) to select the previous kernel or edit the boot line for a one-time boot without the new parameters.

**Change made:** `GRUB_CMDLINE_LINUX_DEFAULT="quiet"` → `GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"`.

First attempt via `sed` failed harmlessly (`unterminated 's' command`) — a PowerShell string-escaping issue mangled the sed script before it reached the remote host; no changes were made, a safe no-op. Fixed by building the remote command as a PowerShell single-quoted string (avoids PowerShell trying to interpret embedded double quotes) rather than double-quoted with backslash-escapes. Retry succeeded, verified via `grep` that the line read exactly as intended. Ran `update-grub` — output confirmed both kernels still found and listed as boot entries.

**Rebooted and verified:** `cat /proc/cmdline` confirmed `intel_iommu=on iommu=pt` present in the live boot line; `dmesg | grep -e DMAR -e IOMMU` now showed `DMAR: IOMMU enabled` explicitly, absent before this change. Kernel now actively using IOMMU.

**Phase 3, Step 5 (Enable IOMMU) — done.**

### Step 6 — Map IOMMU groups, identify GPU PCI IDs

Read-only listing of all PCI devices grouped by IOMMU group, via a loop over `/sys/kernel/iommu_groups/*/devices/*` and `lspci -nns`.

**Result: the RTX 3070 sits alone in IOMMU Group 1** — no unrelated devices share it:
- `01:00.0` VGA controller — GeForce RTX 3070 — PCI ID `10de:2488`
- `01:00.1` Audio device (GA104 HD Audio) — PCI ID `10de:228b`

This is the clean/easy case — no ACS override or physical PCIe slot change needed for the passthrough step. These PCI IDs and addresses are needed for the next phase (vfio-pci binding).

**Phase 3, Step 6 (Map IOMMU groups) — done.**

---

## Phase 3 — COMPLETE (2026-07-13)

All six steps done: SSH access established, Rollback Point 0 taken, repos switched + host updated + rebooted (new kernel `7.0.14-4-pve`), IOMMU verified enabled, GPU confirmed in a clean IOMMU group with PCI IDs logged above. Host `pve-ai` stable and ready for GPU passthrough.

---

## 2026-07-13 — Phase 4 (GPU Passthrough)

**Phase:** E (Phase 4 of the original ai-node plan)
**Goal:** Bind the RTX 3070 (PCI IDs `10de:2488` video, `10de:228b` audio; addresses `01:00.0`/`01:00.1`) to `vfio-pci` so it can be passed through to the future Ubuntu VM.
**Rollback:** Rollback Point 1 (stated below, before any change).
**Transcript:** PowerShell session, this date.

### Rollback Point 1

If anything in this phase caused a problem, the undo order was: remove `/etc/modprobe.d/vfio.conf` (the vfio-pci device ID binding file); remove any GPU-driver blacklist file created in this phase; restore the original `/etc/modules` from Rollback Point 0 (undoes the vfio module load-at-boot lines); run `update-initramfs -u` to rebuild without the vfio/blacklist changes; reboot.

**Important, stated before proceeding:** losing the physical monitor signal on this box during/after this phase's reboots is *expected* once `vfio-pci` claims the GPU — the driver working as intended, not a failure. It does not affect SSH/network access, since passthrough only affects which driver owns the GPU, not the host's NIC or networking stack. All rollback steps above can be done entirely over SSH; physical console access would only be needed if SSH itself became unreachable, which would indicate a boot failure, not a passthrough issue.

### What happened

**Step 1 — Load VFIO kernel modules.** Appended `vfio`, `vfio_iommu_type1`, `vfio_pci` to `/etc/modules` (previously empty/boilerplate comments only). Verified via `cat` — all three lines present. This alone doesn't bind the GPU yet, just ensures the VFIO framework loads at boot.

**Step 2 — Bind RTX 3070 to vfio-pci.** Created `/etc/modprobe.d/vfio.conf` with `options vfio-pci ids=10de:2488,10de:228b`, telling vfio-pci to claim exactly the RTX 3070's video and audio functions at boot, before the normal NVIDIA/audio drivers can grab them. Verified via `cat`.

**Note on SSH session hangs:** the same client-side hang-after-success pattern continued here, more persistently (needed 2–3 kill+retry cycles instead of 1). Every time content was eventually retrieved it was correct and matched what was written — the hang consistently came after the remote command completed successfully, never before or during. Host reachability on port 22 stayed solid throughout, confirming this as a Windows OpenSSH client-side quirk, not a host-side problem.

**Step 3 — Blacklist host GPU drivers.** Checked current driver bindings first (read-only): video function bound to `nouveau`, audio function bound to `snd_hda_intel` — confirmed the conflict the plan anticipated, both needed blacklisting for `vfio-pci` to win at boot.

Checked whether `snd_hda_intel` was shared with onboard motherboard audio before blacklisting it wholesale — it was (`00:1f.3`, "Onboard - Sound"). Flagged this before proceeding: blacklisting the module disables onboard host audio too, not just the GPU's HDMI audio. Confirmed to proceed anyway, since this is a headless Proxmox server and host audio output isn't needed.

Created `/etc/modprobe.d/blacklist-gpu.conf` blacklisting `nouveau`, `nvidia`, `nvidiafb`, `snd_hda_intel`. Verified via `cat`.

**Step 3 — done.** Side effect noted for future reference: onboard motherboard audio is now non-functional on the host (acceptable, headless server).

**Step 4 — Rebuild initramfs and reboot.** Warned before rebooting that the physical monitor would go completely dark once vfio-pci claimed the GPU during boot — expected, not a failure, doesn't affect SSH/network access. Confirmed to proceed. `update-initramfs -u` → success, regenerated cleanly, no errors. Rebooted, waited ~55s, host back up on port 22.

**Step 5 — Verify GPU bound to vfio-pci.** `lspci -nnk -s 01:00.0` and `-s 01:00.1` both showed `Kernel driver in use: vfio-pci`. Both GPU functions successfully claimed, binding confirmed stable across a reboot.

### Outcome
RTX 3070 (both video and audio functions) bound to `vfio-pci` and confirmed stable after reboot. Host `pve-ai` remained fully reachable over SSH throughout (physical monitor now dark, as expected/warned).

### Lessons
1. Once a GPU without host video fallback is bound to `vfio-pci`, expect the physical monitor to go dark permanently at boot — plan all future work on this host around SSH access only.
2. The Windows OpenSSH client's hang-after-success pattern got more persistent under repeated rapid commands in this session — worth treating as a known quirk to work around (kill + retry), not investigate further, since output has never once been wrong when eventually retrieved.

---

## 2026-07-13 — Phase 5 (Ubuntu VM + GPU)

**Phase:** E (Phase 5 of the original ai-node plan)
**Goal:** Download Ubuntu Server 24.04 LTS, create a VM with the RTX 3070 passed through, install the OS, install and verify the NVIDIA driver, snapshot.
**Rollback:** Rollback Point 2 — VM snapshot `clean-gpu-working`, taken at the end of this session.
**Transcript:** PowerShell session + Proxmox web console, this date.

### What happened

**Step 1 — Download Ubuntu Server 24.04 LTS ISO.** Checked storage first: `local` (dir, 87GB free) is the standard ISO storage path; `local-lvm` (1.7TB free) is for VM disks. Downloaded directly on the host rather than via the management PC (faster). Result: success, file size matched the expected release exactly.

**Step 2 — Create the VM.**

**Sizing discussion (deviated from the original plan's "32GB RAM, 8 cores"):** checked actual host resources first — `free -h`/`nproc` showed only ~31GB total RAM (not enough headroom for a 32GB VM allocation at all) and 20 threads (10c/20t, as expected). The original plan's hardware table had never actually listed total RAM as a confirmed spec — the 32GB figure was an unverified planning assumption, not a checked value.

Confirmed this would be the only VM on this host for now, and clustering with `pve01` was explicitly deferred (no timeline, needs quorum/qdevice consideration). Given that, went with a more generous allocation than a "shared host" scenario would warrant, while discussing the risk of going even higher: VM RAM isn't ballooned in this config, so Proxmox reserves the full amount at VM boot — cutting host headroom too thin risks OOM-killer intervention during host-side memory spikes (backups, updates), and since the physical monitor was now dark (GPU passthrough active), a host lockup would require a blind physical power-cycle to recover, not just a screen glance. Landed on **26GB RAM (26624 MiB)**, leaving ~5GB for the host, and **16 cores**, leaving 4 threads for the host. VMID 100 used (no existing VMs on this fresh host).

Commands run, each verified via output before the next: created the VM with `q35` machine type, OVMF BIOS, 16 cores, host CPU type, 26624MB memory, `virtio` network on `vmbr0`; added an EFI disk; set SCSI controller and a 200GB disk on `local-lvm`; attached the Ubuntu ISO as a CD-ROM; set boot order; added the RTX 3070 as a PCI device (`hostpci0`, both functions, PCI-Express checked).

One correction along the way: the boot-order command failed on the first attempt because an unquoted semicolon in the value was interpreted by the remote shell as a command separator instead of being passed literally — immediately corrected by quoting the value. Verified the final config via `qm config 100` — every setting confirmed correct (BIOS, machine type, cores, memory, PCI passthrough, disk, EFI disk, ISO attachment, boot order).

**Phase 5, Step 2 (Create VM) — done.**

**Step 3 — Install Ubuntu Server.** Started the VM; I installed Ubuntu Server 24.04 via the Proxmox web console following this guidance: Ubuntu Server (not minimized); network static `192.168.0.203/24`, gateway/DNS `192.168.0.1` (same static-IP pattern as the host, chosen over DHCP for a reliable address); storage on the entire 200GB disk with the guided/default LVM layout; OpenSSH server installed; all featured server snaps skipped (Docker/Ollama to be installed manually later). VM hostname set to `ai-vm`.

SSH access set up the same way as on `pve01`/`pve-ai` — the existing public key pasted into the VM's console as the primary user, permissions set. Verified: `ssh ai-vm "hostname && whoami"` (using the config shortcut set up in the next step) returned the expected hostname and username. Verified network config: the expected interface showed `192.168.0.203/24` as expected. Added an `ai-vm` shortcut to `~/.ssh/config`. Hit the same recurring Windows `.ssh/config` permission reset (happens each time the file is rewritten) — reapplied the known `icacls` fix, confirmed working afterward.

**Phase 5, Step 3 (Install Ubuntu Server + SSH access) — done.**

**Step 4 — Verify GPU visible in VM and install NVIDIA drivers.** Verified passthrough end-to-end: `lspci | grep -i nvidia` inside the VM showed both GPU functions visible.

Set up passwordless sudo for the primary user (needed for automated driver install over SSH) — run interactively at the console since it required a password Claude Code never had access to. Verified from the SSH side afterward.

Checked the recommended driver: `ubuntu-drivers devices` → `nvidia-driver-595-open` recommended (the newest, open-kernel-module variant). Installed it — success, 129 packages, ~405MB, GRUB/initramfs regenerated automatically as part of the install, no errors. Rebooted, waited ~40s, confirmed reachable, then `nvidia-smi` → **success**: NVIDIA GeForce RTX 3070, driver `595.71.05`, CUDA `13.2`, 8192MiB VRAM, idle at 34°C, no errors.

**Phase 5, Step 4 (GPU driver install + verify) — done.**

**Step 5 — Snapshot (Rollback Point 2).** `qm snapshot 100 clean-gpu-working` with a description noting Ubuntu 24.04 + NVIDIA driver 595-open installed and verified via `nvidia-smi`. Result: success, both the main disk and EFI disk snapshotted, verified via `qm listsnapshot 100`.

**Rollback Point 2:** if anything in later work (Ollama/Docker/Open WebUI setup) breaks the VM, restore with `qm rollback 100 clean-gpu-working` — returns to this exact state: fresh Ubuntu 24.04, NVIDIA driver working, nothing else installed yet.

### Outcome
Ubuntu Server 24.04 VM (`ai-vm`, VMID 100, `192.168.0.203`) created with GPU passthrough, NVIDIA driver installed and verified via `nvidia-smi`, snapshotted as `clean-gpu-working`.

### Lessons
1. Don't trust an unverified planning assumption (the original "32GB RAM" figure) against live host resources — checking `free -h`/`nproc` before committing to a VM size caught a real mismatch before it became a problem.
2. An unquoted semicolon in a value passed to a remote command can be interpreted by the shell as a command separator rather than part of the argument — quote the whole value.
3. VM RAM that isn't ballooned reserves its full allocation at boot; on a host with no physical display available (post-passthrough), leaving generous headroom matters more than on a host where a screen glance would catch an OOM situation early.

---

## Phase 5 — COMPLETE (2026-07-13)

Ubuntu Server 24.04 VM (`ai-vm`, VMID 100, `192.168.0.203`) built with GPU passthrough, NVIDIA driver installed and verified, snapshotted. Ready for the Ollama/Open WebUI stack whenever picked back up — see `LAB-BLUEPRINT.md` Phase E, not yet started as of this file's last update.

---


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

**VLAN move.** Confirmed Win11-LTSC-Victim (VM 101) was powered off. Via Proxmox's Hardware tab (Claude in Chrome, point-and-describe — I clicked, Claude confirmed each field via screenshot before commit), changed `net0` from `bridge=vmbr1` (flat, untagged) to `bridge=vmbr3,tag=30` (Range VLAN). MAC address (`BC:24:11:5B:13:31`) and NIC model left untouched — confirmed via the Hardware tab listing before and after.

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
- Uploaded the ISO to Proxmox's `local` storage (System > Node > local > ISO Images > Upload) — this specific step had to be done by me directly, since selecting a local file requires the OS's native file picker, which sits outside anything the browser extension can see or interact with even in principle.
- Attached the ISO to VM 101's `ide0` CD/DVD drive (Storage: `local`, ISO image: `soclab-tools.iso`) via the Hardware tab — confirmed via the resulting device string: `local:iso/soclab-tools.iso,media=cdrom,size=9920K`.
- Confirmed inside the VM: all three files visible on the new CD drive, landed at **`E:`** (not the assumed `D:` — checked live via `Get-Volume` rather than guessing a second time).

**Sysmon installed.** Copied files from `E:\` to `C:\SOC-Tools`, extracted `Sysmon.zip`. Installed with `Sysmon64.exe -accepteula -i C:\SOC-Tools\sysmonconfig-export.xml` in an **elevated** PowerShell window (required for the driver install) — clean install, schema validated (config schema 4.50 against Sysmon's own 4.91, handled correctly), driver and service both installed and started with no errors. Verified real events flowing via `Get-WinEvent` — DNS queries (ID 22), process creation with full hash/parent-chain detail (ID 1), even a `CreateRemoteThread` event (ID 8) from normal DWM background activity — confirming SwiftOnSecurity's config is actually capturing the intended depth of telemetry, not just that the service reports "running."

**Wazuh agent installed, hit a real gap, fixed it.** Installed via `msiexec.exe /i wazuh-agent-4.14.6-1.msi /q WAZUH_MANAGER="10.10.20.100" WAZUH_AGENT_NAME="win11-ltsc-victim"` — silent install returned cleanly but the `WazuhSvc` service was `Stopped`, not auto-started. Started it manually; log showed a repeating cycle: `Requesting a key from server` → `ERROR: Unable to connect to enrollment service at '[10.10.20.100]:1515'`. `client.keys` was empty, confirming enrollment never completed.

**Root cause: the same "OPT/segment interfaces get zero rules by default" pattern that hit INFRA20 three times during Phase B Steps 1 and 4 — this time on RANGE30.** The existing Phase A.5 rule only covers port 1514 (ongoing agent-to-manager data traffic); a Windows agent's *first-time enrollment* handshake needs a separate one-time connection to port **1515** (the enrollment service), which RANGE30 had no rule for at all.

Wrote and applied a new pfSense rule on RANGE30 (I wrote the final rule per the two-tier standing rule, reviewed against a Claude-drafted reference first): **Pass, RANGE30 subnets → INFRA20 subnets, TCP, port 1515, description `allow RANGE30 -> Wazuh enrollment (1515)`** — placed above the existing rules, no reordering needed since order between two Pass rules above a single Block doesn't matter. Applied cleanly (`The changes have been applied successfully`), confirmed via screenshot showing all three rules (1515 Pass, 1514 Pass, default Block) in the list.

**One process note:** several browser tabs controlled by Claude in Chrome closed unexpectedly mid-session tonight for unclear reasons, disrupting the point-and-describe workflow a few times. When it kept happening around this specific rule, switched to me building the rule directly in pfSense's GUI myself and sending a screenshot for Claude to verify field-by-field afterward — worked cleanly and is a reasonable fallback pattern if the extension misbehaves again.

**Retried enrollment** — `Restart-Service WazuhSvc`, waited 15 seconds, checked `client.keys`: now populated with **agent ID `001`, name `win11-ltsc-victim`, and a real registration key**. Log confirmed `Connected to the server ([10.10.20.100]:1514/tcp)` and `Agent is now online`.

**Confirmed in the Wazuh dashboard directly** (I checked myself, since this required my new password, which Claude does not have and should never be asked to enter): agent `001`, name `win11-ltsc-victim`, IP `10.10.30.100`, OS correctly identified as Windows 11 Enterprise LTSC 2024, status **active**, agent version `4.14.6` matching the manager exactly, group `default`.

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
## 2026-08-11 — SSH key-only hardening on wazuh-host

**Phase:**       C (Phase B follow-up item, closed out)
**Goal:**        Disable password authentication for interactive SSH login on `wazuh-host`, closing the open follow-up from Phase B.
**Rollback:**    Snapshot `pre-ssh-hardening-20260811` (VM 103, no RAM — crash-consistent), plus Proxmox console access as an out-of-band fallback that bypasses SSH entirely.
**Transcript:**  session-2026-08-11-1722.txt

### What happened
Generated a personal SSH key pair (`wazuh-host_personal_ed25519`), separate from Claude Code's existing dedicated key and the git deploy key. Installed the public key into `wazuh-host`'s `~/.ssh/authorized_keys` via password-authenticated `ssh` (last password-based login before hardening). Verified key-based login worked end-to-end (`whoami` confirmed) before making any config changes — this was the hard gate before touching `sshd_config`.

Edited `/etc/ssh/sshd_config` (`sudo sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/'`), validated syntax with `sudo sshd -t` before restarting anything. First restart attempt (`systemctl restart sshd`) failed — Ubuntu names the unit `ssh.service`, not `sshd.service` (only the binary is `sshd`). Corrected to `systemctl restart ssh`, succeeded cleanly. Kept the original session open throughout as a live fallback and tested from a second window only after the restart.

**Verification (two-sided):**
- Key auth: `ssh -i wazuh-host_personal_ed25519 michael@10.10.20.100 "whoami"` → succeeded, prompted for key passphrase only, returned `michael`.
- Password auth: `ssh -o PubkeyAuthentication=no michael@10.10.20.100` → rejected after repeated password prompts (`Permission denied`), confirming the server now refuses password auth outright.

**One correction mid-session:** initially set the new key's passphrase to match the `michael` account password by mistake. Re-keyed with `ssh-keygen -p -f wazuh-host_personal_ed25519` to set an independent passphrase (new value stored in Keeper), then re-ran both verification tests to confirm nothing broke.

Confirmed Claude Code's existing dedicated key (`wazuh-host_ed25519`, no passphrase, non-interactive use) still connects correctly post-hardening — unaffected by the change, as expected.

### Outcome
- `PasswordAuthentication no` is live and verified on `wazuh-host`. Interactive login now requires the personal key + passphrase; Claude Code's non-interactive key path is unaffected.
- **Phase B's open SSH hardening follow-up is now closed.**
- Snapshot `pre-ssh-hardening-20260811` retained as a rollback point; not needed, but kept per standing practice.
- **Still open:** rotating the `wazuh-host` account password itself (was briefly typed in plaintext by an automation tool earlier in the build, caught before use) — deprioritized for now, revisit later if desired.

### Lessons
1. **Ubuntu's systemd unit for the SSH daemon is `ssh.service`, not `sshd.service`** — only the binary itself is named `sshd`. `systemctl restart sshd` fails with "Unit not found" even though the daemon and config are both correctly named `sshd` everywhere else. Use `ssh.service` for any systemctl operation on Ubuntu.
2. **Always test a new key's passphrase independence, not just that a passphrase exists.** Reusing an existing account password as a key passphrase defeats the separation a key-based scheme is meant to provide — caught here only because it was flagged before the hardening step went further, not by any tooling.
3. **The two-sided verification pattern (prove the allowed path works AND the disallowed path fails) is worth reusing** for any future auth-hardening step (e.g. Wazuh dashboard auth changes, Entra Connect service account scoping) — confirming only the happy path leaves open the possibility that the restriction itself silently didn't take effect.

## 2026-08-13/14 — Custom detection rule for T1547.001 (Registry Run/RunOnce key persistence)

**Phase:**       C (Detection Engineering)
**Goal:**        Write a custom Wazuh rule detecting registry Run/RunOnce key persistence (T1547.001), building on the pattern established with `100002`/`100003`.
**Rollback:**    Snapshot `pre-atomic-persistence-20260811` (Win11-LTSC-Victim), no manager-side snapshot taken — config-only change, `local_rules.xml` copies retained throughout.
**Transcript:**  (session spanning 2026-08-13 evening into 2026-08-14)

### What happened
Staged Atomic Red Team test **T1547.001-1 (Reg Key Run)** — writes `HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run\Atomic Red Team` via `reg.exe`, with a clean single-command cleanup. Ran it, confirmed the registry value landed (`Get-ItemProperty`), then confirmed Sysmon captured it as **Event ID 13** with `RuleName: T1060,RunKey` — SwiftOnSecurity's own config already tags this pattern. Confirmed Wazuh's built-in rule **`92302`** (level 6, generic "Registry entry... modified using reg.exe") already catches this, but non-specifically — the actual gap being filled.

**Found the stale local file problem before writing anything new.** `~/local_rules.xml` on `wazuh-host` still had the **broken, pre-fix version** of `100002`/`100003` (old `data.win.*` namespace, bare `decoded_as`) — never updated after Claude Code fixed the live container copy directly in the prior session. Deleted the stale local file and re-pulled the real, current, working version straight from the container (`docker exec ... cat ... > ~/local_rules.xml`) before touching anything, avoiding a rule regression.

**Designed rule `100004`** with me writing the actual XML: `<if_group>sysmon_event_13</if_group>`, `<field name="win.eventdata.targetObject" type="pcre2">(?i)CURRENTVERSION\\\\(RUN|RUNONCE)\\\\</field>`, level 14, MITRE `T1547.001`. Field path (`win.eventdata.targetObject`) and the four-backslash escaping convention were both confirmed directly against Wazuh's own built-in rule `92300` (same file, `0860-sysmon_id_13.xml`) rather than assumed — `92300`'s working regex uses the identical `\\\\` pattern for a single literal backslash, confirming PCRE2's own doubling plus this ruleset's XML-level doubling stack to four.

Deployed via `docker exec -u wazuh ... sh -c 'cat > ...' < ~/local_rules.xml` (avoiding `docker cp`'s known ownership-reassignment bug from the prior session), verified ownership stayed `wazuh:wazuh`, validated with `wazuh-analysisd -t` (exit code 0), restarted only the manager container.

**Rule didn't fire.** Re-ran the atomic test, searched `rule.id:"100004"` in Discover — no results, while `rule.id:"92302"` on the same fresh event did return a hit. Confirmed via direct `archives.json` grep that `100004` had never matched a single event since going live, despite passing syntax validation. Field value (`targetObject`) confirmed correct via the raw archive JSON. Attempted `wazuh-logtest` for a definitive rule-trace, but it stalled at Phase 2 (decoding) for every test event, including a known-working control case — confirmed as a `wazuh-logtest`-specific artifact (raw JSON input uses the `json` decoder, not `windows_eventchannel`, so the `sysmon_event_13` group-assignment rule, which requires a text-format match, never applies) rather than evidence about `100004` itself.

**Root cause, found by Claude Code via live event capture (`tail -f archives.json` while re-running the atomic test):** Wazuh's rule engine evaluates root-level rules in ruleset file-load order and **stops at the first matching root rule for a given event** — it does not continue testing other independent root-level rule groups afterward. Built-in rule `92300` (root-level, `<if_group>sysmon_event_13</if_group>`, broad pattern covering essentially any `...CurrentVersion\Run...` write) loads before `local_rules.xml` (per `ossec.conf`'s `<rule_dir>` order) and claims the event first, since its pattern is broad enough to match the atomic test. `100004`, also root-level via `<if_group>`, never got evaluated — confirmed by `archives.json` showing only one `rule` object per event line (`92302`), never two. `100002`/`100003` avoided this same structural issue only by luck — no built-in `sysmon_event1`-group rule happens to match PowerShell-spawned-by-cmd/wscript with the specific field combination used in that atomic test.

**Fix:** changed `100004`'s dispatch from `<if_group>sysmon_event_13</if_group>` to **`<if_sid>92300</if_sid>`** — chaining as a child of the built-in rule that already claims these events (the same pattern `92301`/`92302`/`92303` themselves use), rather than competing as an independent root-level rule. Re-validated, restarted, re-tested: **`100004` fired correctly** — confirmed in `archives.json` (`"id":"100004"`, level 14, correct description and MITRE tag) and independently confirmed live in the Wazuh dashboard on a fresh atomic-test run.

### Outcome
- **Rule `100004` is live, verified, and correctly detecting T1547.001 Run/RunOnce key persistence** — chained under `<if_sid>92300</if_sid>`, firing alongside built-in rule `92302` as a sibling child of the same matched parent.
- Registry artifact from the final successful test left in place on Win11-LTSC-Victim (not cleaned up, no urgency since it's inert).
- Local `local_rules.xml` on `wazuh-host` is now confirmed in sync with the container's live copy — the earlier stale-file gap is closed.
- **This is a genuinely reusable structural lesson** for every future custom rule that chains via `<if_group>` rather than `<if_sid>`: any root-level group-gated rule can be silently short-circuited by an earlier-loading built-in rule with a broader pattern on the same event. `100002`/`100003` are not immune to this — they simply haven't hit it yet.

### Lessons
1. **Wazuh's rule engine stops at the first matching root-level rule per event — it does not evaluate all independent root rule groups against the same event.** A custom root-level `<if_group>` rule can be silently starved by an earlier-loading, broader built-in rule that claims the event first. This is invisible from XML syntax validation alone (`wazuh-analysisd -t` passes regardless) and only shows up as "the rule never fires, with no error anywhere."
2. **Chaining via `<if_sid>` onto a specific already-matched rule (rather than `<if_group>` onto a broad tag) guarantees independent, sibling evaluation** — this is the same mechanism `92301`/`92302`/`92303` use as children of `92300`, and the same mechanism `100003` already used as a child of `100002`. When a custom rule needs to coexist with built-in coverage on the same event type, prefer chaining onto the specific built-in rule that already claims those events over an independent `<if_group>` rule.
3. **`wazuh-logtest` fed raw JSON directly (bypassing the normal `windows_eventchannel` ingestion path) uses the generic `json` decoder, not the production decoder** — this can prevent event dispatch from ever reaching decoder-specific rule groups (like `sysmon_event_13`, which requires a text-format match only the `windows_eventchannel` decoder produces). A `logtest` stall or unexpected non-match doesn't necessarily indicate anything wrong with the rule itself — verify against real, live-captured production events (`tail -f archives.json` during a fresh test run) before concluding a rule is broken.
4. **Local working copies of rules files can silently drift from the deployed, fixed version inside the container** if a prior session's fix was applied directly via `docker exec`/`docker cp` without also updating the local file. Always re-pull the container's live copy as the source of truth before editing further, rather than trusting a local file's assumed currency.
5. **When confirming an escaping or field-path convention, check it against a real, currently-working built-in rule in the same rules file** rather than deriving it from first principles — `92300`'s existing, functioning `\\\\` pattern settled the four-backslash question definitively, faster and more reliably than reasoning through the escaping layers from scratch.
## 2026-08-17 — Custom detection rule for T1003.001 (Credential Dumping — LSASS Memory / Silent Process Exit)

**Phase:**       C (Detection Engineering)
**Goal:**        Write a custom Wazuh rule detecting T1003.001 credential dumping, building on the pattern established with `100002`/`100003`/`100004`.
**Rollback:**    Snapshot `pre-atomic-credaccess-20260817` (Win11-LTSC-Victim), taken before any T1003 activity.
**Transcript:**  (session spanning evening of 2026-08-17)

### What happened

**Sub-technique selection.** Chose T1003.001 (LSASS Memory) over T1003.002 (SAM), .003 (NTDS, no DC yet), and .006 (DCSync, planned for Phase C.6) — LSASS is the most common real-world credential dumping technique, works standalone without a domain, and lines up with the new telemetry (Sysmon Event ID 10, ProcessAccess) flagged at session start.

**First attempt — `T1003.001-2` (comsvcs.dll MiniDump) blocked twice, at two different layers.** AMSI blocked the script outright (`ScriptContainedMaliciousContent`) until a Defender exclusion was added for `C:\AtomicRedTeam`. After that, the test failed differently — `Access is denied` starting the process. Root-caused via direct registry check: `HKLM\SYSTEM\CurrentControlSet\Control\Lsa\RunAsPPL = 2` — LSASS is running as a **UEFI-locked Protected Process Light**, a real Windows 11 hardening feature blocking non-protected processes from opening handles to LSASS at all. Confirmed via Sysmon: zero Event ID 10 entries exist anywhere in the log for this attempt — PPL blocks the access before Sysmon's kernel callback ever sees it. This is a legitimate defensive finding, not a lab misconfiguration.

**Pivoted to `T1003.001-14` (Silent Process Exit via nanodump)** — a technique that abuses WerFault.exe (which runs with its own elevated access) to dump LSASS on process exit, rather than the attacker's own process opening a handle directly. This test requires the third-party `nanodump.x64.exe` binary, which the atomic test's own prereq step downloads from GitHub.

**Staging friction, several rounds:**
1. RANGE30 has no internet by design — the download had to happen on the management PC and be staged onto the victim VM via ISO, same pattern as prior sessions (`powershell-yaml`, original Atomic Red Team library).
2. Bitdefender on the management PC blocked the download and, separately, blocked the file mid-copy into the ISO staging folder — required two rounds of exclusion (first a narrow file-level exclusion, which didn't fully take due to the elevation gotcha already documented in this build; then a broader folder-level exclusion with Bitdefender confirmed running as Administrator).
3. First ISO build silently produced a corrupted filesystem — 45KB output (smaller than the 108KB source binary), and mounting it showed `EXTERNALPAYLOADS` as unreadable/corrupted. Root cause: Bitdefender likely touched the file mid-`oscdimg` write, before the folder-level exclusion was confirmed in force. Cleaned up and rebuilt from scratch under the confirmed exclusion — second build succeeded (153,600 bytes, contents verified readable via mount before uploading).
4. Uploaded to Proxmox `local` storage, attached to VM 101, copied `nanodump.x64.exe` into `C:\AtomicRedTeam\ExternalPayloads\` on the victim (the exact path the atomic test's prereq check expects) — confirmed via `-CheckPrereqs`.

**Ran the test — Defender blocked it again, via a different mechanism than PPL.** Exit code 0, but nanodump self-reported "WerFault did not create the dump." Checked `Get-MpThreatDetection` (initially unhelpful — several days stale) and the real-time Defender Operational event log directly, which showed the actual cause: two **named cloud/behavioral signatures**, `Behavior:Win32/LsassDump.G` and `Behavior:Win32/LsassDump.H`, fired on the exact IFEO registry writes the technique makes — not a folder-path scan, so the existing exclusion didn't help. Confirmed via a popup notification and the event log detail.

**This turned out to be the actual detection opportunity, not just a blocker.** The named signatures fired on:
- `HKLM\...\Image File Execution Options\lsass.exe\GlobalFlag` (DWORD `0x00000200` — `FLG_MONITOR_SILENT_PROCESS_EXIT`)
- `HKLM\...\SilentProcessExit\lsass.exe\...` (DumpType, LocalDumpFolder)

Checked Sysmon directly for the underlying registry events (independent of whether Defender's own remediation succeeded) — confirmed Event ID 12 (DeleteKey) and Event ID 13 (SetValue) both captured, tagged `RuleName: T1183,IFEO` by SwiftOnSecurity's config, with the real `GlobalFlag = 0x00000200` value visible in the raw event. This is a legitimate, narrow, high-fidelity artifact: almost nothing legitimate sets this specific value on `lsass.exe`'s IFEO key.

**Clock skew discovered and fixed, mid-session, as a real environment-correctness issue.** While checking whether the archive pipeline had actually captured the test events, found the manager's archive file hadn't been updated in 20+ minutes despite ongoing agent activity — turned out the victim VM's clock was running roughly 2 hours ahead of real time, from two independent causes: (1) timezone was set to Pacific instead of Central (a leftover default from initial VM setup, never corrected), and (2) even after fixing the timezone, the underlying system clock itself was still off by a separate ~2-hour drift, unrelated to timezone. Fixed both — `Set-TimeZone -Id "Central Standard Time"`, then `Set-Date` to match the management PC's real time. Verified via `(Get-Date).ToUniversalTime()` matching `wazuh-host`'s own `date -u` output afterward. Re-ran the atomic test cleanly under the corrected clock for the final verification below.

**Designed and deployed rule `100005`**, I wrote the actual XML: chained via `<if_sid>61615</if_sid>` (Event 13's real dispatch rule, confirmed live via `grep` against `0595-win-sysmon_rules.xml` — no built-in rule already covers IFEO/lsass, confirmed via a targeted grep returning nothing), single field condition on `win.eventdata.targetObject` matching `Image File Execution Options\\lsass\.exe\\GlobalFlag` (case-insensitive, correct four-backslash doubling convention), level 14, dual MITRE tags `T1003.001` (credential access objective) and `T1546.012` (the actual IFEO-injection mechanism — added after discussing which tag most accurately describes the artifact vs. the attacker's goal).

**Verification took several wrong turns before landing on the right method.** `wazuh-logtest` fed raw JSON via `stdin` matched only the generic syslog catch-all (`1003`, "message too large") — confirmed this is the same `json`-vs-`windows_eventchannel` decoder mismatch already documented for `100004`'s troubleshooting; `-l EventChannel` didn't resolve it either. Delegated a diagnostic pass to Claude Code, which (after an SSH username-guessing incident — see Lessons) confirmed via `ossec.log` that the manager restart was clean, `local_rules.xml` ownership was correct, and rule 100005 was syntactically well-formed and loaded — but could not get a conclusive live-fire confirmation via `wazuh-logtest` either. **Realized the actual blocker: the archived test event being checked (01:52:54 UTC) predated the manager's restart/rule-load time (02:10:42 UTC)** — a pure sequencing issue, not a rule problem. Ran one more fresh atomic test after the rule was confirmed loaded, then checked `archives.json` directly (the same authoritative method already trusted for `100002`–`100004`) — **rule `100005` fired correctly**, confirmed via the real alert JSON: `"id":"100005"`, level 14, both MITRE tags present, correct auto-derived tactics/techniques (Credential Access, Privilege Escalation, Persistence / LSASS Memory, Image File Execution Options Injection), `firedtimes: 1`, timestamped 02:32:31 UTC — well after the rule went live.

**Claude Code SSH incident.** When delegated the diagnostic pass, Claude Code could not resolve the host alias `wazuh-host` (no `.ssh/config` entry exists for it) and, rather than stopping to ask, looped through eighteen different guessed usernames (`wazuh`, `ubuntu`, `admin`, `root`, `ec2-user`, etc.) against the live host before finally stopping to ask for the correct one. The correct username was already documented in `PROJECT-INSTRUCTIONS.md`'s Claude Code allow-rule the whole time. No lockout or security impact, but the behavior itself — repeated credential guessing against a live host — is a real problem worth preventing structurally, not just correcting in the moment. Added a new standing rule to `PROJECT-INSTRUCTIONS.md` prohibiting username/credential guessing loops going forward.

### Outcome
- **Rule `100005` is live, verified, and correctly detecting T1003.001/T1546.012 (Silent Process Exit LSASS credential dumping)** — chained under `<if_sid>61615</if_sid>` (Sysmon Event 13's real dispatch rule), confirmed firing end-to-end against real atomic-test traffic via direct `archives.json` inspection.
- **Phase C now has three working custom detections spanning three tactics:** `100002`/`100003` (Execution, T1059.001), `100004` (Persistence, T1547.001), `100005` (Credential Access, T1003.001/T1546.012).
- **Two real defensive layers documented as legitimate findings, not obstacles:** LSASS PPL (UEFI-locked) blocking direct memory access outright, and Defender's named behavioral signatures (`LsassDump.G`/`H`) catching the Silent Process Exit technique via its registry footprint even when the direct-access path is already closed.
- **Win11-LTSC-Victim's clock is now correctly set** (Central timezone, drift corrected) — a real environment-correctness fix with value beyond this one session, since accurate timestamps matter for all future timeline/correlation work.
- **New standing rule added to `PROJECT-INSTRUCTIONS.md`:** Claude Code must never guess at credentials/usernames against any host — stop and report back on the first failure with the documented credential instead.
- `nanodump.x64.exe` and its ISO remain staged on the management PC and victim VM; Bitdefender exclusions on the management PC (`C:\AtomicRedTeam\Staging`) are still in place and not yet cleaned up — flagged as non-blocking deferred cleanup, consistent with how the original AtomicRedTeam-related exclusions were handled.

### Lessons
1. **LSASS PPL (`RunAsPPL=2`, UEFI-locked) blocks direct-handle credential dumping techniques (`comsvcs.dll`, ProcDump, direct Mimikatz) before Sysmon's ProcessAccess callback even fires** — zero Event ID 10 telemetry is itself the signal that PPL intervened, not evidence the test failed to run. Techniques that launder the access through a trusted intermediary process (like WerFault via Silent Process Exit) are a documented, real-world way around this specific control.
2. **A technique blocked by AV/EDR is not a dead end for detection engineering — the artifact left behind by the blocked attempt can be the detection itself.** Defender's named signature catch on the IFEO/`GlobalFlag` registry writes (not the dump file, which never completed) provided the exact field values the custom rule needed.
3. **Clock skew can masquerade as a broken detection pipeline.** A "stale-looking" archive file with no recent entries was actually correct — the manager's clock was right, the victim VM's was ~2 hours ahead, so events genuinely hadn't happened yet from the manager's point of view. Two independent causes existed simultaneously (wrong timezone AND separate underlying clock drift) — fixing one didn't fix the other; both needed checking.
4. **`wazuh-logtest` fed raw JSON via `stdin` (with or without `-l EventChannel`) does not reliably reproduce the real Sysmon rule-matching path in this Wazuh version** — same underlying issue as documented for `100004`'s troubleshooting (`json` decoder vs. `windows_eventchannel`), confirmed again independently. Direct `archives.json`/alert-log inspection against a real, freshly-generated, correctly-timed test event remains the more reliable verification method for this environment.
5. **When delegating diagnostic work to Claude Code, always confirm it has (or is pointed at) the correct connection details before it starts** — an unresolvable host alias or missing credential should trigger one clarifying question, not a guessing loop. This is now a standing rule, not just a one-off correction.
6. **A test event's timestamp must be checked against the rule-load timestamp before concluding a rule "isn't firing."** An event that genuinely predates the rule's existence in the running manager will never match it, regardless of how correct the rule is — this cost significant troubleshooting time before the sequencing was noticed directly in Claude Code's own timestamp report.

## 2026-08-18 — Fixed `100002` short-circuit risk (if_group → if_sid)

**Phase:**       C (Detection Engineering)
**Goal:**        Close the flagged-but-not-yet-fixed vulnerability in `100002` — the same silent short-circuit risk originally found and fixed in `100004`.
**Transcript:**  Short session, SSH to `wazuh-host`.

### What happened
`100002` was still chaining onto `<if_group>sysmon_event1</if_group>` instead of a specific rule ID — the same pattern that silently broke `100004` before its fix. Confirmed the correct anchor via `grep` against `0595-win-sysmon_rules.xml`: rule `61603` matches Sysmon Event ID 1 specifically, chains onto `61600`, and its own `<group>` tag is literally `sysmon_event1` — i.e. `61603` is the actual rule generating the group tag `100002` was riding on. Re-chained `100002` onto `<if_sid>61603</if_sid>`. `100003` needed no change — it already chains onto `100002` via `if_sid`, correct from the start.

Deployed via the standard `docker cp` → `chown wazuh:wazuh` → verify ownership → `wazuh-control restart` sequence. Restart clean, no parse errors. Re-ran `T1059.001-17` on Win11-LTSC-Victim; confirmed via direct `archives.json` inspection that `100003` fired correctly (level 14, both MITRE tags `T1059.001`/`T1027` intact) at 20:46:28 UTC, well after the 20:39:59 restart — since `100003` only fires when `100002` fires first, this confirms both rules are working under the new anchor.

### Outcome
- **`100002`/`100003` no longer at risk of the silent `if_group` short-circuit.** All four custom rules (`100002`–`100005`) now chain via `if_sid` onto specific rule IDs rather than group tags.
- Closes the last flagged open item from the T1003.001 session (2026-08-17).

### Lessons
Reinforces the `100004` lesson: `if_group` chains are a known, silent failure risk regardless of how long a rule has been running without issue — "hasn't broken yet" is not the same as "safe." Worth a final pass checking any other custom rules for the same pattern before considering Phase C's rule set closed out.

## 2026-08-18 — Rules `100006` (T1087.001) and `100007` (T1070.005): Discovery + Defense Evasion

**Phase:**       C (Detection Engineering)
**Goal:**        Add two more custom rules covering new tactics (Discovery, Defense Evasion), applying the new pacing rules (low-friction-first, batched read-only checks) from the prior session.
**Transcript:**  Short, fast session — pacing changes worked as intended.

### What happened
Selected **T1087.001 (Local Account Discovery)** and **T1070.005 (Network Share Connection Removal)** specifically for low friction — both use only built-in `net.exe`, no staging, no elevation required. Checked atomic test lists for both in one batch before picking specific tests (`T1087.001-8`, `T1070.005-2`).

**Hit one real, unplanned blocker:** the originally-planned Defense Evasion pick (T1070.004, Delete Prefetch File) generated no Sysmon telemetry at all. Investigated directly rather than assuming a rule problem — confirmed via `Get-MpThreatDetection` that Defender had not interfered (no detections since 8/17), then found the actual cause in `C:\SOC-Tools\sysmonconfig-export.xml`: the FileDelete (Event ID 23) `RuleGroup` is present only as a documentation comment header with no active rule block underneath — SwiftOnSecurity's config doesn't monitor file deletion at all in its current form on this VM. Rather than reconfigure Sysmon mid-session (a real environment change with broader effects), pivoted to T1070.005 instead, which relies on already-proven Event ID 1 telemetry.

**Designed and deployed rules `100006` and `100007`**, both narrow-scope per explicit decision (matching exactly what was tested, not broader `net` command patterns), both chained via `<if_sid>61603</if_sid>` (the same Event 1 dispatch rule `100002` now uses), both set to **level 8** rather than 14 — a deliberate departure from prior rules, since `net user`/`net share /delete` are common legitimate admin activity and tagging them at credential-dumping severity would create alert fatigue.

**Caught and fixed one drafting error before deployment:** the first draft of both rules used four backslashes (`\\\\net\.exe$`) for the `win.eventdata.image` field, incorrectly copying the escaping convention from `targetObject`-based rules (100004/100005). Cross-checked against `100002`'s own proven-working `image` field pattern (`\\powershell\.exe$`, two backslashes) before deploying — corrected both rules to match.

Deployed via the standard sequence, restart confirmed clean (no parse errors), re-ran both atomic tests, verified via direct `archives.json` inspection: `100006` fired three times (once per enumeration command — `net user`, `net localgroup "Users"`, `net localgroup`), `100007` fired once (`net share \\test\share /delete`). Both timestamped after the restart, both with correct MITRE tags and Wazuh's auto-derived tactic/technique labels.

### Outcome
- **Six verified custom rules now live, spanning five tactics:** Execution (`100002`/`100003`, T1059.001/T1027), Persistence (`100004`, T1547.001), Credential Access (`100005`, T1003.001/T1546.012), Discovery (`100006`, T1087.001), Defense Evasion (`100007`, T1070.005).
- **New pacing rules worked as intended** — this whole session (technique selection, two atomic tests, one real blocker diagnosed and pivoted around, two rules designed/deployed/verified) ran significantly faster than the T1003.001 session, despite hitting a genuine unplanned issue (missing FileDelete telemetry) along the way.
- **Known gap documented, not silently worked around:** Sysmon's current config on Win11-LTSC-Victim does not capture FileDelete (Event ID 23) events. File-deletion-based detections (T1070.004 and similar) are not currently possible without a deliberate Sysmon config change — logged as a next-step, not attempted as a quick fix mid-session.

### Lessons
1. **A Sysmon config can have a named section for an event type without actually enabling it** — the presence of a "SYSMON EVENT ID 23 : FILE DELETE" header in the config is not evidence the event type is monitored; only an active (non-commented) `<RuleGroup>` block confirms that. Worth checking this directly for any future technique whose detection depends on a specific Sysmon event ID, rather than assuming standard SwiftOnSecurity coverage.
2. **Field escaping conventions are not uniform across all Wazuh fields** — `win.eventdata.image` uses two backslashes per literal backslash, while `win.eventdata.targetObject` uses four, in this ruleset. Always cross-check a new rule's escaping against an existing *working* rule using the *same field*, not just any working rule.
3. **Deliberately varying alert severity (level) by real-world base rate matters.** Rules for common legitimate admin actions (`net user`, `net share`) were set to level 8, not the level 14 used for genuinely rare/malicious-only artifacts (`100004`, `100005`) — same detection engineering principle as tuning for signal-to-noise, worth being able to articulate in an interview.

---

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
