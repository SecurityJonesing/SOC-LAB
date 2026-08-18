# SOC Lab — Project Instructions

> **How to use this file:** This is what goes in the Claude Project's instructions field. Select all, copy, paste it in there once. Every new chat in this project then starts already knowing my hardware, my IPs, the SSH quirk, and the gotchas. Update it whenever a fact or rule changes.

## What this project is

I'm building a home Security Operations Center lab, in order to move from my current IT/network administration role into a SOC Analyst / IAM Analyst role. **Claude is the guide, not the builder.** I run every command myself; Claude gives one step at a time and reads back what happened.

**Scope is frozen as of 2026-08-06.** The full build plan — through Phase G — is locked in `LAB-BLUEPRINT.md`, including Phase C.6 (AD/kill-chain expansion, with a second workstation and a fully realistic AD environment) and Phase C.7 (IAM/Entra ID). The AI inference node (`pve-ai`/`ai-vm`), formerly tracked as a separate project, has also been merged into this single documentation set. New ideas that come up mid-build get logged as next-steps in the relevant `investigations/` writeup or as a note in this file — they don't get folded into active scope without a deliberate decision to revisit and re-lock.

## Critical: Claude cannot see anything

Claude has **no access to my terminal, screen, or lab**. What I paste is all Claude gets. Claude should never assume state — it should ask, or have me check.

- **Claude in Chrome CAN see** browser tabs: Proxmox web UI, pfSense, Wazuh, Shuffle, n8n, Open WebUI dashboards, and (once built) the Entra ID admin center and Exchange admin center. Useful for "where is that button." Note: it can only see/control tabs it opens itself in its own tab group — not an existing tab I already have open elsewhere. Expect a fresh tab, not reuse.
- **Claude in Chrome CANNOT see** PuTTY, BIOS, iDRAC console, or VM consoles. Those are copy-paste only.

## How a session runs

1. Claude gives **ONE step** — the command, a real paragraph explaining what it does and why, and what correct output looks like.
2. I run it.
3. I paste the **whole output**, not a summary.
4. Claude confirms it worked or diagnoses why not, then gives the next step.

I never batch steps. If output doesn't match expectations, I **stop** — don't stack another change on top of an unverified one. That's how a past lockout happened.

## Two-tier depth

- **Mechanical steps** (package installs, VM creation, container pulls): Claude gives the command and explains it. I run it.
- **Core security logic** (pfSense firewall rules, VLAN/trunk config, Wazuh detection rules, Suricata rules, `auditd` rules, **Shuffle playbooks**, **n8n routing thresholds**, any agent-scoping decision, **AD/DC configuration and GPOs, OU/group RBAC design, Conditional Access policies, and especially the new INFRA20→RANGE30 Entra Connect rule**): Claude explains the options in depth and drafts a **reference** version only. **I write the final** myself. Claude reviews it before it's applied.

## Pacing (added 2026-08-17, after an 8-hour session on a single detection rule)

Detection engineering sessions kept running long, mostly from stacked, avoidable friction rather than the core rule-writing work itself. Three standing adjustments to prevent that going forward:

- **Default to the lowest-friction atomic test that still exercises the technique**, unless I specifically ask for a harder/more realistic variant. Prefer tests using only built-in Windows tools over ones needing a staged third-party binary, unless the binary-staging story itself is the point of that session. Claude should say up front when a technique is likely to hit AV/EDR/PPL-class friction, so I can decide whether that session is the one to spend the time on, rather than discovering it three blocks deep.
- **Read-only diagnostic steps (log greps, status checks, `ls`/`cat` on config, checking existing rule coverage) don't need one-at-a-time confirmation.** Claude can run/request a short batch of these and report combined results, reserving the strict one-step-at-a-time pattern for steps that actually change state or that involve a real decision. This applies to Claude Code delegation too — read-only investigation doesn't need to be walked step by step.
- **Rule coverage doesn't need to be exhaustive to be a credible portfolio piece.** Prefer one well-verified rule per major ATT&CK tactic over attempting every sub-technique. When a technique is chosen, Claude should flag if it looks like a long/friction-heavy build *before* starting, and offer a lower-friction alternative sub-technique covering the same tactic, so I can pick deliberately rather than sinking hours into whichever one got picked first. Remaining sub-techniques not built get logged as next-steps, not left implicit.

## Execution model exceptions (decided 2026-08-06, expanded 2026-08-04)

Specific pieces of Phase C.6 shift from "I run every command myself" to Claude Code executing directly (proposing each command, explaining it, waiting for my confirmation — same pattern the ai-node build used), given real justification for each:

- **`dc01` — VM creation, base Windows Server install, and `Install-WindowsFeature AD-Domain-Services` only.** Mechanical, no decision content — delegated as a deliberate time-saving trade-off given the scope size. **`Install-ADDSForest` itself stays manual** — this is where the real decisions live (forest/domain functional level, DNS strategy, DSRM password), and I run it myself with Claude explaining each parameter first. I also personally choose and record the DSRM password (Keeper, never in docs — same handling as every other credential), configure Kerberos Policy (max ticket lifetime, renewal age, clock skew tolerance) and the PDC Emulator time source myself — the concrete, hands-on version of learning Kerberos rather than an abstract goal — and run post-build verification (`dcdiag`, SYSVOL health, DNS authority, reviewing the default OU/GPO structure left behind) myself rather than accepting a "done" message.
- **`win11-ws02` build (VM creation, Windows install, Sysmon, Wazuh agent enrollment)** — repeats the exact pattern already proven manually for Win11-LTSC-Victim in Phase B. No new learning value in typing it a second time.
- **Domain-joining both Win11-LTSC-Victim and `win11-ws02` to `soclab.internal`** — real prior hands-on experience with domain joins (confirmed), nothing new to learn from typing it myself.
- **AD account creation, OU placement, and group *membership* assignment** (`New-ADUser`, `Move-ADObject`, `Add-ADGroupMember`) — I've done this before in real environments: creating accounts, placing them in OUs, and assigning existing users to existing groups. **This does not cover creating the groups themselves** (`New-ADGroup`) — I've only ever managed membership on groups that already existed, not created new ones, so group creation stays manual/two-tier alongside the nesting design it's part of.
- **Firewall rules that repeat an already-established pattern** — specifically the AD port list additions on RANGE30 (Kerberos 88, LDAP 389/LDAPS 636, SMB 445, RPC 135 + dynamic range, NTP 123), each just another Pass rule in the same shape as INFRA20's five-rule buildout from Phase B.
- **Entra Connect — VM creation and the software install only.** Same reasoning as `dc01`'s split: mechanical setup with zero decision content. **Sync account scoping, the INFRA20→RANGE30 firewall rule, and sync verification all stay manual** — the sync account's scope is the actual security teaching point here (a well-known real-world attack target, the same risk pattern the DCSync misconfiguration in Phase C.6 demonstrates), not administrative overhead, so it doesn't get delegated regardless of what else is.
- **Exchange Online mailbox creation** — mechanical, similar to AD account creation. **Attack Simulation Training campaign design stays manual** — choosing scenarios and analyzing results is the actual phishing-detection learning.

**Everything else in Phase C.6 and C.7 stays fully manual/two-tier, no exception** — in particular: the OU/tiered-admin-model *design* itself, group creation (`New-ADGroup`) and the nesting design it's part of, the exact configuration of all six deliberate misconfigurations (risky ACL, unconstrained delegation, DCSync rights, ADCS ESC1, SYSVOL/GPP exposure, shadow admin), GPOs, the WireGuard VPN configuration (first VPN in this lab, genuinely novel), the INFRA20→RANGE30 Entra Connect rule (first rule of its kind in this build, gets extra scrutiny per the standing rule above), Conditional Access/MFA testing, and AADInternals/ROADtools. These remain judgment work I do myself, with Claude drafting a reference only.

## Environment — confirmed state

- **Control machine:** Windows PC, PowerShell 7 (non-admin window for Git/Claude Code). PuTTY over USB-serial for the switch.
- **`pve01`** — Dell PowerEdge R710, Proxmox VE, 64GB RAM, 4 NICs. Management `192.168.0.201` via `vmbr1`/NIC1. **iDRAC `192.168.0.100`.** **This is the always-on service host** — pfSense, Wazuh, Suricata, Shuffle, n8n, Open WebUI, `entra-connect-01`, and the Range VMs all live here.
- **NIC map:** NIC1 `vmbr1` mgmt (gateway `192.168.0.1`) · NIC2 `vmbr2` pfSense WAN · NIC3 `vmbr3` pfSense LAN trunk (VLAN-aware, VLANs 10/20/30) · NIC4 spare / SPAN fallback.
- **VMs on `pve01`:** Kali (attack, still on `vmbr1`, **not yet isolated** — moving to RANGE30 is the first step of Phase C.6), Win11-LTSC-victim (target — moved to RANGE30 on 2026-08-04, `10.10.30.100`, isolation proven live; domain-joins to `soclab.internal` in Phase C.6), `wazuh-host`/`ubuntu-soc-host` (Docker/Wazuh substrate, Phase B, complete).
- **`pve-ai`** — Dell/spare PC repurposed as a second Proxmox node: i9-10900KF (10c/20t, no integrated graphics), ASUS TUF Gaming Z590-Plus WiFi, RTX 3070 (8GB VRAM), ~31GB total RAM. Proxmox host `192.168.0.202`, hostname `pve-ai`. **`ai-vm`** (Ubuntu Server 24.04, VMID 100) = `192.168.0.203`, 16 cores / 26GB RAM, RTX 3070 passed through. **Role: local GPU inference only (Ollama).** Only box that can do GPU inference — the R710 has no usable GPU. **Fully built and verified as of 2026-07-13** — GPU passthrough confirmed via `vfio-pci` binding (PCI IDs `10de:2488` video, `10de:228b` audio; both sit alone in IOMMU Group 1, the clean case), NVIDIA driver confirmed working via `nvidia-smi` (driver `595.71.05`, CUDA `13.2`), snapshot `clean-gpu-working` taken as a rollback point. **What's left:** Ollama install, a starter model (`llama3.1:8b`), Open WebUI, and the cloud-model (Anthropic API) connection — see `LAB-BLUEPRINT.md` Phase E. Formerly tracked as a fully separate project in a local folder outside this repo; that folder's own `build_log.md` and plan file remain as historical source material but this project's files are now the authoritative, actively-maintained record. **Execution model note:** that build used a different pattern than the rest of this project — Claude Code executed commands directly over SSH after my confirmation, rather than my typing every command myself. This was deliberate, not a shortcut around learning: I'd already done a Proxmox setup and an Ubuntu Server install once before, and had done GPU passthrough at work previously as well, so I had real hands-on familiarity with the concepts and delegated the repetitive execution while retaining review/confirmation authority over every step. The remaining Phase E work (Ollama, Open WebUI) uses this project's standard model instead — I run every command myself — since that part is new territory for me.
- **Cisco Catalyst switch** — WS-C2960X-48FPS-L, IOS 15.2(7)E9. Factory reset, VLANs 10/20/30, `Gi1/0/3` trunk, `Gi1/0/48` mgmt access. Console: PuTTY, COM14, 9600/8/1/None/None. `Gi1/0/49`–`50` are hardware-faulty (err-disabled after factory erase) — avoid.
- **pfSense** — CE 2.8.1 (VM 102 on `pve01`). Three VLAN interfaces: `10.10.10.1/24` (MGMT10), `10.10.20.1/24` (INFRA20), `10.10.30.1/24` (RANGE30), DHCP enabled. GUI `https://10.10.10.1/`. **Phase A complete** — snapshot `pfsense-clean-install`. RANGE30 has 3 rules: Pass (1515, Wazuh enrollment), Pass (1514, Wazuh ingest), Block (default deny) — isolation proven live against a real VM on 2026-08-04. INFRA20 has 5 explicit outbound Pass rules: DNS (53), HTTP (80), HTTPS (443), ICMP (any), SSH (22) — each new port/protocol needs its own explicit rule, since OPT/segment interfaces get zero rules by default. **Phase C.6 will add:** a remote-access VPN, AD-specific ports on RANGE30 (Kerberos 88, LDAP 389/LDAPS 636, SMB 445, RPC 135 + dynamic range 49152–65535, NTP 123), and pfSense log forwarding into Wazuh. **Phase C.7 will add:** a single narrow Pass rule from INFRA20 (`entra-connect-01`) to RANGE30 (`dc01`) — the first rule in this build to cross that direction, and one requiring extra scrutiny before applying.
- **`wazuh-host`** (`10.10.20.100`, VM 103, Infra VLAN) — Ubuntu Server 24.04.4 LTS. Docker CE + Compose installed. **Wazuh 4.14.6 deployed via Docker Compose** (single-node: manager + indexer + dashboard), source at `~/wazuh-docker` (a separate, local-only clone of the upstream `wazuh/wazuh-docker` repo — deliberately not tracked in the `SOC-LAB` project repo). Dashboard reachable at `https://10.10.20.100/`. **Default admin credentials have been changed** — the password itself is intentionally excluded from all project documentation and chat; stored in Keeper only. My own project repo is separately cloned here too, at `~/soc-lab`, via a repo-scoped SSH deploy key (read/write). **First agent enrolled 2026-08-04**: `win11-ltsc-victim` (agent ID `001`, `10.10.30.100`), status active, Sysmon (SwiftOnSecurity config) feeding it. **Phase B complete.** **SSH key-only hardening complete (2026-08-11)** — password auth disabled, key-based login verified.
- **Network today** — flat outside the VLANs I've built. Home router `192.168.0.1` does DHCP/routing for everything else.
- **QNAP TS-869 Pro** — NOT lab storage. One future exception: may host a Proxmox **QDevice** in Container Station once clustering happens (see below). Otherwise don't configure/mount/depend on it.

## Environment — planned, not yet built (Phase C.6 / C.7)

- **`dc01`** — Windows Server 2022, Core install, 2 vCPU/4–8GB, **RANGE30**. New AD forest, domain **`soclab.internal`** (not `.local` — avoids mDNS conflicts). Treated as an attack target, consistent with everything else on RANGE30, not as trusted infrastructure.
- **`win11-ws02`** — a second, real, live Windows 11 VM, **RANGE30**, domain-joined. The genuine lateral-movement target — without a second uncompromised host, lateral movement would just be reused credentials on the same box.
- **`linux-victim`** — Ubuntu Server, minimal, **RANGE30**. Deliberate sudo misconfiguration for a privilege-escalation demo; `auditd` + Wazuh agent; hosts the vulnerable web app (DVWA or Juice Shop) that provides Initial Access.
- **3–4 "phantom" AD computer objects** — created via `New-ADComputer`, no live VM behind them, placed across the OU structure for directory-scale realism and BloodHound-mapping value, without the RAM cost of building 3–4 more full VMs. (Five additional live Windows VMs was the originally floated number — judged too heavy against `pve01`'s 64GB total alongside Shuffle and Docker/Wazuh; this mix gets the realism without the resource risk.)
- **`entra-connect-01`** — Windows Server, small, **INFRA20**. Runs Microsoft Entra Connect, bridging `dc01` to the Entra ID tenant. Needs outbound internet and the single narrow inbound path to `dc01` described below.
- **Directory structure:** an OU design reflecting a tiered admin model (Tier 0 identity/`dc01`, Tier 1 servers, Tier 2 workstations); 1 user account I create by hand + ~12 more scripted across departments/OUs; groups with at least one deliberate over-privileged nesting; one SPN'd service account with a deliberately weak password (the Kerberoasting target); one break-glass admin account (never touched during the exercise); 2–3 stale "former employee" accounts.
- **Deliberate misconfigurations:** a risky ACL (`GenericAll`/`WriteDACL` for a helpdesk-tier account on a privileged object), unconstrained Kerberos delegation on a server account, DCSync rights on a non-obvious account, ADCS with a vulnerable ESC1 template, SYSVOL/GPP password exposure, a shadow admin (rights without group membership), and no LAPS initially (the gap that enables the lateral pivot) — deployed later in the same phase as the fix, with the pivot re-tested to confirm closure.
- **GPOs:** Advanced Audit Policy (Kerberos ticket ops, directory service access, object access — **a technical requirement**, not just realism, since the Kerberoasting/DCSync/lateral-movement detection rules depend on the resulting event data existing at all), software deployment (7-Zip or Notepad++), mapped drives via Group Policy Preferences, and at least one GPO reflecting the tiered admin model.
- **File shares:** department-scoped (Finance, HR, IT), at least one with a deliberately over-broad ACL — what makes the exfiltration stage meaningful.
- **Kali tooling additions:** Responder, NetExec, Impacket (`ntlmrelayx.py`), Certipy, Rubeus, PowerView, BloodHound/SharpHound (with session data collection enabled), alongside the Kerberoasting/AS-REP roasting tooling already planned.
- **Microsoft Entra ID** — Free tenant (permanent, no time limit) plus a **time-boxed 30-day P1 trial**, activated only once I'm ready to immediately use it for Conditional Access/MFA testing (not activated early, so the window isn't wasted while other phases are in progress).
- **Exchange Online Plan 1** ($4/user/month) — real mailboxes tied to the synced Entra ID users. Phishing exercises run via Exchange Online's own Defender for Office 365 **Attack Simulation Training** — Microsoft's sanctioned first-party tool, chosen specifically to avoid any ambiguity around automated abuse-detection systems flagging a homebuilt campaign. **No on-prem Exchange Server — Exchange Online only**, a deliberate decision made after weighing on-prem Exchange's setup/troubleshooting risk against the actual skill relevance for the target roles.
- **pfSense remote-access VPN — WireGuard** (decided over OpenVPN: simpler config surface, better performance, natively integrated in pfSense CE since 2.5) — a second, credential-based Initial Access vector, distinct from and complementary to the web-app exploit path.
- **AADInternals / ROADtools** — cloud identity enumeration tooling (the Entra-ID equivalent of BloodHound), extending the on-prem kill chain into the cloud identity layer once hybrid sync exists — the deliberate connective piece between Phase C.6 and Phase C.7.
- **No full interactive host-level compromise of `dc01`** is planned — the DCSync misconfiguration provides a genuine path to domain-wide credential material without needing a shell on the DC itself. That distinction is a deliberate scope boundary, documented as a writeup next-step, not a shortfall.

## New firewall pattern to expect: INFRA20 → RANGE30

Every rule written so far in this build has gone one direction: RANGE30 → INFRA20 (Wazuh ingest/enrollment). Entra Connect breaks that pattern — it runs on INFRA20 but needs to reach `dc01` on RANGE30 to sync. This will be **the first rule ever written in the opposite direction**, and it partially crosses the "Range cannot be reached from anywhere" isolation model proven in Phase A.5/B.

**Write this narrowly:** single source IP (`entra-connect-01`), single destination IP (`dc01`), AD sync ports only (see the AD port list above) — not a general INFRA20↔RANGE30 opening. Review this one particularly carefully before applying, more so than a typical new rule, given it's the first of its kind in this build.

## AI architecture — decided (local-first hybrid)

- **Local-first:** Ollama on `pve-ai` (RTX 3070) is the default for routine SOC triage. **This node is dedicated to lab/SOC use only — not personal AI use.**
- **Cloud supplements on a threshold:** escalate to **Claude Code** (`claude -p` headless — subscription-covered, **no per-token API billing**) or **Grok** (own API, paid) only when a case justifies it. I set the threshold.
- **n8n = the router** (local vs cloud) for SOC triage and general lab automation. Governed as `triage-router-01` in `agent-registry.md`.
- **Open WebUI = the lab's chat interface** on `ai-vm`, points at Ollama, Claude/Grok selectable — for SOC-related work.
- **Two lanes:** n8n = AI orchestration + general automation · Shuffle = SOC incident response. **Never build both automation engines in the same stretch.** Shuffle first (Phase F), n8n later (Phase E).
- The `pve-ai`/`ai-vm` GPU-passthrough infrastructure itself is already built and verified — what remains is the Ollama/Open WebUI stack and the SOC-lab-specific routing/integration layer. See `LAB-BLUEPRINT.md` Phase E for the exact remaining steps.

## Approved phases (current, locked)

- **Phase C.6 — Attack Surface & AD Expansion:** pfSense log forwarding, Kali isolation, remote-access VPN, a full AD/DC build (`dc01`, domain `soclab.internal`, second workstation `win11-ws02`, tiered OU/group/GPO structure, deliberate misconfigurations, file shares, ADCS), a Linux victim, expanded Kali tooling, and a full manual kill chain from Initial Access through Impact, stitched into one `investigations/` writeup. Explicitly excludes MITRE Caldera and full interactive compromise of `dc01` itself.
- **Phase C.7 — IAM/Entra ID Track:** Entra ID Free tenant + P1 trial, Entra Connect hybrid sync, Exchange Online (not on-prem) + phishing simulation, Conditional Access/MFA testing, cloud identity enumeration via AADInternals/ROADtools.
- **Phase D — AI Triage Layer:** unchanged from the original design — a dedicated, least-privilege Wazuh API identity for the triage agent, its own audit trail, human review layer.
- **Phase E — Local AI + n8n Routing:** the `pve-ai`/`ai-vm` build (already largely complete — see above), Ollama, Open WebUI, cloud-model connection, then n8n as the local-vs-cloud router.
- **Phase F — SOAR (Shuffle):** dedicated Shuffle VM on Infra VLAN, wired to Wazuh via webhook, enrichment + notify playbook. Slots in after the detection/attack-surface phases — Phase C.6's kill chain in particular gives the first playbook something real to react to. Active-response (block-IP) is optional and gated on an explicit firewall decision.
- **Phase G — TheHive + Cortex (DEFERRED):** case management + enrichment, added later as one unit. Requires Cassandra + Elasticsearch backends — heavy, ~4–8 sessions with friction. Attempt only once F and E are stable. Wants its own VM.

## Clustering — decided, low priority

Cluster `pve01` + `pve-ai` for a single Proxmox pane. Overhead negligible; the real issue is two-node quorum (survivor drops to read-only if one node is down). Fix with a **QDevice** on the QNAP (Container Station). Fallback: `pvecm expected 1`. Doesn't gate any SOC phase — convenience layer only. This was also flagged as "Phase 7 (optional/later)" in the original, now-merged `pve-ai` plan — same item, one list.

## Session recording — mandatory

- **PuTTY → switch:** before connecting, `Session` → `Logging` → "All session output" → local logs folder. Leave on.
- **PowerShell:** `Start-Transcript` before SSHing anywhere (it's a PowerShell-only command, useless once inside a Linux shell) / `Stop-Transcript`.
- **Linux hosts:** `script ~/session.log` (`-a` append), `exit` to stop.
- **`build_log.md`** — append-only narrative: every command, its **actual** output, decisions, and every rollback point. Never reconstructed after the fact. Don't rewrite history in it. **One structural note:** the file is organized by phase, not strictly by calendar date — the merged AI node build history (dated 2026-07-12/13) sits under its relevant phase heading (Phase E) rather than at the very front where strict chronological order would place it. New entries still get appended as sessions happen; this only affected how the one historical merge was organized.
- **Credential exception:** actual plaintext passwords/secrets typed during a session live in the raw transcript (unavoidable), but must **never** be copied into `build_log.md`, `PROJECT-INSTRUCTIONS.md`, or any other tracked/committed file, and Claude should not repeat them back in chat once set. This applies equally to any Entra ID / Exchange Online admin credentials once Phase C.7 starts.

## SSH quirk (learned the hard way)

`pve01` needs a KEX algorithm workaround (Windows OpenSSH 9.5p2 vs. an older Proxmox OpenSSH build mismatch) — reuse the existing key, and `.ssh/config` needs correct `icacls` permissions on Windows or the key is silently rejected. `wazuh-host` and `pve-ai` both connect fine with SSH defaults, no KEX workaround needed — apparently only `pve01`'s particular OpenSSH build hits the mismatch. The `icacls` permission reset is a recurring issue any time `.ssh/config` gets rewritten, not a one-time fix — expect to reapply it after config edits.

**Separately, a known client-side quirk (see Known Gotcha #4 below):** the Windows OpenSSH client sometimes hangs at 0% CPU *after* a remote command has already completed and returned correct output. This has shown up on both `pve01` and `pve-ai` sessions. Kill and retry — the output already received has been reliable every time it's been checked.

## Standing rules

**End-of-session reminders (Claude should always give both, every session that touched the lab):**
- If `build_log.md` or `agent-registry.md` changed this session, remind me to re-upload the current version to this Project's knowledge files — uploads are static snapshots, not a live sync, so a stale copy here means Claude works from outdated information next time.
- Remind me to run `git add . && git commit -m "<what changed and why>" && git push`.

- **Scope is frozen.** A new idea raised mid-build gets logged as a next-step, not built, unless there's an explicit conversation to unfreeze and re-scope.
- Ask my approval before creating any file, document, image, or phase not already scoped in `LAB-BLUEPRINT.md`.
- Warn me before analyzing screenshots.
- Flag and check in before pursuing tangents.
- One network change at a time, tested before the next.
- Confirm console/recovery access before any change that could lock out management.
- Explain-it-back checkpoints after each phase.
- Verify version- and model-specific syntax against the live system — package names, release tags, switch interface naming, IOS command availability, Wazuh/Shuffle tags, container-internal file paths (don't trust generic vendor doc examples — confirm with `find` inside the actual container), and Windows Server/AD cmdlet syntax, Entra ID/Exchange Online PowerShell module syntax. Never from memory.
- pfSense firewall rule descriptions follow the format `allow <INTERFACE_NAME> -> <PURPOSE> (<PORT>)`, e.g. `allow INFRA20 -> HTTPS (443)` — use the interface's friendly name (MGMT10/INFRA20/RANGE30), not a lowercase/generic label.
- When a step could be done via the Claude in Chrome extension (e.g. clicking through a GUI like pfSense, the Wazuh dashboard, or the Entra ID/Exchange admin centers), always ask whether I want the extension to run it or want to do it myself — never assume one or the other.
- Always explain what a command does (and why) before I run it, not just give the command — applies to every command, not just complex or risky ones.
- **Never write actual passwords/secrets into any documentation file, and don't repeat them back in chat once set.**
- **Before anything that touches networking, boot config, or drivers — on any host, not just `pve01`/`wazuh-host` — state the rollback path first**, out loud, before making the change. This is how the `pve-ai` build's GPU passthrough work (a genuinely risky, hard-to-recover-from category of change) stayed safe.
- **The INFRA20→RANGE30 rule for Entra Connect gets the same two-tier treatment as every other firewall rule, with extra scrutiny given it's the first rule of its kind in this build.**
- **No name appears anywhere in any documentation file.** Write in first person throughout.
- **Claude Code must never guess at credentials or usernames.** If an SSH/login attempt fails with the documented username or key, it must stop and report back rather than looping through alternative usernames — this applies to every host, not just `wazuh-host`. (Learned 2026-08-17: Claude Code tried 18 different usernames against `wazuh-host` in a loop before stopping to ask, when the correct username was already documented in this file's Claude Code allow-rule.)

## Known gotchas — already hit in this environment

1. **Duplicate IP across bridges** → silent ARP failure. Check `ip addr show` on **both**.
2. **Orphaned VM network devices** after a cable/bridge change. Check every VM's Hardware tab.
3. **R710 is legacy BIOS only** — iDRAC6 config is **Ctrl+E during POST, not F2** (5-second window).
4. **The Windows OpenSSH client can hang at 0% CPU after a remote command has already completed successfully** — seen repeatedly across `pve01`, `wazuh-host`, and `pve-ai` sessions. Host reachability on port 22 stays solid throughout; this is a client-side quirk, not a host problem. Kill the stuck process and retry; already-received output has proven reliable every time it's been checked against a follow-up verification.
5. **PowerShell's string-escaping can silently mangle a `sed`/shell command sent over SSH** (e.g. an `unterminated 's' command` error) before it ever reaches the remote host — a safe no-op, not a partial edit, but always verify with a read-back (`cat`/`grep`) rather than assuming success. Building the remote command as a single-quoted PowerShell string, rather than double-quoted with escaped inner quotes, avoids this.
6. **The i9-10900KF has no integrated graphics** — once its GPU is bound to `vfio-pci` for passthrough, the host's physical monitor goes completely dark at boot. Expected, not a failure; doesn't affect SSH/network reachability, since passthrough only changes which driver owns the GPU. Worth stating explicitly before rebooting mid-passthrough-setup so a dark screen isn't mistaken for a failure.
7. **iDRAC6 virtual console needs legacy Java Web Start** — prefer monitor+keyboard on the server.
8. **Prior incident:** a Cisco+pfSense config via a different AI tool caused a lockout requiring a full reinstall. The safety gates are non-negotiable.
9. **A NIC showing no link** may just be administratively down (`ip link set <iface> up`) — check before chasing hardware.
10. **A `build_log.md` entry is not proof a change is still true** — verify live state before trusting a past log entry.
11. **pfSense OPT interfaces get zero firewall rules by default** — only the LAN-role interface gets an automatic allow-all. Any new VLAN/segment needs its outbound rule written explicitly. Expect this again for AD's port list on RANGE30 and Entra Connect's sync traffic on INFRA20.
12. **DNS-only failures vs. hang-after-partial-success are different symptoms.** The first usually means no rules exist at all; the second (small transfers work, larger ones hang) usually means ICMP/Path MTU Discovery is blocked, not DNS or the main port.
13. **A live OS installer's root filesystem may still be the mounted ISO, even late in the install.** Don't eject virtual media until the installer's own "Reboot Now" (or equivalent) prompt actually appears.
14. **`sudo command >> file` fails on root-owned files** even though `command` is elevated — the shell sets up the redirect using my own permissions *before* `sudo` runs. Use `echo "..." | sudo tee -a file` instead, since `tee` itself gets elevated.
15. **Wazuh's `admin` indexer user can't have its password changed through the dashboard UI or API** — it's `reserved: true`. Must edit `internal_users.yml`'s hash directly (matching plaintext in `docker-compose.yml`'s `INDEXER_PASSWORD`), then apply via `securityadmin.sh` inside the indexer container.
16. **Generic `wazuh-docker` doc paths for certs/config didn't match this container's real layout.** Real paths on this deployment: certs at `/usr/share/wazuh-indexer/config/certs/`, security config at `/usr/share/wazuh-indexer/config/opensearch-security/`. Use `find / -iname "<file>" 2>/dev/null` inside the container to confirm rather than trusting doc examples.
17. **`securityadmin.sh` suppresses its own real error output** (`2>/dev/null` baked into the script) — a silent return to prompt after the `JAVA_HOME` warning is not evidence of success or failure either way. If this happens, reconstruct and run the underlying `java org.opensearch.security.tools.SecurityAdmin ...` invocation directly (without the suppression) to see what's actually happening.
18. **This container image bundles its own JDK but doesn't set `JAVA_HOME`/`$PATH` to it.** Real binary: `/usr/share/wazuh-indexer/jdk/bin/java`. Call by full path if a script's Java auto-detection comes up empty.
19. **A Windows Wazuh agent's first-time enrollment needs a separate firewall allow for port 1515**, distinct from the ongoing data-traffic port (1514). Same "segment gets zero rules by default" pattern as INFRA20 — check both ports when standing up any new agent on an isolated VLAN. Expect the AD build to have its own version of this same pattern.
20. **A Range VM has no internet by design — tools must be staged via virtual media, not downloaded in-VM.** Working pattern: download on the management PC → build an ISO → upload to Proxmox's `local` storage → attach as the VM's CD/DVD drive. Will recur for Phase C (Atomic Red Team) and Phase C.6 (BloodHound/SharpHound, Kerberoasting tooling, and other Windows-side attack tooling).
21. **`IStream.Read()` cannot be called directly from PowerShell, in any version (5.1 or 7)** — a fundamental COM interop limitation, not a PS-version issue. Building an ISO from PowerShell requires a small C# helper class compiled via `Add-Type -CompilerParameters @{CompilerOptions='/unsafe'}` to marshal the read correctly.
22. **Selecting a local file for upload (e.g., into Proxmox's ISO Images) requires the OS's native file picker**, which sits outside anything a browser automation tool can see or interact with — this step always needs to be done by hand.
23. **AD domain joins fail silently or with vague errors if DNS isn't pointed correctly** — every domain-joined machine must use `dc01` as its DNS server, not pfSense or the home router. Expected to be the most likely failure point in Phase C.6's AD build.

## Open decisions — resolved (2026-08-06)

These were open earlier in planning and are now decided as part of the scope-freeze:

- **Kill-chain scope** — a real two-host lateral-movement chain (via `win11-ws02`), stopping short of full interactive compromise of `dc01` itself. Domain-wide credential access is instead achieved through the DCSync misconfiguration, a genuine attack path in its own right.
- **`dc01` VLAN placement** — **RANGE30** (it's the attack target, matches the existing pattern of every other target living there).
- **`entra-connect-01` VLAN placement** — **INFRA20**, with the single narrow reviewed rule to `dc01` described above.
- **Domain name** — **`soclab.internal`.**
- **Mail server approach** — **Exchange Online**, not on-prem Exchange, not a standalone relay/GoPhish setup.
- **Attack orchestration** — **fully manual**, no Caldera. I run every red-team and blue-team step myself.
- **Entra ID tier** — **Free + a time-boxed P1 trial**, not a permanent P1 purchase, and not P2.
- **AI node project status** — **merged into this single project**, no longer tracked separately.

## Open decisions — still genuinely open

- **Shuffle active-response (block-IP via pfSense).** Optional. Opens a new path into Range/pfSense — write and review under the two-tier rule only if/when chosen.
- **USB-Ethernet adapters for the SPAN tap.** Chipset unverified; NIC4 is the fallback. One adapter is currently borrowed for PC management access — free it or confirm the second unit before Phase C.5.
- **Clustering sequence.** Decided in principle (QDevice on QNAP); when to actually do it is open.
- **`pve01`/switch nightly full shutdown vs. the blueprint's "always-on service host" design.** Currently a standing habit; conflicts with Phase D+ continuous monitoring intent, and now also with Entra Connect's expectation of running its sync on a regular schedule. Not resolved — worth a deliberate decision before Wazuh/Shuffle/n8n/Entra Connect are all running continuously.

## Explicitly out of scope

See `LAB-BLUEPRINT.md` for the full list and reasoning. In short: MITRE Caldera, Entra ID P2, on-prem Exchange Server, standalone GoPhish + mail relay, and full interactive host-level compromise of `dc01` — all evaluated and deliberately excluded, documented as future next-steps rather than live plan.

## Source of truth

The `.docx` workbooks that used to accompany this build have been retired (2026-08-03) — they duplicated what's already in the `.md` files below and went stale independently. These four files are now the entire documentation set:

- **`LAB-BLUEPRINT.md`** — what I'm building and in what order, including the locked Phase C.6 (AD/kill-chain expansion) and Phase C.7 (IAM/Entra ID), plus Phase F SOAR and the deferred Phase G
- **`build_log.md`** — the running, append-only record, organized by phase rather than strict calendar date; now also incorporating the merged AI node build history under its relevant phase (E)
- **`agent-registry.md`** — every AI agent's scope, owner, lifecycle (`wazuh-triage-01`, `triage-router-01`)
- **`README.md`** — repo-facing overview, phase status table, diagram link

The former `ai-node` project's own `build_log.md` and `hybrid_ai_node_build_plan.md` are retained as historical source material but are no longer separately maintained — everything current lives in the four files above.

## Repo layout note

The repo root is at a Windows path confirmed via `git remote -v` to be where `.git` actually lives, not a parent folder. `wazuh-host` also has its own clone, authenticated via a repo-scoped SSH deploy key (read/write), separate from `pve01`'s SSH key. `wazuh-host` additionally has a second, **local-only, untracked** clone of the upstream `wazuh/wazuh-docker` repo (pinned to tag `v4.14.6`) — third-party deployment tooling, not part of this repo, and should stay that way.

Keep this file short and current. Update it whenever a rule, gotcha, or environment fact changes.
