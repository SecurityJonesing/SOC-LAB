# LAB-BLUEPRINT.md
### Home SOC Lab — what we're building, and in what order

**Purpose:** The shared map. What gets built, in what sequence, and how you know each phase actually worked. Michael runs the commands; Claude guides one step at a time and reads back the output. `build_log.md` carries the detailed, dated narrative of every step actually taken — the `.docx` workbooks that used to accompany this file were retired (2026-08-03) in favor of the `.md` files being the sole source of truth.

**Working principle:** Build incrementally. One change, one test, then the next change. Do not stack multiple untested changes (network config in particular — this environment has a documented history of lockouts from big-bang changes).

---

## Confirmed Environment (as of build start)

### Hardware
| Node | Spec | Role |
|---|---|---|
| `pve01` | Dell PowerEdge R710, Xeon X5675, 64GB RAM, RAID 10, 4x physical NICs | Primary Proxmox host + **always-on service host** — victims, attacker, pfSense, Wazuh, Shuffle, n8n, service UIs |
| `pve-ai` | i9-10900KF, RTX 3070 (8GB VRAM) | **Local AI inference node** — GPU muscle only (Ollama). No usable iGPU once GPU passthrough is active. |
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

### Existing VMs on `pve01`
- **Kali** — attack box (runs real network attacks: nmap, responder). Moves to Range VLAN in Phase B.
- **Win11-LTSC-victim** — Windows victim (Sysmon + Wazuh agent + Atomic Red Team run locally). Moves to Range VLAN in Phase B.
- **Ubuntu SOC host** — built in Phase B on Infra VLAN. Docker host for Wazuh, Suricata, Shuffle (Phase F), and the service-side containers (Open WebUI, n8n).

### `pve-ai` status
- **`pve-ai`** (the Proxmox host itself) = `192.168.0.202`. **`ai-vm`** (the Ubuntu Server VM on it) = `192.168.0.203`.
- Built and tracked under its own separate, authoritative plan: `C:\Users\micha\lab\ai-node\hybrid_ai_node_build_plan.md`, progress logged in that folder's `build_log.md`. **Do not duplicate that plan's steps here** — Phase E defers to it entirely for the Ollama install.
- Role in this lab: **local inference only.** Runs Ollama on the RTX 3070. It is the only box that can do GPU inference; the R710 has no usable GPU, so all local AI work physically lives here.
- Not part of the attack range. Stays reachable to the Infra VLAN so the triage layer and Open WebUI can call it.

### Network — current state
- pfSense CE 2.8.1 running (VM 102), all three VLAN interfaces created and reachable — **Phase A complete**: `10.10.10.1/24` MGMT, `10.10.20.1/24` INFRA, `10.10.30.1/24` RANGE.
- Cisco switch factory reset, VLANs 10/20/30 created, `Gi1/0/3` trunk to pfSense, `Gi1/0/48` management access port. Console via USB-serial + PuTTY (COM14, 9600/8/1/None/None).
- iDRAC recovered and secured: `https://192.168.0.100`.

---

## Target AI Architecture (local-first hybrid)

The decided shape, folded into the phases below:

- **Local-first.** Ollama on `pve-ai` (RTX 3070) is the default brain for everything — routine SOC triage and Michael's own daily AI use. Costs electricity, keeps data on-network, unlimited.
- **Cloud supplements, on a threshold.** Escalate to **Claude Code** (`claude -p` headless — draws from the existing subscription, **no per-token API billing**) or **Grok** (its own API, pay-as-you-go) only when a harder/ambiguous case justifies it. Michael sets the escalation threshold.
- **n8n is the router.** n8n makes the local-vs-cloud decision for both SOC triage and personal use, and hosts general automation. It is a non-human decision-maker → governed in `agent-registry.md` as `triage-router-01`.
- **Open WebUI is the personal chat interface** — points at Ollama by default, with Claude/Grok selectable. Hosted on `pve01` (service host), reached from any browser.
- **Two engines, two lanes:** n8n = AI orchestration + general automation; Shuffle = SOC incident response (Phase F). Kept separate deliberately. **Never build both automation engines in the same stretch** — Shuffle first (Phase F), n8n later with the AI stack.

### Where things run
- **`pve-ai`:** Ollama (GPU inference). Nothing the user clicks lives here.
- **`pve01` (service host):** pfSense, Wazuh, Suricata, Shuffle, Open WebUI, n8n, and the Range VMs. All service UIs live here, reached from the browser.
- **Browser / phone (MGMT VLAN):** where Michael interacts with everything.

### Clustering (decided — sequence TBD, low priority)
Cluster `pve01` + `pve-ai` into one Proxmox web UI so both nodes manage from a single pane. Overhead is negligible (corosync + config DB, a few hundred MB RAM — does not touch inference performance). **The real issue is two-node quorum:** if either node goes offline the survivor drops to read-only. Fix with a **QDevice** tiebreaker hosted on the QNAP (Container Station) — the clean path. Manual `pvecm expected 1` is the fallback when a node is intentionally down. Treat clustering as an optional convenience layer; it does not gate any SOC phase.

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
 lab port)  host: Wazuh,  Win11-victim)  receive-only)
            Suricata,                        │
            Shuffle,               USB-Ethernet adapter
            Open WebUI, n8n;       (passed through to Wazuh host,
            + pve-ai inference)     Suricata listens here — no IP)
```

**Two layers of visibility (as in a real SOC):**
- **Host-based** — Sysmon on the Win11 victim → Wazuh agent → Wazuh (crosses the Range→Infra fence on TCP 1514 only).
- **Network-based** — SPAN port mirrors Range VLAN traffic → Suricata (passive, no IP) → alerts into Wazuh. Correlating both is the high-value detection-engineering skill this lab demonstrates.

### VLAN plan
| VLAN | Members | Access rules |
|---|---|---|
| **Management (10)** | Your PC (lab port) | Can reach all VLANs for admin (Proxmox UI, Wazuh/Shuffle/n8n/Open WebUI dashboards, pfSense UI, iDRAC) |
| **Infra (20)** | Ubuntu SOC host, `pve-ai` | Receives Range logs on Wazuh's port only. **Needs a defined outbound-internet rule** (Docker/apt/`suricata-update`, Ollama model pulls, and Shuffle enrichment to VirusTotal/AbuseIPDB) — see Open Decisions. |
| **Range (30)** | Kali, Win11-LTSC-victim | **Cannot** reach home network, internet, or Management. Only allowed path: outbound to Wazuh's log-ingest port on Infra. |

**Critical pfSense rule:** Range VLAN → default deny all, single explicit allow to Infra's Wazuh port. Verify before any Atomic Red Team simulation.

---

## Build Phases

### Phase A — Network Rebuild ✅ COMPLETE (2026-07-25)
VLANs, trunk, management access port, pfSense VM + VLAN interfaces all built and acceptance-checked. Snapshot `pfsense-clean-install` taken. See `build_log.md`.

### Phase A.5 — Isolation Rule
1. Write (do not yet apply) a pfSense rule: Range → deny all, except one explicit allow to Infra's Wazuh ingest port.
2. Review the rule text before applying — confirm it doesn't block the interface pfSense is managed from.
3. Apply; acceptance-check from a Range VM (no internet, no home net, no MGMT; Wazuh port retested end of Phase B).

### Phase B — Docker/Git/Wazuh Substrate
1. ✅ Build Ubuntu Server VM on `pve01`, Infra VLAN. **8GB RAM / 4 cores / 100GB+ disk.** Enable OpenSSH. *(2026-07-31 — `wazuh-host`, `10.10.20.100`, SSH confirmed. INFRA20 outbound rules written/tested along the way.)*
2. ✅ Host prep: `sudo sysctl -w vm.max_map_count=262144`, persist in `/etc/sysctl.conf`. *(2026-08-03)*
3. ✅ Install Docker + Compose via the apt repo method (**not** `curl | sh`); verify package names live. *(2026-08-03 — Docker CE 29.7.1, verified with `hello-world`, non-root usage enabled.)*
4. ✅ Install Git, create/clone the repo. Pre-push secret check. *(2026-08-03 — cloned to `~/soc-lab` via a dedicated SSH deploy key. INFRA20 needed a 5th rule, port 22, discovered along the way. Pre-push secret check still to confirm on this box's own future commits.)*
5. ⏳ Deploy Wazuh (manager + indexer + dashboard) via Compose; check the current tag against `documentation.wazuh.com`. Change the default password. **Mount rules host-side** (`./config/rules/local_rules.xml`). *(Not started.)*
6. Move Win11-LTSC-victim to Range VLAN (check Hardware tab), install Sysmon + Wazuh agent pointed at the Sysmon channel.
7. Move Kali to Range VLAN (check Hardware tab).
8. Re-test the isolation rule: victim reaches Wazuh on 1514, still no internet.
9. **Acceptance check:** dashboard shows the victim active with Sysmon events.

### Phase C — Detection Engineering
1. Snapshot Win11-LTSC-victim (`pre-atomic-clean`).
2. Install Atomic Red Team **on the victim** (it runs on the box you want telemetry from). Add the `C:\AtomicRedTeam` Defender exclusion — safe only because the VM is fenced (Phase A.5).
3. Run a first technique (`T1059.001`) locally, confirm Wazuh alerts.
4. Write a first custom detection rule. **Michael writes the final rule**; Claude drafts a reference only. Rules live host-side in `config/rules/local_rules.xml`. IDs start at 100000.
5. Commit rules/config-as-code.
6. **Acceptance check:** a committed rule of Michael's authorship fires on a re-run.

### Phase C.5 — Network Visibility (SPAN + Suricata)
**Prereq:** verify the USB-Ethernet adapter (`lsusb`, `ip link show`, passthrough test, throughput). Decision gate: reliable adapter → use it; flaky → fall back to NIC4.
1. Configure a SPAN session on the switch: source = Range VLAN, destination = monitor port (receive-only).
2. Physically connect the SPAN destination to the adapter/NIC4.
3. Pass it through to the **Wazuh/Infra host** (not Kali). Monitoring interface gets **no IP**: `ip link set <iface> up` + `promisc on`; confirm no inet.
4. Install Suricata bound to the monitoring interface; integrate `eve.json` into Wazuh.
5. **Acceptance check:** a network attack from Kali (e.g. `nmap -sS`) shows up in Wazuh from Suricata *and* has a host-based alert — two layers, one story.

### Phase D — AI Triage Layer + Agent-Identity Governance
Subscription-covered Claude Code / local model — no separate API billing for Claude Code.
1. **Give the triage agent its own identity, not yours** — dedicated Wazuh API user, not admin, not Michael's session.
2. **Scope it least-privilege** — `alerts:read`, `agent:read`; no delete, no config, nothing outside the Infra alert pipeline.
3. **Separate its audit trail** — every action logs `agent=wazuh-triage-01`; Michael's review logs `user=michael`.
4. **Create/maintain `agent-registry.md`** (already scaffolded) — the living record of every AI agent.
5. Build the triage service on the Infra host: pull new alerts → model → verdict suggestion, under the agent's own credential.
6. Human reviews/confirms/overrides, logged separately.
7. **Acceptance check:** an alert flows detection → Wazuh → agent verdict → reviewed verdict, both logged distinguishably; registry matches reality.

### Phase E — Local AI Model Integration + n8n Routing
Defers to `C:\Users\micha\lab\ai-node\hybrid_ai_node_build_plan.md` (its Phase 6) for the Ollama install itself. This phase adds the **integration + routing**.
1. Confirm `ai-vm` (`192.168.0.203`) reachable from the Infra host and Ollama's API responding (`curl http://192.168.0.203:11434/api/tags`). Requires the Infra→ai-vm route/rule.
2. Stand up **Open WebUI** (service container on `pve01`) pointed at Ollama; local models as default, Claude/Grok selectable.
3. Stand up **n8n** (service container on `pve01`) as the router: routine/low-severity → local Ollama; ambiguous/high-severity → escalate to `claude -p` (subscription) or Grok. Michael writes the routing threshold.
4. **Register the router in `agent-registry.md`** as `triage-router-01` — a second non-human identity making autonomous escalate/local decisions. The routing logic is governed regardless of which engine hosts it.
5. **Acceptance check:** one full triage cycle completes on the local model with no external call; a second escalates correctly; registry entry accurate.

### Phase F — SOAR (Shuffle) — NEW
**Why:** SOAR (Security Orchestration, Automation, and Response) is directly named on the SOC job listings Michael is targeting. Shuffle is the community-standard open-source SOAR engine and pairs natively with Wazuh. Slots in after Phase C, once real detections are firing that are worth automating a response to.

**Lane discipline:** Shuffle = SOC incident response. n8n = AI orchestration + general automation. Do not blur them; do not build Shuffle and n8n in the same stretch.

1. Build a dedicated **Shuffle VM** on `pve01`, Infra VLAN (own VM — do not co-locate with Wazuh; both are memory-hungry). Docker deploy.
2. Wire Wazuh → Shuffle: add a `<integration>` webhook block in Wazuh so alerts POST to a Shuffle workflow.
3. Build a first playbook (Michael writes the final logic; Claude drafts a reference): alert in → parse IOCs → **light enrichment** (VirusTotal/AbuseIPDB via HTTP) → notify. Enrichment-only needs the Infra outbound rule (below), nothing inbound.
4. **(Optional, deliberate)** Active-response step (e.g. block an IP via pfSense). This opens a new path *into* Range/pfSense — a firewall decision written and reviewed under the two-tier rule. Do **not** build auto-response until explicitly decided.
5. Commit the playbook/config-as-code.
6. **Acceptance check:** a real detection fires → Shuffle receives it → enriches → produces a case/notification end to end. Traditional manual alert-to-case is 35–60 min; the automated path is seconds — that before/after is the interview artifact.

### Phase G — Case Management (TheHive + Cortex) — DEFERRED
**Not started. Added later as one matched-pair unit once everything else is stable.** TheHive and Cortex are both from StrangeBee and are designed to pair.
- **TheHive** = case management (tickets, timeline, tasks, observables, status). The analyst's desk. No overlap with anything else in the stack.
- **Cortex** = enrichment/analysis engine (runs analyzers against IOCs at scale). Overlaps with Shuffle's light enrichment; Cortex is the systematic version.
- **Why deferred:** TheHive does not store its own data — it requires **Cassandra** (case DB) + **Elasticsearch** (search/index) as backends, all three healthy and version-matched simultaneously. Running its Elasticsearch alongside Wazuh's own indexer is the specific coexistence wrinkle. Realistic cost with friction: ~4–8 sessions (a couple of weeks at this pace), most of it on version-matching, startup-order, and memory issues — not conceptual difficulty.
- **Sequence:** attempt only after Phases F and E are solid, so the dependency-hell troubleshooting happens against a calm backdrop. TheHive wants its own dedicated VM.
- Deferring costs the polished case-management layer and the "full autonomous SOC" showpiece — but detection engineering, SOAR automation, AI triage, and agent governance (the harder, more demonstrable skills) all stand without it. `investigations/` writeups in Git serve as case documentation in the meantime.

---

## Standing Rules (carried over, apply throughout)
- Ask approval before creating any file, document, image, or phase not already scoped here
- Warn before analyzing screenshots
- Flag and check in before pursuing tangents
- One network change at a time, tested before the next
- Console/recovery access confirmed before any change that could lock out management
- Core security logic (firewall rules, VLAN/trunk config, detection rules, Shuffle playbooks, agent scoping) — Claude drafts a reference, **Michael writes the final**
- Explain-it-back checkpoint after each phase, including the new F and (eventually) G
