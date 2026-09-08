# Blueprint — Phases 1 through 6

Part of `LAB-BLUEPRINT.md`'s "Build Phases" section, split out 2026-09-07 to stay under the ~16,000-character tool-read limit. See `LAB-BLUEPRINT.md` for the full purpose, environment, and phase index; see `Blueprint/Phase 7 AD Expansion.md` and `Blueprint/Phases 8-12.md` for the rest.

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

