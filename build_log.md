# build_log.md
### Home SOC Lab — running record

Every change, its **actual** output, the decision behind it, and every rollback point. Appended as it happens — never reconstructed afterward.

**Why this file exists:** a previous attempt at Cisco + pfSense configuration (via a different AI tool) caused a lockout. There was no record of what had been changed, so there was no way to reverse it — which is what forced a full Proxmox reinstall. This file is the answer to "what exactly did I change?" before I need to ask it.

**Note on scope (2026-08-06):** this file now also incorporates the build history of the `pve-ai`/`ai-vm` AI inference node, formerly tracked as a separate project (`hybrid_ai_node_build_plan.md` and its own `build_log.md`). That work chronologically predates the SOC lab build itself (2026-07-12/13, vs. the lab's own start on 2026-07-16), but this file is organized by **phase**, not strictly by date — so those entries are grouped under Phase E below, where that work actually belongs in `LAB-BLUEPRINT.md`, with their original dates retained. The original standalone files remain as historical source material but are no longer separately maintained.

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

## Phase C, C.5, C.6, C.7, D — not started

No entries yet. Per `LAB-BLUEPRINT.md`, Phase C (Detection Engineering) is next up. Phase D (AI Triage Layer) has no work logged yet either — it comes after Phase C.6/C.7 in the current sequencing.

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

