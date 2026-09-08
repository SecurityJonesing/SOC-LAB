# LAB-BLUEPRINT.md
### Home SOC Lab, what I'm building, and in what order

**Purpose:** My shared map with Claude. What gets built, in what sequence, and how I know each phase actually worked. I run every command myself; Claude guides one step at a time and reads back the output. `build_log.md` is a short index into `Build Logs/`, which carries the detailed, dated narrative of every step I've actually taken, the `.docx` workbooks that used to accompany this file were retired (2026-08-03) in favor of the `.md` files being the sole source of truth.

**Working principle:** Build incrementally. One change, one test, then the next change. Don't stack multiple untested changes (network config in particular, this environment has a documented history of lockouts from big-bang changes).

**Scope status (2026-08-06):** This document reflects the full, locked scope of the build, start to finish, Phases 1 through 12, including Phase 7 (AD/kill-chain expansion, now including a second workstation and a fully realistic AD environment) and Phase 8 (IAM/Entra ID). Everything in it is committed, planned work, assembled across an extended planning conversation. New ideas raised mid-build get logged as next-steps in the relevant `Write-Up ....md` file (embedded in that phase's step folder under `Build Logs/`) or as a note in `PROJECT-INSTRUCTIONS.md`, they don't get folded into active scope without a deliberate decision to revisit and re-lock. See "Explicitly out of scope" near the end for everything already evaluated and deliberately excluded.

---

## Confirmed Environment (as of build start)

### Hardware
| Node | Spec | Role |
|---|---|---|
| `pve01` | Dell PowerEdge R710, Xeon X5675, 64GB RAM, RAID 10, 4x physical NICs | Primary Proxmox host + **always-on service host**, victims, attacker, pfSense, Wazuh, Shuffle, n8n, service UIs |
| `pve-ai` | i9-10900KF (10c/20t, **no integrated graphics**), ASUS TUF Gaming Z590-Plus WiFi, RTX 3070 (8GB VRAM), ~31GB RAM | **Local AI inference node**, GPU muscle only (Ollama). Proxmox VE, static `192.168.0.202`. Built and fully documented 2026-07-12/13, see below. |
| Cisco Catalyst switch | WS-C2960X-48FPS-L, IOS 15.2(7)E9 | Factory reset complete and confirmed clean; carries VLAN trunk + SPAN monitor port |

### Existing VMs on `pve01`
- **Kali** — attack box (runs real network attacks: nmap, Responder, NetExec, etc.). **Moved from flat `vmbr1` to RANGE30 on 2026-08-18, isolation confirmed live via three pings** (no internet, no home/MGMT reach, no ICMP to INFRA20), the same isolation model already proven for Win11-LTSC-Victim. This closed the first of three infra-hardening sub-steps in Phase 6.
- **Win11-LTSC-victim** — Windows victim. Moved to RANGE30 in Phase 3 (2026-08-04), isolation proven live, Sysmon + Wazuh agent reporting (agent ID `001`). Domain-joins to `soclab.internal` in Phase 7; serves as the first lateral-movement hop.
- **`wazuh-host` / `ubuntu-soc-host`** — Docker/Wazuh substrate on INFRA20, built in Phase 3, complete. Suricata 8.0.6 added in Phase 5, integrated with Wazuh via `eve.json`.

### Planned new VMs and objects (Phase 7 / 8, not yet built)
| Node / object | Spec | VLAN | Role |
|---|---|---|---|
| `dc01` | Windows Server 2022, Core install, 2 vCPU/4–8GB | **RANGE30** | Domain Controller, new forest, domain **`soclab.internal`**. Treated as an attack target, consistent with everything else on RANGE30, not as trusted infrastructure. |
| `win11-ws02` | Windows 11, real live VM | **RANGE30** | Second, previously-uncompromised workstation, the actual lateral-movement target. Domain-joined. Without a second real host, "lateral movement" would only be reused credentials on the same box, not a genuine pivot. |
| `linux-victim` | Ubuntu Server, minimal | **RANGE30** | Deliberate sudo misconfiguration, `auditd` + Wazuh agent, hosts the vulnerable web app (DVWA or Juice Shop) that provides Initial Access |
| `entra-connect-01` | Windows Server, small | **INFRA20** | Runs Microsoft Entra Connect, bridges `dc01` to the Entra ID tenant. Needs outbound internet and one narrow inbound path to `dc01`, see VLAN plan below. |
| 3–4 "phantom" computer objects | AD computer accounts only, `New-ADComputer`, no live VM behind them | N/A (directory objects only) | Populate the OU structure and give BloodHound/enumeration tooling a realistic-sized environment to map, without the RAM cost of building 3–4 more full VMs. (Five additional live Windows VMs was the originally floated number, judged too heavy against `pve01`'s 64GB total alongside Shuffle and Docker/Wazuh; this mix gets the realism without the resource risk.) |

### VLAN / Network Plan

| VLAN | Purpose | Access |
|---|---|---|
| **Management (10)** | Management PC | Full access to all VLANs for admin |
| **Infra (20)** | `wazuh-host`, `pve-ai`, `entra-connect-01`, Shuffle | Receives Range logs. Needs a defined outbound-internet rule (Docker/apt/`suricata-update`, Ollama model pulls, Shuffle enrichment to VirusTotal/AbuseIPDB, Entra Connect's sync traffic to Microsoft). **A single narrow Pass rule from `entra-connect-01` to `dc01` on RANGE30, for Entra Connect's sync traffic only**, see the callout below. |
| **Range (30)** | Kali, Win11-LTSC-victim, `win11-ws02`, `dc01`, `linux-victim` | **Cannot** reach home network, internet, or Management. Only allowed outbound path: to Wazuh's log-ingest/enrollment ports on Infra. **One narrow, reviewed exception, inbound from Infra:** the Entra Connect sync rule below. |

**Critical pfSense rule (unchanged principle):** Range VLAN → default deny all, explicit allows only. Verify before any Atomic Red Team or Phase 7 simulation.

**New pattern to expect, INFRA20 → RANGE30 (Phase 8):** every rule written in this build so far has gone one direction, RANGE30 → INFRA20 (Wazuh ingest/enrollment). Entra Connect breaks that pattern: it runs on INFRA20 but must reach `dc01` on RANGE30 to sync. This is **the first rule ever written in the opposite direction**, and it's the one deliberate exception to the "Range cannot be reached from anywhere" isolation model proven in Phase 2/3. I'll write this narrowly, single source IP (`entra-connect-01`), single destination IP (`dc01`), AD sync ports only (see Phase 7's AD port list), not a general INFRA20↔RANGE30 opening. I write the final rule, reviewed with extra scrutiny given it's the first of its kind in this build.

---

## Build Phases

Split into three files under `Blueprint/` on 2026-09-07 to keep this index file under the ~16,000-character tool-read limit (mirrors the same split already applied to `build_log.md` → `Build Logs/`). Each file carries the full per-phase detail: purpose, steps, acceptance checks, and status.

| File | Covers | Status |
|---|---|---|
| [`Blueprint/Phases 1-6.md`](./Blueprint/Phases%201-6.md) | Network Rebuild, Isolation Rule, Wazuh Substrate, Detection Engineering, Network Visibility, Attack Surface | 1-5 complete, 6 in progress |
| [`Blueprint/Phase 7 AD Expansion.md`](./Blueprint/Phase%207%20AD%20Expansion.md) | Full AD/DC build, kill chain, LAPS fix, writeup — the single largest phase | Not started |
| [`Blueprint/Phases 8-12.md`](./Blueprint/Phases%208-12.md) | Hybrid Identity, AI Triage Layer, Local AI and Routing, SOAR, Case Management | 8-9, 11-12 not started; 10 step 1 complete |

## Time estimate, locked scope

| Block | Manual (hrs) | Automated (Claude Code) |
|---|---|---|
| Phase 4 | 5–7 (complete) | — |
| SSH key-only hardening, `wazuh-host` | 0.5–1 (complete) | — |
| Phase 5 (SPAN, Suricata, east-west visibility gap, hypervisor `tc` mirroring) | complete, persistence step outstanding | — |
| Phase 6 (infra hardening: Kali isolation complete, pfSense log forwarding, WireGuard VPN) | ~2–4 remaining | — |
| Phase 7 (full AD build design + misconfigurations, expanded Kali tooling, full kill chain incl. Discovery/BloodHound workflow, LAPS before/after, stitching + writeup) | 36–48 | ~5.5–9 |
| Phase 8 (Entra ID Free + Entra Connect + Exchange Online + P1 trial + AADInternals/ROADtools) | 11–19 | ~1.5–2.5 |
| Phase 9 | 5.5–9 | — |
| Phase 10 | 4.5–6.5 | — |
| Phase 11 | 4.5–8.5 | — |
| **Total remaining (Phase 6 through 11; Phases 1/2/3/4 complete; Phase 5 nearly complete)** | **~69–103 hrs** | **~7–11.5 hrs** |

**What's automated, final version (2026-08-06, revised):** `dc01`'s VM creation, base Windows Server install, and AD DS role install only, **not** `Install-ADDSForest` itself, which is where the real decisions live (forest/domain functional level, DNS strategy, DSRM password) and stays manual, along with Kerberos Policy configuration, the PDC Emulator time source, and post-build verification. `win11-ws02`'s full build (VM/Windows/Sysmon/Wazuh agent), domain-joining both workstations, AD account creation/OU placement/group membership assignment (not group creation), the AD port-list firewall rules on RANGE30, Entra Connect's VM creation and software install (not sync account scoping or the firewall rule), and Exchange Online mailbox creation (not Attack Simulation Training design) are all Claude Code-executed. Everything with real security or design judgment, the OU/tiered-model design, group creation, GPOs, all six deliberate misconfigurations, the WireGuard VPN configuration, the INFRA20→RANGE30 Entra Connect rule, Conditional Access/MFA testing, and AADInternals/ROADtools, stays fully manual. See "Execution model exceptions" in `PROJECT-INSTRUCTIONS.md` for the full reasoning.

At ~15–17 hrs/week (baseline pace inferred from `build_log.md`, plus a 5 hr/week weekday addition): **roughly 4.1–6.9 weeks**, realistically **4–7 weeks**, for the remaining scope.

**Current priority note (2026-08-24):** all Phase 6+ work above is paused pending an active job search, see `PROJECT-INSTRUCTIONS.md`'s "Current priority" section. Nothing in this phase-by-phase plan is cancelled; the estimates above remain the plan for whenever work resumes.

---

## Explicitly out of scope (documented next-steps, not live plan)

- **MITRE Caldera**, post-compromise attack-chain orchestration. Would work against the "I executed this myself, end to end" narrative for a modest time saving. Revisit only with runway to spare after the locked plan is complete.
- **Microsoft Entra ID P2**, Identity Protection's risk-based sign-in scoring, Privileged Identity Management, Access Reviews. More architect-tier than day-to-day analyst work; the Free tier plus a time-boxed P1 trial covers the target skill set.
- **On-prem Exchange Server**, Exchange Online only. On-prem Exchange's setup/troubleshooting risk outweighed its actual relevance to the target roles.
- **Standalone GoPhish + mail relay**, Exchange Online's own Defender for Office 365 Attack Simulation Training is used instead, specifically to avoid ambiguity around automated abuse-detection systems flagging a homebuilt phishing campaign.
- **Full interactive host-level compromise of `dc01`**, the DCSync misconfiguration provides a genuine, real-world path to domain-wide credential material without needing a shell on the DC itself. Documented as a deliberate scope boundary and a future writeup next-step, not a shortfall.

---

## Clustering, decided, low priority

Cluster `pve01` + `pve-ai` for a single Proxmox pane. Overhead negligible; the real issue is two-node quorum (survivor drops to read-only if one node is down). Fix with a **QDevice** on the QNAP (Container Station). Fallback: `pvecm expected 1`. Doesn't gate any SOC phase, convenience layer only.

---

## Source of truth

The `.docx` workbooks that used to accompany this build have been retired (2026-08-03), they duplicated what's already in the `.md` files below and went stale independently. These four files are now the entire documentation set:

- **`LAB-BLUEPRINT.md`** (this file), what I'm building and in what order, including the locked Phase 7 (AD/kill-chain expansion) and Phase 8 (IAM/Entra ID), plus Phase 11 SOAR and the deferred Phase 12. The full per-phase "Build Phases" detail lives in `Blueprint/` (split 2026-09-07 into `Phases 1-6.md`, `Phase 7 AD Expansion.md`, and `Phases 8-12.md` to keep each file under the ~16,000-character tool-read limit)
- **`build_log.md`**, a short index into `Build Logs/`, which carries the running, append-only record organized by phase, one numbered folder per phase, split into Step subfolders where a phase has more than one step or is still in progress; a complete phase with a single entry gets one flat file directly in the phase folder instead
- **`agent-registry.md`**, every AI agent's scope, owner, lifecycle (`wazuh-triage-01`, `triage-router-01`)
- **`README.md`**, repo-facing overview, phase status table, diagram link

The former `ai-node` project's own `build_log.md` and `hybrid_ai_node_build_plan.md` are retained as historical source material but are no longer separately maintained, everything current lives in the four files above.

Keep this file short and current. Update it whenever a phase status, scope decision, or environment fact changes.

---

## Phase numbering, renamed 2026-09-06

Phases used to be lettered (A, A.5, B, C, C.5, C.6, C.7, D, E, F, G). They are now whole numbers, 1 through 12, permanent, never renumbered or reused again. The old C.6 letter covered two very different bodies of work under one label, infra hardening and the full AD build, so it was split into two phases (6 and 7) during the rename. Everything else is a straight one-to-one swap.

| New | Old | Name |
|---|---|---|
| 1 | A | Network Rebuild |
| 2 | A.5 | Isolation Rule |
| 3 | B | Wazuh Substrate |
| 4 | C | Detection Engineering |
| 5 | C.5 | Network Visibility |
| 6 | C.6, sub-step 1 (infra hardening: pfSense log forwarding, Kali isolation, WireGuard VPN) | Attack Surface |
| 7 | C.6, sub-steps 2-11 (AD/DC build through kill chain, LAPS, writeup) | AD Expansion |
| 8 | C.7 | Hybrid Identity |
| 9 | D | AI Triage Layer |
| 10 | E | Local AI and Routing |
| 11 | F | SOAR |
| 12 | G | Case Management |

Any older note, commit message, or chat title using a letter refers to the phase in this table, not a different one.
