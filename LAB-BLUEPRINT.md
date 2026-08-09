# LAB-BLUEPRINT.md
### Home SOC Lab — what I'm building, and in what order

**Purpose:** My shared map with Claude. What gets built, in what sequence, and how I know each phase actually worked. I run every command myself; Claude guides one step at a time and reads back the output. `build_log.md` carries the detailed, dated narrative of every step I've actually taken — the `.docx` workbooks that used to accompany this file were retired (2026-08-03) in favor of the `.md` files being the sole source of truth.

**Working principle:** Build incrementally. One change, one test, then the next change. Don't stack multiple untested changes (network config in particular — this environment has a documented history of lockouts from big-bang changes).

**Scope status (2026-08-06):** This document reflects the full, locked scope of the build, start to finish — Phases A through G, including Phase C.6 (AD/kill-chain expansion, now including a second workstation and a fully realistic AD environment) and Phase C.7 (IAM/Entra ID). Everything in it is committed, planned work, assembled across an extended planning conversation. New ideas raised mid-build get logged as next-steps in the relevant `investigations/` writeup or as a note in `PROJECT-INSTRUCTIONS.md` — they don't get folded into active scope without a deliberate decision to revisit and re-lock. See "Explicitly out of scope" near the end for everything already evaluated and deliberately excluded.

---

## Confirmed Environment (as of build start)

### Hardware
| Node | Spec | Role |
|---|---|---|
| `pve01` | Dell PowerEdge R710, Xeon X5675, 64GB RAM, RAID 10, 4x physical NICs | Primary Proxmox host + **always-on service host** — victims, attacker, pfSense, Wazuh, Shuffle, n8n, service UIs |
| `pve-ai` | i9-10900KF (10c/20t, **no integrated graphics**), ASUS TUF Gaming Z590-Plus WiFi, RTX 3070 (8GB VRAM), ~31GB RAM | **Local AI inference node** — GPU muscle only (Ollama). Proxmox VE, static `192.168.0.202`. Built and fully documented 2026-07-12/13 — see below. |
| Cisco Catalyst switch | WS-C2960X-48FPS-L, IOS 15.2(7)E9 | Factory reset complete and confirmed clean; carries VLAN trunk + SPAN monitor port |
| 2x USB-to-Ethernet adapters | — | For SPAN destination (Suricata monitoring NIC) via USB passthrough. Chipset/throughput **must be verified** before Phase C.5 relies on them. NIC4 is the fallback. One adapter is currently in use for PC management access — free it (or confirm the second unit) before Phase C.5. |
| QNAP TS-869 Pro | — | **Excluded from the lab as storage.** Existing home-file-share config left untouched. **One new role only:** may host a Proxmox **QDevice** (quorum tiebreaker) in Container Station once clustering happens — see Clustering below. |

### `pve01` NIC allocation (confirmed working)
| NIC | Linux interface | Bridge | Purpose | Status |
|---|---|---|---|---|
| NIC1 | `eno1` | `vmbr1` | Proxmox host management — `192.168.0.201/24`, gateway `192.168.0.1` confirmed | ✅ Confirmed working |
| NIC2 | `eno2` | `vmbr2` (no IP) | pfSense WAN — plugs into home dumb switch | ✅ Attached to pfSense VM (net0), DHCP lease `192.168.0.131/24` confirmed |
| NIC3 | `eno3` | `vmbr3` (no IP, VLAN-aware) | pfSense LAN — 802.1Q trunk to Cisco switch, carries all lab VLANs | ✅ Attached to pfSense VM (net1), all three VLAN interfaces created and IP-addressed |
| NIC4 | `eno4` | — | Spare — **fallback SPAN destination NIC** if USB adapters prove unreliable | ✅ Confirmed link-up 2026-07-21; otherwise unused |

**Known gotcha #1:** Proxmox does not allow two bridges to claim the same static IP simultaneously, even if one has no physical link — causes ARP ambiguity and silent ping failures. Always confirm with `ip addr show <bridge>` on *both* bridges after any reassignment.

**Known gotcha #2:** Moving a physical cable or reassigning a bridge silently orphans any VM whose network device still points at the old bridge. Check every VM's Hardware tab after any bridge change.

**Known gotcha #3:** A NIC showing no link may simply never have been brought administratively up (`ip link set <iface> up`) — not a hardware fault. Check admin state first. Related: a `build_log.md` entry stating a change was made is not proof it's still true weeks later — verify current state (`ip route show`, etc.) against the live system.

**Known gotcha #4 (from the `pve-ai` build):** the Windows OpenSSH client has a recurring habit of hanging at 0% CPU *after* a remote command has already completed successfully and returned correct output — never before or during. Confirmed repeatedly not to be a host-side problem (host stayed reachable on port 22 throughout). Treat verified output as trustworthy even when the client process itself needs to be killed and the session retried.

**Known gotcha #5 (from the `pve-ai` build):** PowerShell's string-escaping can silently mangle a `sed` command being sent over SSH (e.g. `unterminated 's' command`) before it ever reaches the remote host — a safe no-op, not a partial/dangerous edit, but worth verifying with a read-back (`cat`/`grep`) rather than assuming success. Building the remote command as a single-quoted PowerShell string, rather than double-quoted with escaped inner quotes, avoids the mangling.

**Known gotcha #6 (from the `pve-ai` build):** the i9-10900KF has no integrated graphics — once its RTX 3070 is bound to `vfio-pci` for passthrough, the host's physical monitor goes completely dark at boot. This is expected, not a failure, and doesn't affect SSH/network reachability, since passthrough only changes which driver owns the GPU, not the NIC or networking stack. Worth stating this explicitly before rebooting any host mid-passthrough setup, so a dark screen isn't mistaken for a boot failure.

### Existing VMs on `pve01`
- **Kali** — attack box (runs real network attacks: nmap, Responder, NetExec, etc.). Still on `vmbr1` (flat) as of Phase B — moves to RANGE30 in Phase C.6.
- **Win11-LTSC-victim** — Windows victim. Moved to RANGE30 in Phase B (2026-08-04), isolation proven live, Sysmon + Wazuh agent reporting (agent ID `001`). Domain-joins to `soclab.internal` in Phase C.6; serves as the first lateral-movement hop.
- **`wazuh-host` / `ubuntu-soc-host`** — Docker/Wazuh substrate on INFRA20, built in Phase B, complete.

### Planned new VMs and objects (Phase C.6 / C.7 — not yet built)
| Node / object | Spec | VLAN | Role |
|---|---|---|---|
| `dc01` | Windows Server 2022, Core install, 2 vCPU/4–8GB | **RANGE30** | Domain Controller — new forest, domain **`soclab.internal`**. Treated as an attack target, consistent with everything else on RANGE30, not as trusted infrastructure. |
| `win11-ws02` | Windows 11, real live VM | **RANGE30** | Second, previously-uncompromised workstation — the actual lateral-movement target. Domain-joined. Without a second real host, "lateral movement" would only be reused credentials on the same box, not a genuine pivot. |
| `linux-victim` | Ubuntu Server, minimal | **RANGE30** | Deliberate sudo misconfiguration, `auditd` + Wazuh agent, hosts the vulnerable web app (DVWA or Juice Shop) that provides Initial Access |
| `entra-connect-01` | Windows Server, small | **INFRA20** | Runs Microsoft Entra Connect — bridges `dc01` to the Entra ID tenant. Needs outbound internet and one narrow inbound path to `dc01` — see VLAN plan below. |
| 3–4 "phantom" computer objects | AD computer accounts only, `New-ADComputer`, no live VM behind them | N/A (directory objects only) | Populate the OU structure and give BloodHound/enumeration tooling a realistic-sized environment to map, without the RAM cost of building 3–4 more full VMs. |

**Sizing note:** five additional live Windows VMs was the originally floated number and was judged too heavy against `pve01`'s 64GB total, especially with Shuffle (memory-hungry, flagged in Phase F) and Docker/Wazuh already resident. The mix above — two real, live, fully-instrumented hosts (`dc01`, `win11-ws02`) plus phantom computer objects for the rest — gets the directory-scale realism and BloodHound-mapping value without the resource risk.

### `pve-ai` / `ai-vm` — built, confirmed working (2026-07-12/13)

This was originally tracked as a separate project in a local folder outside this repo and has since been merged into this single documentation set — one project, not two. Its own `build_log.md` and `hybrid_ai_node_build_plan.md` remain as historical source material but are no longer the authoritative, actively-maintained record; this file and the main `build_log.md` are.

- **`pve-ai`** (the Proxmox host itself) = `192.168.0.202`, hostname `pve-ai`. **`ai-vm`** (the Ubuntu Server 24.04 VM on it, VMID 100) = `192.168.0.203`.
- **Role in this lab: local inference only.** Runs Ollama on the RTX 3070. It's the only box that can do GPU inference — the R710 has no usable GPU — so all local AI work physically lives here. Not part of the attack range. Stays reachable to the Infra VLAN so the triage layer and Open WebUI can call it.
- **What's already done (Phases 0–5 of the original ai-node plan, all complete 2026-07-12/13):**
  - BIOS virtualization (VMX, VT-d) enabled; Proxmox VE installed, no-subscription repo enabled (enterprise/ceph repos disabled by rename, not deletion).
  - IOMMU verified via `dmesg` (DMAR/VT-d confirmed at the ACPI level), then actually enabled via `intel_iommu=on iommu=pt` in `GRUB_CMDLINE_LINUX_DEFAULT`, confirmed live via `/proc/cmdline` and `dmesg` showing `IOMMU enabled` post-reboot.
  - IOMMU groups mapped — the RTX 3070 sits **alone** in its own group (Group 1), the clean passthrough case with no ACS override or slot change needed. PCI IDs: `10de:2488` (video, `01:00.0`), `10de:228b` (audio, `01:00.1`).
  - GPU bound to `vfio-pci` via `/etc/modprobe.d/vfio.conf` (`options vfio-pci ids=10de:2488,10de:228b`) plus `/etc/modules` (`vfio`, `vfio_iommu_type1`, `vfio_pci`) and a driver blacklist (`nouveau`, `nvidia`, `nvidiafb`, `snd_hda_intel`) at `/etc/modprobe.d/blacklist-gpu.conf`. Confirmed via `lspci -nnk` showing `Kernel driver in use: vfio-pci` on both GPU functions, stable across reboot. **Side effect, accepted:** onboard motherboard audio is disabled too, since it shares the `snd_hda_intel` module — a non-issue on a headless server.
  - `ai-vm` (Ubuntu Server 24.04, `q35`/OVMF, 16 cores, 26GB RAM — sized down from an original 32GB plan once actual host RAM was confirmed at ~31GB total, leaving 5GB host headroom since VM memory isn't ballooned in this config) created with the RTX 3070 passed through as `hostpci0`.
  - NVIDIA driver (`nvidia-driver-595-open`) installed and verified: `nvidia-smi` shows the RTX 3070, driver `595.71.05`, CUDA `13.2`, `8192MiB` VRAM.
  - Snapshot `clean-gpu-working` taken (Rollback Point 2 in the original plan) — a known-good restore point if anything in the remaining Ollama/Open WebUI work goes wrong.
- **What's left:** Phase 6 of the original plan (Ollama install, a starter model sized for 8GB VRAM, Open WebUI, cloud-model connection) plus this project's own Phase E integration/routing work (n8n router, `agent-registry.md` entry). See Phase E below — this is where the two plans now meet as one.

### Network — current state
- pfSense CE 2.8.1 running (VM 102), all three VLAN interfaces created and reachable — **Phase A complete**: `10.10.10.1/24` MGMT, `10.10.20.1/24` INFRA, `10.10.30.1/24` RANGE.
- Cisco switch factory reset, VLANs 10/20/30 created, `Gi1/0/3` trunk to pfSense, `Gi1/0/48` management access port. Console via USB-serial + PuTTY (COM14, 9600/8/1/None/None).
- iDRAC recovered and secured: `https://192.168.0.100`.

---

## Target AI Architecture (local-first hybrid)

The decided shape, folded into the phases below:

- **Local-first.** Ollama on `pve-ai` (RTX 3070) is the default brain for everything — routine SOC triage and my own daily AI use. Costs electricity, keeps data on-network, unlimited.
- **Cloud supplements, on a threshold.** Escalate to **Claude Code** (`claude -p` headless — draws from my existing subscription, **no per-token API billing**) or **Grok** (its own API, pay-as-you-go) only when a harder/ambiguous case justifies it. I set the escalation threshold.
- **n8n is the router.** n8n makes the local-vs-cloud decision for SOC triage and general lab automation. It's a non-human decision-maker → governed in `agent-registry.md` as `triage-router-01`. **The `pve-ai` node is dedicated to lab/SOC use only — not personal AI use.**
- **Open WebUI is the lab's chat interface for SOC-related work** — points at Ollama by default, with Claude/Grok selectable. Hosted on `pve01` (service host), reached from any browser.
- **Two engines, two lanes:** n8n = AI orchestration + general automation; Shuffle = SOC incident response (Phase F). Kept separate deliberately. **Never build both automation engines in the same stretch** — Shuffle first (Phase F), n8n later with the AI stack.

### Where things run
- **`pve-ai`:** Ollama (GPU inference). Nothing I click on lives here.
- **`pve01` (service host):** pfSense, Wazuh, Suricata, Shuffle, Open WebUI, n8n, `entra-connect-01`, and the Range VMs (`dc01`, `win11-ws02`, `linux-victim`, Win11-LTSC-victim, Kali). All service UIs live here, reached from the browser.
- **Browser / phone (MGMT VLAN):** where I interact with everything.
- **Microsoft Entra ID tenant / Exchange Online:** cloud-hosted, outside `pve01`/`pve-ai` entirely — reached over the internet from INFRA20 (via `entra-connect-01`) and from MGMT10 (via browser).

### Clustering (decided — sequence TBD, low priority)
Cluster `pve01` + `pve-ai` into one Proxmox web UI so both nodes manage from a single pane. Overhead is negligible (corosync + config DB, a few hundred MB RAM — does not touch inference performance). **The real issue is two-node quorum:** if either node goes offline the survivor drops to read-only. Fix with a **QDevice** tiebreaker hosted on the QNAP (Container Station) — the clean path. Manual `pvecm expected 1` is the fallback when a node is intentionally down. Treat clustering as an optional convenience layer; it does not gate any SOC phase.

---

## Target Network Topology

```
Home Router (192.168.0.1) — DHCP/internet for the house
        │
   [dumb switch @ desk] ── My PC (stays here, unaffected)
        │
     NIC2 (pfSense WAN)
        │
   pfSense VM (on pve01) — includes remote-access VPN (Phase C.6)
        │
     NIC3 (pfSense LAN — 802.1Q trunk)
        │
   Cisco Switch (factory-reset, freshly configured)
   ┌────┴────┬──────────────────────┬────────────────────┐
Mgmt VLAN  Infra VLAN             Range VLAN           SPAN monitor port
(my PC     (Wazuh/Docker          (Kali, Win11-         (mirrors Range VLAN
 lab port)  host: Wazuh,           victim, win11-ws02,    traffic, receive-only)
            Suricata,              dc01, linux-victim,          │
            Shuffle,               + phantom computer     USB-Ethernet adapter
            Open WebUI, n8n,       objects in AD)          (passed through to Wazuh
            entra-connect-01,                                host, Suricata listens
            + pve-ai inference)                               here — no IP)

                Microsoft Entra ID tenant + Exchange Online
                (cloud, reached via entra-connect-01 on INFRA20
                 and directly via browser from MGMT10)
```

**Three layers of visibility (as in a real SOC):**
- **Host-based** — Sysmon/`auditd` on victim VMs → Wazuh agent → Wazuh (crosses the Range→Infra fence on TCP 1514/1515 only).
- **Network-based** — SPAN port mirrors Range VLAN traffic → Suricata (passive, no IP) → alerts into Wazuh. Correlating both is the high-value detection-engineering skill this lab demonstrates.
- **Identity-based** — Entra ID sign-in logs, Conditional Access enforcement outcomes, Exchange Online Protection mail-flow logs (Phase C.7).

### VLAN plan
| VLAN | Members | Access rules |
|---|---|---|
| **Management (10)** | My PC (lab port) | Can reach all VLANs for admin (Proxmox UI, Wazuh/Shuffle/n8n/Open WebUI dashboards, pfSense UI, iDRAC, Entra ID admin center, Exchange admin center) |
| **Infra (20)** | Ubuntu SOC host, `pve-ai`, `entra-connect-01`, Shuffle (Phase F) | Receives Range logs on Wazuh's ports only. Needs a defined outbound-internet rule (Docker/apt/`suricata-update`, Ollama model pulls, Shuffle enrichment to VirusTotal/AbuseIPDB, Entra Connect's sync traffic to Microsoft). **A single narrow Pass rule from `entra-connect-01` to `dc01` on RANGE30, for Entra Connect's sync traffic only** — see the callout below. |
| **Range (30)** | Kali, Win11-LTSC-victim, `win11-ws02`, `dc01`, `linux-victim` | **Cannot** reach home network, internet, or Management. Only allowed outbound path: to Wazuh's log-ingest/enrollment ports on Infra. **One narrow, reviewed exception, inbound from Infra:** the Entra Connect sync rule below. |

**Critical pfSense rule (unchanged principle):** Range VLAN → default deny all, explicit allows only. Verify before any Atomic Red Team or Phase C.6 simulation.

**New pattern to expect — INFRA20 → RANGE30 (Phase C.7):** every rule written in this build so far has gone one direction, RANGE30 → INFRA20 (Wazuh ingest/enrollment). Entra Connect breaks that pattern: it runs on INFRA20 but must reach `dc01` on RANGE30 to sync. This is **the first rule ever written in the opposite direction**, and it's the one deliberate exception to the "Range cannot be reached from anywhere" isolation model proven in Phase A.5/B. I'll write this narrowly — single source IP (`entra-connect-01`), single destination IP (`dc01`), AD sync ports only (see Phase C.6's AD port list) — not a general INFRA20↔RANGE30 opening. I write the final rule, reviewed with extra scrutiny given it's the first of its kind in this build.

---

## Build Phases

### Phase A — Network Rebuild ✅ COMPLETE (2026-07-25)
VLANs, trunk, management access port, pfSense VM + VLAN interfaces all built and acceptance-checked. Snapshot `pfsense-clean-install` taken. See `build_log.md`.

### Phase A.5 — Isolation Rule ✅ COMPLETE
1. Write (do not yet apply) a pfSense rule: Range → deny all, except one explicit allow to Infra's Wazuh ingest port.
2. Review the rule text before applying — confirm it doesn't block the interface pfSense is managed from.
3. Apply; acceptance-check from a Range VM (no internet, no home net, no MGMT; Wazuh port retested end of Phase B).

### Phase B — Docker/Git/Wazuh Substrate ✅ COMPLETE (2026-08-04)
1. ✅ Build Ubuntu Server VM on `pve01`, Infra VLAN. **8GB RAM / 4 cores / 100GB+ disk.** Enable OpenSSH. *(2026-07-31 — `wazuh-host`, `10.10.20.100`, SSH confirmed. INFRA20 outbound rules written/tested along the way.)*
2. ✅ Host prep: `sudo sysctl -w vm.max_map_count=262144`, persist in `/etc/sysctl.conf`. *(2026-08-03)*
3. ✅ Install Docker + Compose via the apt repo method (**not** `curl | sh`); verify package names live. *(2026-08-03 — Docker CE 29.7.1, verified with `hello-world`, non-root usage enabled.)*
4. ✅ Install Git, create/clone the repo. Pre-push secret check. *(2026-08-03 — cloned to `~/soc-lab` via a dedicated SSH deploy key. INFRA20 needed a 5th rule, port 22, discovered along the way.)*
5. ✅ Deploy Wazuh (manager + indexer + dashboard) via Compose; check the current tag against `documentation.wazuh.com`. Change the default password. Mount rules host-side (`./config/rules/local_rules.xml`). *(2026-08-03/04 — Wazuh 4.14.6 deployed, admin password rotated via the `internal_users.yml`/`securityadmin.sh` procedure.)*
6. ✅ Move Win11-LTSC-victim to Range VLAN, install Sysmon + Wazuh agent pointed at the Sysmon channel. *(2026-08-04 — isolation proven live, agent ID `001` active.)*
7. Move Kali to Range VLAN — **not done in Phase B, moved to Phase C.6 step 1.**
8. ✅ Re-test the isolation rule: victim reaches Wazuh on 1514/1515, still no internet. Confirmed 2026-08-04.
9. ✅ **Acceptance check:** dashboard shows the victim active with Sysmon events. Confirmed.

**Still open from Phase B:** SSH key-only hardening on `wazuh-host` — not yet done, scheduled alongside Phase C.

---

### Phase C — Detection Engineering ⏳ NOT STARTED
1. Snapshot Win11-LTSC-victim (`pre-atomic-clean`).
2. Install Atomic Red Team **on the victim** (it runs on the box I want telemetry from), reusing the ISO-staging pattern proven in Phase B (download on the management PC → build ISO → attach as virtual CD — Range has no internet by design). Add the `C:\AtomicRedTeam` Defender exclusion — safe only because the VM is fenced (Phase A.5).
3. Run a first technique (`T1059.001`) locally, confirm Wazuh alerts.
4. Write a first custom detection rule. **I write the final rule**; Claude drafts a reference only. Rules live host-side in `config/rules/local_rules.xml`. IDs start at 100000.
5. Commit rules/config-as-code.
6. **Acceptance check:** a committed rule of my own authorship fires on a re-run.

**Also scheduled in this window:** SSH key-only hardening on `wazuh-host`, using the existing `id_ed25519` — the open Phase B follow-up.

---

### Phase C.5 — Network Visibility (SPAN + Suricata) ⏳ NOT STARTED
**Prereq:** verify the USB-Ethernet adapter (`lsusb`, `ip link show`, passthrough test, throughput). Decision gate: reliable adapter → use it; flaky → fall back to NIC4.
1. Configure a SPAN session on the switch: source = Range VLAN, destination = monitor port (receive-only).
2. Physically connect the SPAN destination to the adapter/NIC4.
3. Pass it through to the **Wazuh/Infra host** (not Kali). Monitoring interface gets **no IP**: `ip link set <iface> up` + `promisc on`; confirm no inet.
4. Install Suricata bound to the monitoring interface; integrate `eve.json` into Wazuh.
5. **Acceptance check:** a network attack from Kali (e.g. `nmap -sS`) shows up in Wazuh from Suricata *and* has a host-based alert — two layers, one story.

---

### Phase C.6 — Attack Surface & AD Expansion ⏳ NOT STARTED

**Why:** the lab as built through Phase C.5 proves single-host detection engineering against one isolated Windows victim. It can't demonstrate lateral movement to a genuinely separate host, a real initial-access vector, AD-specific credential and certificate attacks, or Discovery-stage tooling (BloodHound). This phase closes those gaps with a realistic, full-depth Active Directory environment and one coherent, end-to-end kill chain.

**Scope: a real two-host lateral movement chain, deliberately stopping short of full interactive compromise of `dc01` itself.** The chain runs foothold → privilege escalation → credential/certificate access → discovery → lateral movement (to a genuinely separate, previously-uncompromised second workstation) → persistence → exfiltration/impact. Where domain-wide credential material is needed (e.g. via a DCSync misconfiguration), it's obtained through that AD-level abuse rather than by gaining an interactive shell on `dc01` — a real, valid attack path in its own right, but one that stops short of treating `dc01` as a fully compromised host with its own persistence/impact stage. That distinction is a deliberate scope boundary, documented as a next-step in the eventual `investigations/` writeup, not a shortfall.

**Explicitly excludes MITRE Caldera.** Caldera was evaluated as a way to autonomously orchestrate the post-compromise portion of this chain. Decided against: real setup cost (~4–6 hrs) for less time savings than expected, since most of this phase's hours are in understanding techniques and writing detections, not command execution — and it would work against the chosen interview narrative of executing this myself, end to end.

**Sub-steps:**

1. **Infra hardening (do first, before the AD work):**
   - Forward pfSense logs (firewall pass/block, DHCP, DNS) into Wazuh.
   - Move Kali to RANGE30 (still flat on `vmbr1` since Phase B).
   - Configure a pfSense remote-access **WireGuard** VPN — decided over OpenVPN for its simpler configuration surface (fewer places for a subtle rule mistake), better performance, and native integration in pfSense CE since 2.5. A second, credential-based Initial Access vector, distinct from and complementary to the web-app exploit path. I write the final firewall rules, Claude drafts a reference.

2. **AD/DC build:**
   - Stand up `dc01` — Windows Server 2022, Core install, on RANGE30, sized modestly (2 vCPU / 4–8GB). **VM creation and Windows install are Claude Code-executed** — genuinely new territory, delegated as a deliberate time-saving trade-off given the overall scope size, not because it repeats known work.
   - `Install-WindowsFeature AD-Domain-Services` — **Claude Code-executed**, mechanical, no decision content.
   - `Install-ADDSForest` — new forest and domain, **`soclab.internal`** (not `.local`, to avoid mDNS conflicts; this is the forest root, not a join to an existing one). **Manual — I run this myself.** This is where the real decisions live: forest/domain functional level, DNS strategy, NetBIOS name, DSRM password. Claude explains each parameter before I run it.
   - **I choose and record the DSRM password myself** (Keeper, never in docs) — same handling as every other credential in this build. A real break-glass-equivalent for the DC's own boot-level recovery, distinct from but conceptually parallel to the directory's own break-glass account.
   - **I configure Kerberos Policy** (Default Domain Policy — max ticket lifetime, max renewal age, clock skew tolerance) and **the PDC Emulator time source** (`w32tm` setup) myself — this is the concrete, hands-on version of learning Kerberos, not an abstract concept. The clock skew tolerance is directly why NTP (123) is in the AD firewall port list below.
   - **I personally run post-build verification** (`dcdiag`, SYSVOL replication health, confirming DNS is authoritative, reviewing the default OU/GPO structure `Install-ADDSForest` leaves behind) — this is the real "is my DC healthy" acceptance check, not something to accept on a "done" message alone.
   - **Expect a DNS gotcha here** — the single most common AD lab failure point. `dc01`'s own DNS must be authoritative for `soclab.internal`, and every future domain-joined machine must point its DNS at `dc01`, not pfSense or the home router.
   - Write and apply the AD-specific firewall rule set on RANGE30: DNS (53), Kerberos (88), LDAP (389)/LDAPS (636), SMB (445), RPC endpoint mapper (135) plus a dynamic RPC range (49152–65535), NTP (123). Expect this to grow one rule at a time, same pattern as INFRA20's growth from zero to five rules in Phase B. **These rules are Claude Code-executed** — each one repeats the same shape already established for INFRA20, no new judgment required per rule.
   - Join Win11-LTSC-Victim and the new `win11-ws02` to `soclab.internal`. **Both domain joins are Claude Code-executed** (decided 2026-08-04) — real prior hands-on experience with domain joins. **`win11-ws02`'s build (VM creation, Windows install, Sysmon, Wazuh agent enrollment) is also Claude Code-executed** — repeats the exact pattern already proven manually for Win11-LTSC-Victim in Phase B.
   - **Configure Advanced Audit Policy via GPO** — Kerberos service ticket operations (4769), directory service access (4662), object access, and related event categories. **This is a technical requirement, not optional realism:** without it, Windows never generates the events the Kerberoasting, DCSync, and lateral-movement detection rules in this phase actually depend on. I write the final GPO, Claude drafts a reference.

3. **Directory structure, users, groups (RBAC):**
   - Design an OU structure reflecting a tiered admin model — Tier 0 (identity/`dc01`), Tier 1 (servers), Tier 2 (workstations) — even if only partially enforced. I write the final OU/GPO design, Claude drafts a reference. **The design decision itself stays manual; only account creation, OU placement, and group *membership* assignment below are delegated — not group creation.**
   - Create 3–4 phantom computer objects (`New-ADComputer`, no live VM behind them) and place them across the OU structure alongside the real hosts, for directory-scale realism and BloodHound mapping.
   - I create one user account by hand; **Claude Code executes the creation of the remaining ~12** (varied departments/roles, distributed across OUs) — given prior real-world experience creating AD accounts, assigning groups, and placing objects in OUs.
   - Create security groups reflecting realistic nested permissions — including at least one deliberate over-privileged nesting (a low-tier group inheriting Domain Admin-adjacent rights through the nesting itself, not direct membership). **Which groups exist, how they nest, and the actual group-creation commands are manual/two-tier** — I've only ever managed membership on already-existing groups, not created new ones, so this stays alongside the nesting design rather than being delegated. **Assigning existing users to the created groups is Claude Code-executed**, matching real prior experience with group membership assignment specifically.
   - Create one service account with an SPN set and a deliberately weak password — the Kerberoasting target.
   - Create one break-glass/emergency admin account — explicitly not touched during the attack chain, called out as such in the writeup.
   - Create two or three stale/inactive "former employee" accounts, left enabled.

4. **Deliberate misconfigurations (the actual attack surface for Discovery/Credential Access):**
   - A risky ACL: a helpdesk-tier account granted `GenericAll`/`WriteDACL` on a privileged object.
   - Unconstrained Kerberos delegation configured on a server account.
   - DCSync rights (`GetChanges`/`GetChangesAll`) granted to a non-obvious account (e.g. a "backup service" account) — the path to domain-wide credential material without an interactive shell on `dc01`.
   - **Active Directory Certificate Services (ADCS)** deployed with one deliberately vulnerable template (**ESC1** — client-auth EKU plus enrollee-supplies-subject enabled for low-privilege enrollers).
   - SYSVOL/Group Policy Preferences password exposure (a GPP-stored, recoverable credential).
   - A shadow admin — an account with Domain Admin-equivalent *rights* via ACL abuse, without Domain Admins group membership, discoverable only through graph-based analysis, not a group-membership check.
   - **No LAPS deployed initially** — local admin credential reuse across hosts is the gap that enables the lateral-movement pivot from the first compromised host to `win11-ws02`. LAPS gets deployed later in this same phase as the fix, and the pivot re-tested to confirm it's closed — a before/after story mirroring Shuffle's manual-vs-automated comparison in Phase F.

5. **GPO software deployment and shares:**
   - Deploy at least one benign application (7-Zip or Notepad++) via GPO Software Installation.
   - Deploy mapped drives to file shares via Group Policy Preferences.
   - Create department-scoped file shares (Finance, HR, IT) with realistic, imperfect ACLs — at least one deliberately over-broad (e.g. Domain Users granted Read on the Finance share) — this is what makes the later exfiltration stage meaningful rather than arbitrary.

6. **Linux victim and web app:**
   - Build `linux-victim` on RANGE30, deliberate sudo misconfiguration, `auditd` rules (I write the final, Claude drafts a reference) + Wazuh agent.
   - Deploy a vulnerable web app (DVWA or Juice Shop) via Docker Compose.

7. **Attacker tooling on Kali:**
   - Install and configure: Responder, NetExec, Impacket (including `ntlmrelayx.py`), Certipy, Rubeus, PowerView, BloodHound/SharpHound, plus whatever Kerberoasting/AS-REP roasting tooling is already staged.
   - Stand up Sliver C2 (server + implant).

8. **Run the chain, confirming a detection at each stage before moving to the next:**
   - **Initial Access** — exploit the web app, or use the VPN path, to land a shell. Real, manual exploitation.
   - **Windows Privilege Escalation** — unquoted service path / weak service permissions on the first compromised Windows host.
   - **Discovery** — run SharpHound with **session data collection enabled**, not just static structure. In BloodHound: mark high-value targets, run "Shortest Paths to High Value Targets," run the built-in queries for Kerberoastable users, AS-REP roastable users, unconstrained delegation, and DCSync rights, and write at least one hand-crafted Cypher query.
   - **Credential/Certificate Access** — Kerberoasting and/or AS-REP roasting against the SPN'd service account; ADCS ESC1 abuse via Certipy; DCSync via the misconfigured rights above.
   - **Lateral Movement** — pivot from the first compromised host to `win11-ws02`, exploiting the pre-LAPS local admin credential reuse gap, via PsExec/WMI/Pass-the-Hash. Costed and executed as its own distinct step.
   - **Persistence** — a registry run key or scheduled task, or a persistent Sliver implant.
   - **Exfiltration** — pull data from the over-broadly-permissioned file share, exfiltrated via DNS tunneling or HTTPS to an unusual destination.
   - **Impact** — a benign mass file-rename script paired with Wazuh File Integrity Monitoring, simulating ransomware-style behavior.
   - Write a Wazuh (and where applicable Suricata) detection rule for each stage before moving to the next — I write the final rule, Claude drafts a reference.
9. **Deploy LAPS as the fix, re-test the lateral-movement pivot, confirm it's closed** — the explicit before/after comparison.
10. **Write it up as one `investigations/` case** — a single continuous narrative from Initial Access through Impact, including an explicit "next steps" section noting that full interactive compromise of `dc01` itself was intentionally out of scope.
11. **Acceptance check:** the full chain runs start to finish against the live environment in one sitting, with a Wazuh and/or Suricata detection confirmed at every named stage, BloodHound's full analysis workflow demonstrated (not just the collector run), and the `investigations/` writeup accurately reflecting what actually happened.

---

### Phase C.7 — IAM/Entra ID Track ⏳ NOT STARTED

**Why:** directly targets the IAM Analyst role I'm pursuing — Conditional Access, MFA policy, and hybrid identity are core day-to-day IAM work, not just an attack surface. Independent of the rest of the lab's hardware for its first step; ties into Phase C.6 once `dc01` exists.

**Sub-steps:**

1. **Entra ID Free tenant** — sign up (no time limit; the Free tier is permanent). Build out a basic directory structure in parallel with other phases.
2. **Entra Connect (hybrid identity)** — build `entra-connect-01` on INFRA20, install Microsoft Entra Connect, sync `dc01` → Entra ID once Phase C.6's DC exists. **VM creation and the Entra Connect software install are Claude Code-executed** — mechanical, no decision content. **Everything after that is manual:** free-tier capable — full user/group/attribute sync plus one-way Password Hash Sync; only password *writeback* specifically needs P1. Requires the narrow INFRA20→RANGE30 rule described above (I write the final rule), plus outbound rules for `entra-connect-01` to reach Microsoft's sync endpoints. I verify sync landed correctly and test SSO against a cloud resource myself. **I confirm the sync account's scope in `agent-registry.md`** — dedicated service account, minimum AD replication/read permissions, not a Domain Admin account — before the sync goes live; this scoping is the actual security teaching point here (the sync account is a well-known real-world attack target, the same risk pattern the DCSync misconfiguration in Phase C.6 demonstrates), not administrative overhead, so it stays manual regardless of what else is delegated.
3. **Exchange Online (Plan 1, $4/user/month)** — real mailboxes tied to the synced users. **Mailbox creation is Claude Code-executed** — mechanical, similar to AD account creation. **Attack Simulation Training campaign design stays manual** — choosing scenarios and analyzing results is the actual phishing-detection learning. Delivered via Defender for Office 365's **Attack Simulation Training** — Microsoft's own sanctioned tool, chosen specifically to avoid automated-abuse-detection ambiguity. Mail-flow/EOP logs become a genuine detection source.
4. **P1 trial activation (time-boxed, 30 days) — activate last, only once ready to use it immediately:**
   - Build and test Conditional Access policies against the synced hybrid users.
   - Run MFA scenarios via the Microsoft Authenticator app for real push notifications.
5. **AADInternals / ROADtools** — cloud identity enumeration, the Entra-ID equivalent of BloodHound. The deliberate connective piece between Phase C.6 and C.7 — extending the on-prem chain into the cloud identity layer, one continuous compromise story rather than two disconnected efforts.
6. **Acceptance check:** hybrid identity confirmed working end to end; at least one Conditional Access policy built, tested, and enforcement confirmed live during the P1 window; AADInternals/ROADtools enumeration successfully run with a corresponding detection or log reviewed.

**Explicitly out of scope:** Entra ID P2 (Identity Protection's risk-based sign-in scoring, Privileged Identity Management, Access Reviews) — more architect-tier than day-to-day analyst work.

---

### Phase D — AI Triage Layer + Agent-Identity Governance ⏳ NOT STARTED
Subscription-covered Claude Code / local model — no separate API billing for Claude Code.
1. **Give the triage agent its own identity, not mine** — dedicated Wazuh API user, not admin, not my own session.
2. **Scope it least-privilege** — `alerts:read`, `agent:read`; no delete, no config, nothing outside the Infra alert pipeline.
3. **Separate its audit trail** — every action logs `agent=wazuh-triage-01`; my review logs `user=analyst` (see `PROJECT-INSTRUCTIONS.md` for the exact identifier convention).
4. **Create/maintain `agent-registry.md`** — the living record of every AI agent.
5. Build the triage service on the Infra host: pull new alerts → model → verdict suggestion, under the agent's own credential.
6. I review/confirm/override, logged separately.
7. **Acceptance check:** an alert flows detection → Wazuh → agent verdict → reviewed verdict, both logged distinguishably; registry matches reality.

This phase benefits directly from Phase C.6 having run first — a full kill chain produces a much richer, more realistic alert stream for the triage layer to work against than isolated Atomic Red Team tests alone, which is part of why C.6/C.7 are sequenced ahead of D.

---

### Phase E — Local AI Model Integration + n8n Routing ⏳ NOT STARTED (partially complete — GPU node built)

The underlying `pve-ai`/`ai-vm` infrastructure (GPU passthrough, Ubuntu VM, NVIDIA driver) is already built and confirmed working — see "Confirmed Environment" above. What's left picks up at Ollama itself:

1. **Install Ollama on `ai-vm`** (official install script), verify the service is running.
2. **Pull a starter model sized for 8GB VRAM** — `llama3.1:8b` (quantized default). Test with `ollama run llama3.1:8b "hello"`, confirming via `nvidia-smi` that GPU memory is actually in use, not a silent CPU-only fallback.
3. **Install Open WebUI** (Docker method) on `ai-vm`, bound to its IP, connected to the local Ollama instance.
4. **Configure Open WebUI's cloud connection** — add the Anthropic API under external connections. I supply the API key myself directly in the Open WebUI settings; Claude never asks for it in chat.
5. **Verify from the management PC:** Open WebUI loads in a browser, can chat with both the local model and the cloud model from the same interface.
6. **Snapshot `ai-stack-working`** once confirmed.
7. **Confirm `ai-vm` reachable from the Infra host** and Ollama's API responding (`curl http://192.168.0.203:11434/api/tags`) — requires an Infra→ai-vm route/rule if not already permitted.
8. **Stand up n8n** (service container on `pve01`) as the router: routine/low-severity → local Ollama; ambiguous/high-severity → escalate to `claude -p` (subscription) or Grok. I write the routing threshold.
9. **Register the router in `agent-registry.md`** as `triage-router-01` — a second non-human identity making autonomous escalate/local decisions.
10. **Acceptance check:** one full triage cycle completes on the local model with no external call; a second escalates correctly; registry entry accurate.

---

### Phase F — SOAR (Shuffle) ⏳ NOT STARTED
**Why:** SOAR (Security Orchestration, Automation, and Response) is directly named on the SOC job listings I'm targeting. Shuffle is the community-standard open-source SOAR engine and pairs natively with Wazuh. Slots in after the detection/attack-surface work (Phase C through C.7) — Phase C.6's full kill chain in particular gives Shuffle's first playbook something substantial to react to.

**Lane discipline:** Shuffle = SOC incident response. n8n = AI orchestration + general automation. Do not blur them; do not build Shuffle and n8n in the same stretch.

1. Build a dedicated **Shuffle VM** on `pve01`, Infra VLAN (own VM — do not co-locate with Wazuh; both are memory-hungry). Docker deploy. **Register `shuffle-playbook-01` in `agent-registry.md`** with its own Wazuh API credential (separate from `wazuh-triage-01`'s) and its own enrichment API keys — before wiring it to anything live.
2. Wire Wazuh → Shuffle: add a `<integration>` webhook block in Wazuh so alerts POST to a Shuffle workflow.
3. Build a first playbook (I write the final logic, Claude drafts a reference): alert in → parse IOCs → **light enrichment** (VirusTotal/AbuseIPDB via HTTP) → notify.
4. **(Optional, deliberate)** Active-response step (e.g. block an IP via pfSense). Gated on an explicit, separately-reviewed firewall decision. Not built by default. If chosen, `shuffle-playbook-01`'s pfSense credential is scoped to nothing but adding entries to one specific block-list alias — nothing else — per `agent-registry.md`.
5. Commit the playbook/config-as-code.
6. **Acceptance check:** a real detection fires (ideally from Phase C.6's chain) → Shuffle receives it → enriches → produces a case/notification end to end.

---

### Phase G — Case Management (TheHive + Cortex) — DEFERRED
**Not started. Added later as one matched-pair unit once everything else is stable.**
- **TheHive** = case management (tickets, timeline, tasks, observables, status).
- **Cortex** = enrichment/analysis engine, overlapping with Shuffle's light enrichment but systematic.
- **Why deferred:** TheHive requires **Cassandra** (case DB) + **Elasticsearch** (search/index) as backends, all version-matched. Coexisting with Wazuh's own indexer is the specific friction point. Realistic cost: ~4–8 sessions, mostly version-matching and startup-order issues.
- **Sequence:** attempt only after Phases F and E are solid. Wants its own dedicated VM.
- `investigations/` writeups in Git serve as case documentation in the meantime.

---

## Time estimate — locked scope

| Block | Manual (hrs) | Automated (Claude Code) |
|---|---|---|
| Phase C | 5–7 | — |
| SSH key-only hardening, `wazuh-host` | 0.5–1 | — |
| Phase C.6 (infra hardening, VPN, full AD build design + misconfigurations, expanded Kali tooling, full kill chain incl. Discovery/BloodHound workflow, LAPS before/after, stitching + writeup) | 38–52 | ~5.5–9 |
| Phase C.7 (Entra ID Free + Entra Connect + Exchange Online + P1 trial + AADInternals/ROADtools) | 11–19 | ~1.5–2.5 |
| Phase D | 5.5–9 | — |
| Phase E | 4.5–6.5 | — |
| Phase F | 4.5–8.5 | — |
| **Total (Phase C through F; Phase A/A.5/B complete; Phase C.5 unaffected by this revision)** | **~69–103 hrs** | **~7–11.5 hrs** |

**What's automated, final version (2026-08-06, revised):** `dc01`'s VM creation, base Windows Server install, and AD DS role install only — **not** `Install-ADDSForest` itself, which is where the real decisions live (forest/domain functional level, DNS strategy, DSRM password) and stays manual, along with Kerberos Policy configuration, the PDC Emulator time source, and post-build verification. `win11-ws02`'s full build (VM/Windows/Sysmon/Wazuh agent), domain-joining both workstations, AD account creation/OU placement/group membership assignment (not group creation), the AD port-list firewall rules on RANGE30, Entra Connect's VM creation and software install (not sync account scoping or the firewall rule), and Exchange Online mailbox creation (not Attack Simulation Training design) are all Claude Code-executed. Everything with real security or design judgment — the OU/tiered-model design, group creation, GPOs, all six deliberate misconfigurations, the WireGuard VPN configuration, the INFRA20→RANGE30 Entra Connect rule, Conditional Access/MFA testing, and AADInternals/ROADtools — stays fully manual. See "Execution model exceptions" in `PROJECT-INSTRUCTIONS.md` for the full reasoning.

At ~15–17 hrs/week (baseline pace inferred from `build_log.md`, plus a 5 hr/week weekday addition): **roughly 4.1–6.9 weeks**, realistically **4–7 weeks**.

---

## Explicitly out of scope (documented next-steps, not live plan)

- **MITRE Caldera** — post-compromise attack-chain orchestration. Would work against the "I executed this myself, end to end" narrative for a modest time saving. Revisit only with runway to spare after the locked plan is complete.
- **Entra ID P2** (Identity Protection, PIM, Access Reviews) — architect-tier, not day-to-day analyst work.
- **On-prem Exchange Server** — real complexity/troubleshooting risk for marginal skill gain over Exchange Online given the target roles.
- **Standalone GoPhish + a lightweight mail relay** — superseded by Exchange Online + Defender for Office 365 Attack Simulation Training.
- **Full interactive host-level compromise of `dc01`** (gaining a shell on the DC itself, planting persistence there, running impact against it) — the DCSync misconfiguration provides a genuine, real attack path to domain-wide credential material without this; a fully compromised DC as its own stage is documented as a writeup next-step.

---

## Standing Rules (carried over, apply throughout)
- Ask my approval before creating any file, document, image, or phase not already scoped here — including new next-steps identified mid-build: log them, don't build them, unless there's a deliberate decision to revisit scope.
- Warn me before analyzing screenshots.
- Flag and check in before pursuing tangents.
- One network change at a time, tested before the next.
- Confirm console/recovery access before any change that could lock out management.
- Core security logic (firewall rules, VLAN/trunk config, detection rules, Shuffle playbooks, agent scoping, AD/GPO configuration, OU/group design, Conditional Access policies, `auditd` rules, and especially the new INFRA20→RANGE30 Entra Connect rule) — Claude drafts a reference, **I write the final**.
- Explain-it-back checkpoint after each phase.
