# Blueprint — Phase 7, AD Expansion

Part of `LAB-BLUEPRINT.md`'s "Build Phases" section, split out 2026-09-07 to stay under the ~16,000-character tool-read limit. Phase 7 alone earns its own file, it's the single largest phase in the build. See `LAB-BLUEPRINT.md` for the full purpose, environment, and phase index; see `Blueprint/Phases 1-6.md` and `Blueprint/Phases 8-12.md` for the rest.

### Phase 7 — AD Expansion ⏳ NOT STARTED

**Why:** the lab as built through Phase 5 proves single-host detection engineering against one isolated Windows victim. It can't demonstrate lateral movement to a genuinely separate host, a real initial-access vector, AD-specific credential and certificate attacks, or Discovery-stage tooling (BloodHound). This phase closes those gaps with a realistic, full-depth Active Directory environment and one coherent, end-to-end kill chain.

**Scope: a real two-host lateral movement chain, deliberately stopping short of full interactive compromise of `dc01` itself.** The chain runs foothold → privilege escalation → credential/certificate access → discovery → lateral movement (to a genuinely separate, previously-uncompromised second workstation) → persistence → exfiltration/impact. Where domain-wide credential material is needed (e.g. via a DCSync misconfiguration), it's obtained through that AD-level abuse rather than by gaining an interactive shell on `dc01`, a real, valid attack path in its own right, but one that stops short of treating `dc01` as a fully compromised host with its own persistence/impact stage. That distinction is a deliberate scope boundary, documented as a next-step in the eventual Phase 7 writeup, not a shortfall.

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
    - Write it up as one continuous case narrative across the phase's `Build Logs/` step folders, from Initial Access through Impact, including an explicit "next steps" section noting that full interactive compromise of `dc01` itself was intentionally out of scope.
    - **Acceptance check:** the full chain runs start to finish against the live environment in one sitting, with a Wazuh and/or Suricata detection confirmed at every named stage, BloodHound's full analysis workflow demonstrated (not just the collector run), and the writeup accurately reflecting what actually happened.

---

