# LAB-BLUEPRINT.md
### Home SOC Lab, what I'm building, and in what order

**Purpose:** My shared map with Claude. What gets built, in what sequence, and how I know each phase actually worked. I run every command myself; Claude guides one step at a time and reads back the output. `build_log.md` is a short index into `Build Logs/`, which carries the detailed, dated narrative of every step I've actually taken, the `.docx` workbooks that used to accompany this file were retired (2026-08-03) in favor of the `.md` files being the sole source of truth.

**Working principle:** Build incrementally. One change, one test, then the next change. Don't stack multiple untested changes (network config in particular, this environment has a documented history of lockouts from big-bang changes).

**Scope status (2026-08-06):** This document reflects the full, locked scope of the build, start to finish, Phases 1 through 12, including Phase 7 (AD/kill-chain expansion, now including a second workstation and a fully realistic AD environment) and Phase 8 (IAM/Entra ID). Everything in it is committed, planned work, assembled across an extended planning conversation. New ideas raised mid-build get logged as next-steps in the relevant `investigations/` writeup or as a note in `PROJECT-INSTRUCTIONS.md`, they don't get folded into active scope without a deliberate decision to revisit and re-lock. See "Explicitly out of scope" near the end for everything already evaluated and deliberately excluded.

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

### Phase 1 — Network Rebuild ✅ COMPLETE (2026-07-25)
VLANs, trunk, management access port, pfSense VM + VLAN interfaces all built and acceptance-checked. Snapshot `pfsense-clean-install` taken. See `build_log.md` / `Build Logs/Phase 01 Network Rebuild/`.

### Phase 2 — Isolation Rule ✅ COMPLETE
1. Write (do not yet apply) a pfSense rule: Range → deny all, except one explicit allow to Infra's Wazuh ingest port.
2. Review the rule text before applying, confirm it doesn't block the interface pfSense is managed from.
3. Apply; acceptance-check from a Range VM (no internet, no home net, no MGMT; Wazuh port retested end of Phase 3).

### Phase 3 — Docker/Git/Wazuh Substrate ✅ COMPLETE (2026-08-04)
1. ✅ Build Ubuntu Server VM on `pve01`, Infra VLAN. **8GB RAM / 4 cores / 100GB+ disk.** Enable OpenSSH. *(2026-07-31, `wazuh-host`, `10.10.20.100`, SSH confirmed. INFRA20 outbound rules written/tested along the way.)*
2. ✅ Host prep: `sudo sysctl -w vm.max_map_count=262144`, persist in `/etc/sysctl.conf`. *(2026-08-03)*
3. ✅ Install Docker + Compose via the apt repo method (**not** `curl | sh`); verify package names live. *(2026-08-03, Docker CE 29.7.1, verified with `hello-world`, non-root usage enabled.)*
4. ✅ Install Git, create/clone the repo. Pre-push secret check. *(2026-08-03, cloned to `~/soc-lab` via a dedicated SSH deploy key. INFRA20 needed a 5th rule, port 22, discovered along the way.)*
5. ✅ Deploy Wazuh (manager + indexer + dashboard) via Compose; check the current tag against `documentation.wazuh.com`. Change the default password. Mount rules host-side (`./config/rules/local_rules.xml`). *(2026-08-03/04, Wazuh 4.14.6 deployed, admin password rotated via the `internal_users.yml`/`securityadmin.sh` procedure.)*
6. ✅ Move Win11-LTSC-victim to Range VLAN, install Sysmon + Wazuh agent pointed at the Sysmon channel. *(2026-08-04, isolation proven live, agent ID `001` active.)*
7. Move Kali to Range VLAN, **not done in Phase 3, moved to Phase 6 step 1. Completed 2026-08-18, see Phase 6 below.**
8. ✅ Re-test the isolation rule: victim reaches Wazuh on 1514/1515, still no internet. Confirmed 2026-08-04.
9. ✅ **Acceptance check:** dashboard shows the victim active with Sysmon events. Confirmed.

**Phase 3 follow-up (SSH key-only hardening on `wazuh-host`) is now complete**, see Phase 4 entry below.

---

### Phase 4 — Detection Engineering ✅ COMPLETE
1. ✅ Snapshot Win11-LTSC-victim (`pre-atomic-clean`).
2. ✅ Install Atomic Red Team **on the victim** (it runs on the box I want telemetry from), reusing the ISO-staging pattern proven in Phase 3 (download on the management PC → build ISO → attach as virtual CD, Range has no internet by design). Added the `C:\AtomicRedTeam` Defender exclusion, safe only because the VM is fenced (Phase 2).
3. ✅ Ran a first technique (`T1059.001`) locally, confirmed Wazuh alerts.
4. ✅ Wrote six custom detection rules. **I wrote every final rule**; Claude drafted a reference for each. Rules live host-side in `config/rules/local_rules.xml`, IDs `100002`–`100007`:
   - `100002`/`100003`, PowerShell spawned by a suspicious parent process, escalating on encoded commands (T1059.001/T1027, Execution)
   - `100004`, registry Run/RunOnce key persistence (T1547.001, Persistence)
   - `100005`, LSASS credential dumping via Silent Process Exit / IFEO GlobalFlag abuse (T1003.001 + T1546.012, Credential Access)
   - `100006`, local account/group enumeration via `net.exe` (T1087.001, Discovery)
   - `100007`, network share removal via `net.exe` (T1070.005, Defense Evasion)

   Five tactics covered across six rules. All chained via `if_sid` onto specific built-in rule IDs, `100002`/`100004` were both initially chained via the broader `if_group` and silently short-circuited by an earlier-loading built-in rule; both were found and re-chained onto `if_sid`, closing the same structural risk across the full rule set.
5. ✅ Committed rules/config-as-code.
6. ✅ **Acceptance check:** every rule, of my own authorship, confirmed firing on a fresh atomic-test re-run, verified via direct `archives.json` inspection (the trustworthy method in this environment, `wazuh-logtest` fed raw JSON does not reliably reproduce the real Sysmon match path).

**Complete (2026-08-11):** SSH key-only hardening on `wazuh-host`, personal key pair generated and installed, `PasswordAuthentication no` set and verified (password auth rejected, key auth succeeds), Claude Code's existing non-interactive key confirmed unaffected. See `Build Logs/Phase 04 Detection Engineering/` for full session detail.

**Known gap, not yet resolved:** Sysmon's current config on Win11-LTSC-Victim does not capture FileDelete-family events (Event ID 23/26). Attempted enabling Event 26 on 2026-08-18, config validated and reloaded clean, but the event never fired in the live driver; root cause not isolated. Next hypothesis is Defender/PPL interaction, not an XML/config-syntax issue. File-deletion-based detections (T1070.004 and similar) remain blocked until this is deliberately revisited.

**Still open:** rotate the `wazuh-host` account password itself (was briefly typed in plaintext by an automation tool early in the build, caught before use). Key-only SSH login is enforced and closes the access risk, but does not close this item.

---

### Phase 5 — Network Visibility (SPAN + Suricata) ⏳ FUNCTIONALLY COMPLETE, PERSISTENCE OUTSTANDING

**What actually happened (this section previously described a not-yet-started plan; it's been rewritten to match reality as of 2026-09-06):**

1. ✅ Configured a SPAN session on the Cisco switch: source = RANGE30, destination = `Gi1/0/4` (receive-only, ingress disabled).
2. ✅ Passed the SPAN destination through to `pve01` NIC4/`eno4` → a new bridge `vmbr4` → `wazuh-host`'s second NIC `ens19`. Monitoring interface has no IP.
3. ✅ Installed Suricata 8.0.6 (OISF PPA) directly on `wazuh-host`, bound to `ens19`, running the Emerging Threats Open ruleset (52,713 rules enabled). Integrated `eve.json` into Wazuh via a bind mount plus a `<localfile>` block in `ossec.conf`.
4. ✅ Resolved a `Too many fields for JSON decoder` blocker by disabling `stats` inside the eve-log `types:` block (the separate global stats module writing to `stats.log` stayed enabled). This had actually been fixed since the prior session; a verification method that sampled an append-only log by line count instead of by timestamp had masked the fix.
5. **Architectural finding, not part of the original plan:** the physical switch SPAN proved blind to almost all east-west traffic. Kali (10.10.30.101), Win11-LTSC-Victim (10.10.30.100), and pfSense all run on `pve01` attached to the same Linux bridge (`vmbr3`), and a Linux bridge switches VM-to-VM frames in software without ever egressing to the physical Cisco switch, so the SPAN only ever saw broadcast/multicast noise (ARP visible, TCP absent, Wazuh agent traffic on 1514 absent entirely). Confirmed via simultaneous `tcpdump` on `ens19` and a fresh `nmap` scan.
6. ✅ Remediated at the hypervisor layer instead of the switch: `tc mirred` ingress mirrors added on `tap100i0` and `tap101i0` (Kali's and the victim's tap interfaces), targeting `tap103i1` (`wazuh-host`'s tap). Verified bidirectionally via a captured SYN/SYN-ACK exchange on port 3389. A live Suricata alert (rule.id `86601`, `ET INFO Possible Kali Linux hostname in DHCP Request Packet`) confirmed the full path end to end: VM tap → `tc` mirror → `ens19` → Suricata → `eve.json` → Wazuh manager → indexer → dashboard, visible in Threat Hunting under `wazuh.manager`.
7. **Not yet done:** the `tc` mirrors are runtime-only, they do not survive a `pve01` reboot or a VM restart (which recreates the tap and silently drops that VM's mirror). Proxmox hookscripts are the preferred persistence mechanism, since they re-apply on both VM start and host boot. This is the one remaining item before Phase 5 is fully closed.

**Acceptance check status:** the original acceptance check (a network attack from Kali showing up in Wazuh from Suricata and having a host-based alert, two layers, one story) has been demonstrated live via the DHCP-hostname alert above. What's outstanding is persistence across reboots, not detection capability itself.

---

### Phase 6 — Attack Surface ⏳ IN PROGRESS

**Why a separate phase from AD Expansion:** this phase covers the infrastructure hardening that should happen before the AD build starts, moving Kali into the Range, forwarding firewall logs, and standing up remote access, none of which depend on `dc01` existing. Splitting it out from AD Expansion (Phase 7) keeps each phase's log file a manageable size and keeps "infrastructure prep" separate from "AD-specific build work" as a reader scans phase status.

**Sub-steps:**

1. ✅ **Move Kali to RANGE30, complete 2026-08-18.** Isolation confirmed live via three pings (no internet, no home/MGMT reach, no ICMP to INFRA20).
2. **Forward pfSense logs** (firewall pass/block, DHCP, DNS) into Wazuh. **Not yet done.**
3. **Configure a pfSense remote-access WireGuard VPN**, decided over OpenVPN for its simpler configuration surface (fewer places for a subtle rule mistake), better performance, and native integration in pfSense CE since 2.5. A second, credential-based Initial Access vector, distinct from and complementary to the web-app exploit path in Phase 7. I write the final firewall rules, Claude drafts a reference. **Not yet done.**

**Acceptance check:** all three sub-steps complete and verified live before AD work in Phase 7 begins.

---

### Phase 7 — AD Expansion ⏳ NOT STARTED

**Why:** the lab as built through Phase 5 proves single-host detection engineering against one isolated Windows victim. It can't demonstrate lateral movement to a genuinely separate host, a real initial-access vector, AD-specific credential and certificate attacks, or Discovery-stage tooling (BloodHound). This phase closes those gaps with a realistic, full-depth Active Directory environment and one coherent, end-to-end kill chain.

**Scope: a real two-host lateral movement chain, deliberately stopping short of full interactive compromise of `dc01` itself.** The chain runs foothold → privilege escalation → credential/certificate access → discovery → lateral movement (to a genuinely separate, previously-uncompromised second workstation) → persistence → exfiltration/impact. Where domain-wide credential material is needed (e.g. via a DCSync misconfiguration), it's obtained through that AD-level abuse rather than by gaining an interactive shell on `dc01`, a real, valid attack path in its own right, but one that stops short of treating `dc01` as a fully compromised host with its own persistence/impact stage. That distinction is a deliberate scope boundary, documented as a next-step in the eventual `investigations/` writeup, not a shortfall.

**Explicitly excludes MITRE Caldera.** Caldera was evaluated as a way to autonomously orchestrate the post-compromise portion of this chain. Decided against: real setup cost (~4–6 hrs) for less time savings than expected, since most of this phase's hours are in understanding techniques and writing detections, not command execution, and it would work against the chosen interview narrative of executing this myself, end to end.

**Steps (matching `Build Logs/Phase 07 AD Expansion/` step folders):**

1. **Forest and Domain Build:**
   - Stand up `dc01`, Windows Server 2022, Core install, on RANGE30, sized modestly (2 vCPU / 4–8GB). **VM creation and Windows install are Claude Code-executed**, genuinely new territory, delegated as a deliberate time-saving trade-off given the overall scope size, not because it repeats known work.
   - `Install-WindowsFeature AD-Domain-Services`, **Claude Code-executed**, mechanical, no decision content.
   - `Install-ADDSForest`, new forest and domain, **`soclab.internal`** (not `.local`, to avoid mDNS conflicts; this is the forest root, not a join to an existing one). **Manual, I run this myself.** This is where the real decisions live: forest/domain functional level, DNS strategy, NetBIOS name, DSRM password. Claude explains each parameter before I run it.
   - **I choose and record the DSRM password myself** (Keeper, never in docs), same handling as every other credential in this build. A real break-glass-equivalent for the DC's own boot-level recovery, distinct from but conceptually parallel to the directory's own break-glass account.
2. **Kerberos and Verification:**
   - **I configure Kerberos Policy** (Default Domain Policy, max ticket lifetime, max renewal age, clock skew tolerance) and **the PDC Emulator time source** (`w32tm` setup) myself, this is the concrete, hands-on version of learning Kerberos, not an abstract concept. The clock skew tolerance is directly why NTP (123) is in the AD firewall port list below.
   - **I personally run post-build verification** (`dcdiag`, SYSVOL replication health, confirming DNS is authoritative, reviewing the default OU/GPO structure `Install-ADDSForest` leaves behind), this is the real "is my DC healthy" acceptance check, not something to accept on a "done" message alone.
   - **Expect a DNS gotcha here**, the single most common AD lab failure point. `dc01`'s own DNS must be authoritative for `soclab.internal`, and every future domain-joined machine must point its DNS at `dc01`, not pfSense or the home router.
   - Write and apply the AD-specific firewall rule set on RANGE30: DNS (53), Kerberos (88), LDAP (389)/LDAPS (636), SMB (445), RPC endpoint mapper (135) plus a dynamic RPC range (49152–65535), NTP (123). Expect this to grow one rule at a time, same pattern as INFRA20's growth from zero to five rules in Phase 3. **These rules are Claude Code-executed**, each one repeats the same shape already established for INFRA20, no new judgment required per rule.
   - Join Win11-LTSC-Victim and the new `win11-ws02` to `soclab.internal`. **Both domain joins are Claude Code-executed** (decided 2026-08-04), real prior hands-on experience with domain joins. **`win11-ws02`'s build (VM creation, Windows install, Sysmon, Wazuh agent enrollment) is also Claude Code-executed**, repeats the exact pattern already proven manually for Win11-LTSC-Victim in Phase 3.
   - **Configure Advanced Audit Policy via GPO**, Kerberos service ticket operations (4769), directory service access (4662), object access, and related event categories. **This is a technical requirement, not optional realism:** without it, Windows never generates the events the Kerberoasting, DCSync, and lateral-movement detection rules in this phase actually depend on. I write the final GPO, Claude drafts a reference.
3. **Directory Structure and RBAC:**
   - Design an OU structure reflecting a tiered admin model, Tier 0 (identity/`dc01`), Tier 1 (servers), Tier 2 (workstations), even if only partially enforced. I write the final OU/GPO design, Claude drafts a reference. **The design decision itself stays manual; only account creation, OU placement, and group *membership* assignment below are delegated, not group creation.**
   - Create 3–4 phantom computer objects (`New-ADComputer`, no live VM behind them) and place them across the OU structure alongside the real hosts, for directory-scale realism and BloodHound mapping.
   - I create one user account by hand; **Claude Code executes the creation of the remaining ~12** (varied departments/roles, distributed across OUs), given prior real-world experience creating AD accounts, assigning groups, and placing objects in OUs.
   - Create security groups reflecting realistic nested permissions, including at least one deliberate over-privileged nesting (a low-tier group inheriting Domain Admin-adjacent rights through the nesting itself, not direct membership). **Which groups exist, how they nest, and the actual group-creation commands are manual/two-tier**, I've only ever managed membership on already-existing groups, not created new ones, so this stays alongside the nesting design rather than being delegated. **Assigning existing users to the created groups is Claude Code-executed**, matching real prior experience with group membership assignment specifically.
   - Create one service account with an SPN set and a deliberately weak password, the Kerberoasting target.
   - Create one break-glass/emergency admin account, explicitly not touched during the attack chain, called out as such in the writeup.
   - Create two or three stale/inactive "former employee" accounts, left enabled.
4. **Deliberate Misconfigurations (the actual attack surface for Discovery/Credential Access):**
   - A risky ACL: a helpdesk-tier account granted `GenericAll`/`WriteDACL` on a privileged object.
   - Unconstrained Kerberos delegation configured on a server account.
   - DCSync rights (`GetChanges`/`GetChangesAll`) granted to a non-obvious account (e.g. a "backup service" account), the path to domain-wide credential material without an interactive shell on `dc01`.
   - **Active Directory Certificate Services (ADCS)** deployed with one deliberately vulnerable template (**ESC1**, client-auth EKU plus enrollee-supplies-subject enabled for low-privilege enrollers).
   - SYSVOL/Group Policy Preferences password exposure (a GPP-stored, recoverable credential).
   - A shadow admin, an account with Domain Admin-equivalent *rights* via ACL abuse, without Domain Admins group membership, discoverable only through graph-based analysis, not a group-membership check.
   - **No LAPS deployed initially**, local admin credential reuse across hosts is the gap that enables the lateral-movement pivot from the first compromised host to `win11-ws02`. LAPS gets deployed in step 9 as the fix, and the pivot re-tested to confirm it's closed, a before/after story mirroring Shuffle's manual-vs-automated comparison in Phase 11.
5. **GPO Deployment and Shares:**
   - Deploy at least one benign application (7-Zip or Notepad++) via GPO Software Installation.
   - Deploy mapped drives to file shares via Group Policy Preferences.
   - Create department-scoped file shares (Finance, HR, IT) with realistic, imperfect ACLs, at least one deliberately over-broad (e.g. Domain Users granted Read on the Finance share), this is what makes the later exfiltration stage meaningful rather than arbitrary.
6. **Linux Victim and Web App:**
   - Build `linux-victim` on RANGE30, deliberate sudo misconfiguration, `auditd` rules (I write the final, Claude drafts a reference) + Wazuh agent.
   - Deploy a vulnerable web app (DVWA or Juice Shop) via Docker Compose.
7. **Attacker Tooling on Kali:**
   - Install and configure: Responder, NetExec, Impacket (including `ntlmrelayx.py`), Certipy, Rubeus, PowerView, BloodHound/SharpHound, plus whatever Kerberoasting/AS-REP roasting tooling is already staged.
   - Stand up Sliver C2 (server + implant).
8. **Kill Chain Execution**, run the chain, confirming a detection at each stage before moving to the next:
   - **Initial Access**, exploit the web app, or use the VPN path, to land a shell. Real, manual exploitation.
   - **Windows Privilege Escalation**, unquoted service path / weak service permissions on the first compromised Windows host.
   - **Discovery**, run SharpHound with **session data collection enabled**, not just static structure. In BloodHound: mark high-value targets, run "Shortest Paths to High Value Targets," run the built-in queries for Kerberoastable users, AS-REP roastable users, unconstrained delegation, and DCSync rights, and write at least one hand-crafted Cypher query.
   - **Credential/Certificate Access**, Kerberoasting and/or AS-REP roasting against the SPN'd service account; ADCS ESC1 abuse via Certipy; DCSync via the misconfigured rights above.
   - **Lateral Movement**, pivot from the first compromised host to `win11-ws02`, exploiting the pre-LAPS local admin credential reuse gap, via PsExec/WMI/Pass-the-Hash. Costed and executed as its own distinct step.
   - **Persistence**, a registry run key or scheduled task, or a persistent Sliver implant.
   - **Exfiltration**, pull data from the over-broadly-permissioned file share, exfiltrated via DNS tunneling or HTTPS to an unusual destination.
   - **Impact**, a benign mass file-rename script paired with Wazuh File Integrity Monitoring, simulating ransomware-style behavior.
   - Write a Wazuh (and where applicable Suricata) detection rule for each stage before moving to the next, I write the final rule, Claude drafts a reference.
9. **LAPS Fix and Pivot Retest:** deploy LAPS as the fix, re-test the lateral-movement pivot, confirm it's closed, the explicit before/after comparison.
10. **Writeup and Acceptance:**
    - Write it up as one `investigations/` case, a single continuous narrative from Initial Access through Impact, including an explicit "next steps" section noting that full interactive compromise of `dc01` itself was intentionally out of scope.
    - **Acceptance check:** the full chain runs start to finish against the live environment in one sitting, with a Wazuh and/or Suricata detection confirmed at every named stage, BloodHound's full analysis workflow demonstrated (not just the collector run), and the `investigations/` writeup accurately reflecting what actually happened.

---

### Phase 8 — Hybrid Identity ⏳ NOT STARTED

**Why:** directly targets the IAM Analyst role I'm pursuing, Conditional Access, MFA policy, and hybrid identity are core day-to-day IAM work, not just an attack surface. Independent of the rest of the lab's hardware for its first step; ties into Phase 7 once `dc01` exists.

**Sub-steps:**

1. **Entra Connect Sync:** Entra ID Free tenant, sign up (no time limit; the Free tier is permanent). Build out a basic directory structure in parallel with other phases. Build `entra-connect-01` on INFRA20, install Microsoft Entra Connect, sync `dc01` → Entra ID once Phase 7's DC exists. **VM creation and the Entra Connect software install are Claude Code-executed**, mechanical, no decision content. **Everything after that is manual:** free-tier capable, full user/group/attribute sync plus one-way Password Hash Sync; only password *writeback* specifically needs P1. Requires the narrow INFRA20→RANGE30 rule described above (I write the final rule), plus outbound rules for `entra-connect-01` to reach Microsoft's sync endpoints. I verify sync landed correctly and test SSO against a cloud resource myself. **I confirm the sync account's scope in `agent-registry.md`**, dedicated service account, minimum AD replication/read permissions, not a Domain Admin account, before the sync goes live; this scoping is the actual security teaching point here (the sync account is a well-known real-world attack target, the same risk pattern the DCSync misconfiguration in Phase 7 demonstrates), not administrative overhead, so it stays manual regardless of what else is delegated.
2. **Exchange Online and Phishing Sim:** Exchange Online (Plan 1, $4/user/month), real mailboxes tied to the synced users. **Mailbox creation is Claude Code-executed**, mechanical, similar to AD account creation. **Attack Simulation Training campaign design stays manual**, choosing scenarios and analyzing results is the actual phishing-detection learning. Delivered via Defender for Office 365's **Attack Simulation Training**, Microsoft's own sanctioned tool, chosen specifically to avoid automated-abuse-detection ambiguity. Mail-flow/EOP logs become a genuine detection source.
3. **Conditional Access and Cloud Enumeration:**
   - **P1 trial activation (time-boxed, 30 days), activate last, only once ready to use it immediately:**
     - Build and test Conditional Access policies against the synced hybrid users.
     - Run MFA scenarios via the Microsoft Authenticator app for real push notifications.
   - **AADInternals / ROADtools**, cloud identity enumeration, the Entra-ID equivalent of BloodHound. The deliberate connective piece between Phase 7 and Phase 8, extending the on-prem chain into the cloud identity layer, one continuous compromise story rather than two disconnected efforts.
   - **Acceptance check:** hybrid identity confirmed working end to end; at least one Conditional Access policy built, tested, and enforcement confirmed live during the P1 window; AADInternals/ROADtools enumeration successfully run with a corresponding detection or log reviewed.

**Explicitly out of scope:** Entra ID P2 (Identity Protection's risk-based sign-in scoring, Privileged Identity Management, Access Reviews), more architect-tier than day-to-day analyst work.

---

### Phase 9 — AI Triage Layer + Agent-Identity Governance ⏳ NOT STARTED
Subscription-covered Claude Code / local model, no separate API billing for Claude Code.
1. **Give the triage agent its own identity, not mine**, dedicated Wazuh API user, not admin, not my own session.
2. **Scope it least-privilege**, `alerts:read`, `agent:read`; no delete, no config, nothing outside the Infra alert pipeline.
3. **Separate its audit trail**, every action logs `agent=wazuh-triage-01`; my review logs `user=analyst` (see `PROJECT-INSTRUCTIONS.md` for the exact identifier convention).
4. **Create/maintain `agent-registry.md`**, the living record of every AI agent.
5. Build the triage service on the Infra host: pull new alerts → model → verdict suggestion, under the agent's own credential.
6. I review/confirm/override, logged separately.
7. **Acceptance check:** an alert flows detection → Wazuh → agent verdict → reviewed verdict, both logged distinguishably; registry matches reality.

This phase benefits directly from Phase 7 having run first, a full kill chain produces a much richer, more realistic alert stream for the triage layer to work against than isolated Atomic Red Team tests alone, which is part of why 7/8 are sequenced ahead of 9.

---

### Phase 10 — Local AI Model Integration + n8n Routing ⏳ NOT STARTED (partially complete, GPU node built)

The underlying `pve-ai`/`ai-vm` infrastructure (GPU passthrough, Ubuntu VM, NVIDIA driver) is already built and confirmed working, see "Confirmed Environment" above. What's left picks up at Ollama itself:

1. **Install Ollama on `ai-vm`** (official install script), verify the service is running.
2. **Pull a starter model sized for 8GB VRAM**, `llama3.1:8b` (quantized default). Test with `ollama run llama3.1:8b "hello"`, confirming via `nvidia-smi` that GPU memory is actually in use, not a silent CPU-only fallback.
3. **Install Open WebUI** (Docker method) on `ai-vm`, bound to its IP, connected to the local Ollama instance.
4. **Configure Open WebUI's cloud connection**, add the Anthropic API under external connections. I supply the API key myself directly in the Open WebUI settings; Claude never asks for it in chat.
5. **Verify from the management PC:** Open WebUI loads in a browser, can chat with both the local model and the cloud model from the same interface.
6. **Snapshot `ai-stack-working`** once confirmed.
7. **Confirm `ai-vm` reachable from the Infra host** and Ollama's API responding (`curl http://192.168.0.203:11434/api/tags`), requires an Infra→ai-vm route/rule if not already permitted.
8. **Stand up n8n** (service container on `pve01`) as the router: routine/low-severity → local Ollama; ambiguous/high-severity → escalate to `claude -p` (subscription) or Grok. I write the routing threshold.
9. **Register the router in `agent-registry.md`** as `triage-router-01`, a second non-human identity making autonomous escalate/local decisions.
10. **Acceptance check:** one full triage cycle completes on the local model with no external call; a second escalates correctly; registry entry accurate.

---

### Phase 11 — SOAR (Shuffle) ⏳ NOT STARTED
**Why:** SOAR (Security Orchestration, Automation, and Response) is directly named on the SOC job listings I'm targeting. Shuffle is the community-standard open-source SOAR engine and pairs natively with Wazuh. Slots in after the detection/attack-surface work (Phases 4 through 8), Phase 7's full kill chain in particular gives Shuffle's first playbook something substantial to react to.

**Lane discipline:** Shuffle = SOC incident response. n8n = AI orchestration + general automation. Do not blur them; do not build Shuffle and n8n in the same stretch.

1. Build a dedicated **Shuffle VM** on `pve01`, Infra VLAN (own VM, do not co-locate with Wazuh; both are memory-hungry). Docker deploy. **Register `shuffle-playbook-01` in `agent-registry.md`** with its own Wazuh API credential (separate from `wazuh-triage-01`'s) and its own enrichment API keys, before wiring it to anything live.
2. Wire Wazuh → Shuffle: add a `<integration>` webhook block in Wazuh so alerts POST to a Shuffle workflow.
3. Build a first playbook (I write the final logic, Claude drafts a reference): alert in → parse IOCs → **light enrichment** (VirusTotal/AbuseIPDB via HTTP) → notify.
4. **(Optional, deliberate)** Active-response step (e.g. block an IP via pfSense). Gated on an explicit, separately-reviewed firewall decision. Not built by default. If chosen, `shuffle-playbook-01`'s pfSense credential is scoped to nothing but adding entries to one specific block-list alias, nothing else, per `agent-registry.md`.
5. Commit the playbook/config-as-code.
6. **Acceptance check:** a real detection fires (ideally from Phase 7's chain) → Shuffle receives it → enriches → produces a case/notification end to end.

---

### Phase 12 — Case Management (TheHive + Cortex), DEFERRED
**Not started. Added later as one matched-pair unit once everything else is stable.**
- **TheHive** = case management (tickets, timeline, tasks, observables, status).
- **Cortex** = enrichment/analysis engine, overlapping with Shuffle's light enrichment but systematic.
- **Why deferred:** TheHive requires **Cassandra** (case DB) + **Elasticsearch** (search/index) as backends, all version-matched. Coexisting with Wazuh's own indexer is the specific friction point. Realistic cost: ~4–8 sessions, mostly version-matching and startup-order issues.
- **Sequence:** attempt only after Phases 10 and 11 are solid. Wants its own dedicated VM.
- `investigations/` writeups in Git serve as case documentation in the meantime.

---

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
- **Full interactive host-level compromise of `dc01`**, the DCSync misconfiguration provides a genuine, real-world path to domain-wide credential material without needing a shell on the DC itself. Documented as a deliberate scope boundary and a future `investigations/` next-step, not a shortfall.

---

## Clustering, decided, low priority

Cluster `pve01` + `pve-ai` for a single Proxmox pane. Overhead negligible; the real issue is two-node quorum (survivor drops to read-only if one node is down). Fix with a **QDevice** on the QNAP (Container Station). Fallback: `pvecm expected 1`. Doesn't gate any SOC phase, convenience layer only.

---

## Source of truth

The `.docx` workbooks that used to accompany this build have been retired (2026-08-03), they duplicated what's already in the `.md` files below and went stale independently. These four files are now the entire documentation set:

- **`LAB-BLUEPRINT.md`** (this file), what I'm building and in what order, including the locked Phase 7 (AD/kill-chain expansion) and Phase 8 (IAM/Entra ID), plus Phase 11 SOAR and the deferred Phase 12
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
