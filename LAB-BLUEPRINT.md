# LAB-BLUEPRINT.md
### Home SOC Lab — what we're building, and in what order

**Purpose:** The shared map. What gets built, in what sequence, and how you know each phase actually worked. Michael runs the commands; Claude guides one step at a time and reads back the output. The Word companions carry the detail: `SOC-Lab-Phase-Checklists.docx` (the printed workbook, with commands) and `SOC-Lab-Lesson-Plans.docx` (the concepts behind each phase).

**Working principle:** Build incrementally. One change, one test, then the next change. Do not stack multiple untested changes (network config in particular — this environment has a documented history of lockouts from big-bang changes).

---

## Confirmed Environment (as of build start)

### Hardware
| Node | Spec | Role |
|---|---|---|
| `pve01` | Dell PowerEdge R710, Xeon X5675, 64GB RAM, RAID 10, 4x physical NICs | Primary Proxmox host — victims, attacker, pfSense, Wazuh |
| `pve-ai` | i9-10900KF, RTX 3070 | Local AI inference node |
| Cisco Catalyst switch | WS-C2960X-48FPS-L, IOS 15.2(7)E9 | Factory reset complete and confirmed clean; will carry VLAN trunk + SPAN monitor port |
| 2x USB-to-Ethernet adapters | — | For SPAN destination (Suricata monitoring NIC) via USB passthrough. Chipset/throughput **must be verified** before Phase C.5 relies on them (see Phase C.5). NIC4 is the fallback if they prove unreliable. |
| QNAP TS-869 Pro | — | **Excluded from this lab.** Existing home-file-share config left untouched. |

### `pve01` NIC allocation (confirmed working)
| NIC | Linux interface | Bridge | Purpose | Status |
|---|---|---|---|---|
| NIC1 | `eno1` | `vmbr1` | Proxmox host management — `192.168.0.201/24`, gateway `192.168.0.1` confirmed | ✅ Confirmed working |
| NIC2 | `eno2` | `vmbr2` (no IP) | pfSense WAN — plugs into home dumb switch | ✅ Attached to pfSense VM (net0), DHCP lease `192.168.0.131/24` confirmed |
| NIC3 | `eno3` | `vmbr3` (no IP, VLAN-aware) | pfSense LAN — 802.1Q trunk to Cisco switch, carries all lab VLANs | ✅ Attached to pfSense VM (net1), all three VLAN interfaces created and IP-addressed |
| NIC4 | `eno4` | — | Spare — **fallback SPAN destination NIC** if USB adapters prove unreliable; otherwise backup/migration link | ✅ Confirmed link-up 2026-07-21; otherwise unused |

**Current bridge state (confirmed working):** All existing VMs — **Kali** and **Win11-LTSC-victim** — are consolidated on `vmbr1` (NIC1) as the single live bridge, and both reach the internet. `vmbr2` (NIC2) carries pfSense's WAN NIC, live with a DHCP lease from the dumb switch. `vmbr3` (NIC3) carries pfSense's LAN trunk NIC — VLAN-aware, trunk confirmed passing VLANs 10/20/30, VLAN sub-interfaces not yet created inside pfSense. `pve01` itself is fully patched (kernel `7.0.14-5-pve` as of 2026-07-21).

**Known gotcha #1, already hit once:** Proxmox does not allow two bridges to claim the same static IP simultaneously, even if one has no physical link — causes ARP ambiguity and silent ping failures. Always confirm with `ip addr show <bridge>` on *both* bridges after any reassignment.

**Known gotcha #2, already hit once:** Moving a physical cable or reassigning a bridge silently orphans any VM whose network device still points at the old bridge — the VM shows "network unreachable" with no obvious cause. When Phase A builds the VLAN-specific bridges, **each VM's network device must be deliberately re-pointed** to its correct VLAN bridge, not left on `vmbr1` by default. Check every VM's Hardware tab after any bridge change.

**Known gotcha #3, already hit once:** A NIC showing no link and failing cable/BIOS/iDRAC-log checks may simply never have been brought administratively up (`ip link set <iface> up`) — not a hardware fault. Check admin state first on any future "no link" symptom. Related: a `build_log.md` entry stating a change was made (e.g. vmbr1's default gateway) is not proof it's still true weeks later — verify current state (`ip route show`, etc.) against the live system rather than trusting the log.

### Existing VMs on `pve01`
- **Kali** — currently on `vmbr1`, internet-reachable. Becomes the attack box (runs Atomic Red Team simulations). Moves to Range VLAN in Phase B.
- **Win11-LTSC-victim** — currently on `vmbr1`, internet-reachable. Becomes the Windows victim (needs Sysmon installed). Moves to Range VLAN in Phase B.
- No jump box exists — one new Ubuntu VM will be built as the Docker/Git/Wazuh host (Phase B), placed on Infra VLAN.

### `pve-ai` status
- **`pve-ai`** (the Proxmox host itself) = `192.168.0.202`. **`ai-vm`** (the Ubuntu Server VM on it) = `192.168.0.203`. Both confirmed — no discrepancy, just two different things.
- Built and tracked under its own separate, authoritative plan: `C:\Users\micha\lab\ai-node\hybrid_ai_node_build_plan.md`, with progress logged in the companion `build_log.md` in that same folder. **Do not duplicate that plan's steps here** — Phase E below defers to it entirely.
- Ready for Phase 6 of that plan (Ollama + Open WebUI), not yet started.
- Not part of the attack range. Stays on the standard lab VLAN, untouched by isolation changes.

### Network — current state
- Home router (192.168.0.1) does DHCP/routing for everything today — flat network, no segmentation
- pfSense CE 2.8.1 installed and running (VM 102), all three VLAN interfaces (`10.10.10.1/24` MGMT, `10.10.20.1/24` INFRA, `10.10.30.1/24` RANGE) created and confirmed reachable — **Phase A complete**
- Cisco switch identified (WS-C2960X-48FPS-L, IOS 15.2(7)E9; interface naming `GigabitEthernet1/0/1`–`1/0/52` plus `Fa0` dedicated mgmt port), factory reset complete and confirmed clean (`show run`: hostname default, no VLANs, no passwords), console access via USB-serial + PuTTY confirmed working (COM14, 9600/8/1/None/None). Currently powered down. **Ready for Phase A step 4 (VLAN creation).**
- iDRAC recovered and secured: `https://192.168.0.100`, DHCP reservation set on home router

---

## Target Network Topology

```
Home Router (192.168.0.1) — DHCP/internet for the house
        │
   [dumb switch @ desk] ── Your PC (stays here, unaffected)
        │
     NIC2 (pfSense WAN)
        │
   pfSense VM (on pve01)
        │
     NIC3 (pfSense LAN — 802.1Q trunk)
        │
   Cisco Switch (factory-reset, freshly configured)
   ┌────┴────┬─────────────┬──────────────┐
Mgmt VLAN  Infra VLAN   Range VLAN     SPAN monitor port
(your PC   (Wazuh/Docker (Kali +        (mirrors Range VLAN traffic,
 lab port)  host, pve-ai) Win11-victim)  receive-only)
                                              │
                                    USB-Ethernet adapter
                                    (passed through to Wazuh host,
                                     Suricata listens here — no IP)
```

**Two layers of visibility (as in a real SOC):**
- **Host-based** — Sysmon on the Win11 victim → Wazuh. Tells you what happened *on* the endpoint.
- **Network-based** — SPAN port mirrors Range VLAN traffic → Suricata on the Wazuh host → alerts into Wazuh. Tells you what happened *on the wire* (lateral movement, C2 beaconing, DNS tunneling), including from devices with no agent. Correlating both is the high-value detection-engineering skill this lab demonstrates.

### VLAN plan
| VLAN | Members | Access rules |
|---|---|---|
| **Management** | Your PC (lab port) | Can reach all VLANs for admin purposes (Proxmox UI, Wazuh dashboard, pfSense UI, iDRAC) |
| **Infra** | New Ubuntu/Wazuh host, `pve-ai` | Receives logs from Range VLAN on Wazuh's port only. No inbound from Range otherwise. |
| **Range** | Kali, Win11-LTSC-victim | **Cannot** reach home network, internet, or Management VLAN. Only allowed path: outbound to Wazuh's log-ingest port on Infra VLAN. |

**Critical pfSense rule:** Range VLAN → default deny all, single explicit allow rule to Infra VLAN's Wazuh port. This is the safety boundary — verify this rule before any Atomic Red Team simulation is run.

---

## Build Phases

### Phase A — Network Rebuild (safety-gated)
**Gate before starting:**
1. Confirm physical console access to both the Cisco switch (USB-serial, PuTTY — already confirmed working) and `pve01` (iDRAC at 192.168.0.100, confirmed working; direct monitor/keyboard as backup, confirmed working).
2. Create `build_log.md` in the repo root and begin logging immediately — every command, actual output, decision, and rollback point, updated as you go.
3. Turn on session recording before touching anything: PuTTY logging for the switch, `Start-Transcript` in PowerShell, `script` on Linux hosts (see `CLAUDE.md` → Session recording). The prior lockout incident happened with no transcript; that gap is what made recovery impossible.

1. ✅ **Done (2026-07-17)** — Factory reset Cisco switch: `write erase`, `reload`
2. ✅ **Done (2026-07-17)** — Confirm console access to the clean switch
3. ✅ **Done (2026-07-24)** — Build pfSense VM on `pve01`:
   - Bridges: NIC2 → `vmbr2` (WAN side, no IP) ✅, NIC3 → `vmbr3` (LAN side, VLAN-aware, no IP) ✅
   - VM 102 (`pfsense`) created, pfSense CE 2.8.1 installed, booted successfully. `net0`→`vmbr2` (WAN, DHCP, got `192.168.0.131/24` from the dumb switch), `net1`→`vmbr3` (LAN trunk, factory-default `192.168.1.1/24` pending replacement in Step 7).
   - VM resources temporarily bumped to 3 cores/4GB RAM to push through a slow first boot — revisit and right-size after steady-state load is known.
   - Snapshot the VM now that initial setup is complete, before any firewall rules are added
4. ✅ **Done (2026-07-23)** — Configured switch: three VLANs created (Management, Infra, Range), NIC3's switch port (`Gi1/0/3`) as an 802.1Q trunk carrying all three VLANs, PC's management port (`Gi1/0/48`, via dedicated USB-Ethernet adapter link) assigned to the Management VLAN
5. ✅ **Done (2026-07-25)** — In pfSense, created matching VLAN interfaces on the LAN side: `10.10.10.1/24` (MGMT), `10.10.20.1/24` (INFRA), `10.10.30.1/24` (RANGE), each with DHCP enabled. Ran the setup wizard — unchecked "Block RFC1918 Private Networks" on WAN (required, since WAN's upstream is the private home network, not a real ISP).
6. ✅ **Acceptance check passed (2026-07-25):** PC (on Management VLAN, via dedicated USB-adapter link) reached pfSense's web UI at `https://10.10.10.1/`. **Phase A is complete.** Snapshot `pfsense-clean-install` taken as a rollback point before any firewall rules exist.

### Phase A.5 — Isolation Rule
1. Write (do not yet apply) a pfSense firewall rule: Range VLAN → deny all, except one explicit allow to Infra VLAN's Wazuh ingest port
2. Review the rule text before applying — confirm it does not accidentally block the interface pfSense is managed from
3. Apply the rule
4. **Acceptance check:** from a VM on the Range VLAN, confirm you cannot ping the home network or internet, but can reach the Wazuh host's designated port once it exists (retest this specific check again at the end of Phase B)

### Phase B — Docker/Git/Wazuh Substrate
1. Build new Ubuntu Server VM on `pve01`, place on Infra VLAN. **Size it for Wazuh: 8GB RAM / 4 cores / 100GB+ disk.** Wazuh's own docs call for ~6GB minimum for the stack; the indexer will thrash below that. Enable OpenSSH during install.
2. Host prep — the Wazuh indexer requires an increased memory-map limit or it will fail to start: `sudo sysctl -w vm.max_map_count=262144`, and persist it in `/etc/sysctl.conf`
3. Install Docker + Docker Compose. **Do not use `curl | sh`** — this is a security lab; use the apt repository method per `docs.docker.com`, and verify package names against the live system rather than assuming.
4. Install Git, create/clone the lab's GitHub repo. Before the first push, check for anything that shouldn't be public — Git history is permanent and this repo is the portfolio.
5. Deploy Wazuh via Docker Compose (manager + indexer + dashboard). Check the current release tag against `documentation.wazuh.com` (or `git tag -l`) rather than a hardcoded version:
   - `git clone https://github.com/wazuh/wazuh-docker.git -b <current-tag>`, then `cd wazuh-docker/single-node`
   - `docker compose -f generate-indexer-certs.yml run --rm generator`
   - `docker compose up -d` — indexer takes ~1 min; "not ready yet" messages during startup are normal
   - Change the default password. Dashboard is on port 443.
   - **Mount rules host-side now** (in `docker-compose.yml` under `wazuh.manager` volumes): `./config/rules/local_rules.xml:/var/ossec/etc/rules/local_rules.xml`. This is what makes Phase C's rules committable and rebuild-proof.
6. Move **Win11-LTSC-victim** to Range VLAN — **check its Hardware tab afterward** (known gotcha #2). Install Sysmon with a real config, install the Wazuh agent, and point it at the Sysmon event channel (`Microsoft-Windows-Sysmon/Operational`, `eventchannel` format).
7. Move **Kali** to Range VLAN — check its Hardware tab too
8. Re-test Phase A.5's isolation rule now that Wazuh exists: victim must reach Wazuh on 1514, and must still fail to reach the internet
9. **Acceptance check:** Wazuh dashboard shows the Win11 victim as an active, reporting agent, with Sysmon events visible

### Phase C — Detection Engineering
**Correction to an earlier assumption in this file:** Atomic Red Team is a PowerShell *test-execution framework*, not a remote attack tool. It runs **on the machine you want telemetry from**. So `Invoke-AtomicRedTeam` installs on **Win11-LTSC-victim** and runs there — not on Kali. Kali remains the box for genuine network-based attacks (nmap, responder, etc.), which is what feeds Suricata in Phase C.5. Remote execution from Kali is possible via PowerShell Remoting over SSH (`Invoke-AtomicTest -Session $sess`), but that requires PowerShell Core on Kali and PSRemoting configured on the victim — treat it as an advanced follow-up once local execution works.

1. Snapshot Win11-LTSC-victim (`qm snapshot <vmid> pre-atomic-clean`) — atomics leave the box dirty
2. On **Win11-LTSC-victim**, install the framework and atomics (PowerShell as admin):
   - `Set-ExecutionPolicy Bypass -Scope Process`
   - `IEX (IWR 'https://raw.githubusercontent.com/redcanaryco/invoke-atomicredteam/master/install-atomicredteam.ps1' -UseBasicParsing); Install-AtomicRedTeam -getAtomics`
   - Defender **will** quarantine the atomics folder — that is Defender working correctly. Add an exclusion for `C:\AtomicRedTeam`. This is only acceptable because the VM is fenced on the Range VLAN (Phase A.5) — that isolation is the prerequisite that makes this safe.
3. Run a first technique locally and confirm Wazuh alerts on it:
   - `Invoke-AtomicTest T1059.001 -ShowDetailsBrief` (read before you run)
   - `Invoke-AtomicTest T1059.001 -GetPrereqs`, then `Invoke-AtomicTest T1059.001`, then `-Cleanup`
4. Write a first custom Wazuh detection rule. **Michael writes the final rule** — Claude explains the options and drafts a reference version only (tier-two rule, see the Project instructions). Rules live host-side in `config/rules/local_rules.xml`, mounted into the manager container, so they survive rebuilds and can be committed. User-defined rule IDs start at 100000.
5. Commit rules and any config-as-code to the Git repo
6. **Acceptance check:** a committed rule of Michael's authorship fires correctly against a re-run of the same technique

### Phase C.5 — Network Visibility (SPAN + Suricata)
**Prerequisite verification (do this before building the rest of the phase):**
1. Plug a USB-Ethernet adapter into `pve01`. On the Proxmox host, run `lsusb` to identify the chipset (ASIX AX88179 or Realtek RTL8153 are well-supported; note whatever it actually is).
2. Run `ip link show` on the host — confirm a new interface appears (native recognition = good sign for passthrough).
3. Pass the USB device through to a test VM (Kali is fine as a stand-in), boot it, confirm the interface appears inside the VM with `ip link show`.
4. Sanity-check throughput (large file copy or `iperf3`) — cheap/USB2 adapters can drop packets under load.
5. **Decision gate:** if the adapter is reliable → proceed with it. If flaky/unsupported → fall back to dedicating physical **NIC4** as the SPAN destination instead. Either path is valid.

**Build:**
1. On the Cisco switch, configure a SPAN session (`monitor session`): source = Range VLAN (or the Range access ports), destination = the monitor port. SPAN destination ports are receive-only.
2. Physically connect the SPAN destination port to the USB-Ethernet adapter (or NIC4).
3. Pass that adapter/NIC through to the **Wazuh/Infra host** (not Kali — Kali was only the compatibility stand-in). The monitoring interface gets **no IP address** and is not part of any routed bridge — it just listens.
4. Install Suricata on the Wazuh host, bind it to the monitoring interface.
5. Integrate Suricata alerts into Wazuh (Wazuh can ingest Suricata's `eve.json` output).
6. **Acceptance check:** run an Atomic Red Team technique that generates network traffic (e.g., a simulated C2 callback or port scan) from Kali against the victim, and confirm Suricata flags it in Wazuh *in addition to* any host-based Sysmon alert — demonstrating the two layers correlating.

### Phase D — AI Triage Layer + Agent-Identity Governance (Claude API via subscription, no separate billing)

**Why this phase includes identity governance, not just a triage script:** an AI agent calling APIs and touching alert data is a non-human identity, and it should be governed the same way any account is — its own credential, least-privilege scope, its own audit trail, and a documented owner/lifecycle. This mirrors real IAM practice and is the differentiator this lab is built to demonstrate (see `LESSON-PLAN-1-BUILDER.md` Module 7 for the concept).

1. **Give the triage agent its own identity, not yours.** Create a dedicated API credential/service account for the triage service — do not reuse your personal Claude subscription session or any personal API key for this service's calls. If a scoped API key is used, treat it as a distinct, tracked credential.
2. **Scope it to least privilege.** The service's Wazuh access should be read-only on alerts (and write-only to a verdict/comment field if Wazuh supports it) — it should not have broader Wazuh admin rights, and it should have no access to anything outside the Infra VLAN's alert pipeline.
3. **Separate its audit trail from yours.** Every action the agent takes (alert pulled, verdict suggested) logs under an identifiable agent name/ID — distinct from log entries representing your own human review/override. The two should never be ambiguous in the log.
4. **Create `agent-registry.md`** in the repo root: a short, living record of every AI agent in this lab (starting with this triage agent, later the local-model router in Phase E). For each agent, record: name/ID, purpose, what it's allowed to access, owner (you), credential type and rotation plan, current lifecycle status (active/retired), and date of last review.
5. Build the triage service itself (on the Infra VLAN host): pulls new Wazuh alerts, sends them to a model API using the agent's own scoped credential, returns a triage summary/verdict suggestion.
6. Human (you) reviews and confirms/overrides the AI's verdict, logged separately per point 3 — this is the core "AI-augmented analyst" workflow this whole lab exists to demonstrate.
7. **Acceptance check:** an Atomic Red Team-generated alert flows end-to-end: detection → Wazuh alert → agent (under its own identity) triage suggestion → your reviewed verdict, both logged distinguishably. `agent-registry.md` accurately reflects the agent used.

### Phase E — Local AI Model Integration (defers to the existing ai-node plan)
**This phase does not build the local AI stack itself** — that work is owned entirely by `C:\Users\micha\lab\ai-node\hybrid_ai_node_build_plan.md` (Phase 6 of that plan: Ollama + Open WebUI). Check that plan's `build_log.md` for current status before starting this phase.

Once that plan's Phase 6 is complete, this SOC lab phase only adds the *integration* between the two projects:
1. Confirm `ai-vm` (`192.168.0.203`) is reachable from the Wazuh/Infra host and has Ollama's API responding
2. Route simple/cheap triage calls from the Phase D triage service to the local model on `ai-vm`; escalate ambiguous or high-severity alerts to the cloud model
3. **Add the local-model router to `agent-registry.md`** as its own entry — same governance treatment as the Phase D agent, since it's a second non-human identity making autonomous decisions (which alerts to escalate)
4. **Acceptance check:** at least one full triage cycle completes using the local model with no external API call, and the registry entry is accurate

---

## Standing Rules (carried over, apply throughout)
- Ask approval before creating any file, document, image, or phase not already explicitly scoped here
- Warn before analyzing screenshots
- Flag and check in before pursuing tangents
- One network change at a time, tested before the next
- Console/recovery access confirmed before any change that could lock out management
