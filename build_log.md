# Build Log Index

This file is an index, not the log itself. The actual dated, narrative entries live under `logs/`, split by phase and, where a phase has more than one step, by step. Read the phase folders in number order for the build in blueprint order.

## Why this changed (2026-09-06)

The single flat `build_log.md` had grown to roughly 250,000 characters. That made it slow to read, unreliable to search, and it silently exceeded the 16,000 character view limit tools use to read a file, which can drop the middle of a long file without warning. Splitting by phase, and by step within a phase, keeps every file comfortably readable start to finish.

No content was rewritten during the split. Entries were moved into their phase's file as they were originally written, adapted only for the no-dash prose style already used elsewhere in this documentation set.

**One data quality issue found during the split, now resolved:** the prior flat `build_log.md` had a structural defect, most likely a side effect of the 2026-08-24 git-history reconstruction noted in its own text. Past roughly its own line 2,382, the file restarted from the top and duplicated its own header and rules content, and several entries (SSH hardening, rules 100004 through 100007, the FileDeleteDetected investigation, the Kali RANGE30 move) appeared a second time later in the file, in a different location than their first appearance. The split used the first, more complete occurrence of each entry as source and discarded the duplicate. If anyone compares this new structure against a saved copy of the old flat file and finds an entry looks shorter or missing, check whether it was one of the duplicates first before assuming content was lost.

## Phase numbering (permanent, 2026-09-06)

Phases are whole numbers, 1 through 12, and never get renumbered or reused. This replaces the old letter based phases (A, A.5, B, C, C.5, C.6, C.7, D, E, F, G).

| Phase | Name | Was |
|---|---|---|
| 1 | Network Rebuild | A |
| 2 | Isolation Rule | A.5 |
| 3 | Wazuh Substrate | B |
| 4 | Detection Engineering | C |
| 5 | Network Visibility | C.5 |
| 6 | Attack Surface | C.6, sub-step 1 (infra hardening: pfSense log forwarding, Kali isolation, VPN) |
| 7 | AD Expansion | C.6, sub-steps 2 through 11 (AD/DC build through kill chain, LAPS, writeup) |
| 8 | Hybrid Identity | C.7 |
| 9 | AI Triage Layer | D |
| 10 | Local AI and Routing | E |
| 11 | SOAR | F |
| 12 | Case Management | G |

## Repo layout

`logs/Phase NN Name/Step N Name/N.N Name.md`. Phase folders are zero padded (01 through 12) so they sort correctly on disk. Files inside a phase folder never repeat the phase name in their own filename. Numbering within a step starts at `.1`, never `.0`.

A phase that is complete with a single entry gets one flat file directly in the phase folder, no Step subfolder (Phase 2 is the only one that currently qualifies). A phase still in progress always uses Step subfolders, even while it only has one step so far, so nothing ever needs renaming later as more steps are added (Phase 6 and Phase 10 both currently have only one real step each, but keep the Step folder for this reason).

Files split at natural content boundaries, the end of a coherent unit of work, staying under 16,000 characters as a firm ceiling, never a target to hit. A shorter file is fine and preferred when the real break falls earlier. Growth only ever adds the next number in a sequence, or one level deeper (`2.1` to `2.2`, or `2.3` to `2.3.1`/`2.3.2`) — nothing already assigned is ever renamed.

## Phase index

| # | Folder | Covers | Status |
|---|---|---|---|
| 01 | [`logs/Phase 01 Network Rebuild/`](./logs/Phase%2001%20Network%20Rebuild/) | Host and management access, Cisco switch identification and factory reset, VLAN 10/20/30 creation, pfSense VM and its three VLAN interfaces | Complete |
| 02 | [`logs/Phase 02 Isolation Rule/`](./logs/Phase%2002%20Isolation%20Rule/) | RANGE30 default-deny firewall rule, isolation verified live | Complete |
| 03 | [`logs/Phase 03 Wazuh Substrate/`](./logs/Phase%2003%20Wazuh%20Substrate/) | Ubuntu Server host build, INFRA20 outbound rules, Docker and Git setup, Wazuh 4.14.6 Compose deploy, first Wazuh agent enrolled on Win11-LTSC-Victim | Complete |
| 04 | [`logs/Phase 04 Detection Engineering/`](./logs/Phase%2004%20Detection%20Engineering/) | Atomic Red Team staging, all six custom detection rules (100002 through 100007) written and verified, the if_group to if_sid chaining fix, SSH key-only hardening on wazuh-host, the unresolved FileDeleteDetected (Event 26) investigation | Complete, one known open gap (Event 26) |
| 05 | [`logs/Phase 05 Network Visibility/`](./logs/Phase%2005%20Network%20Visibility/) | Switch SPAN session, NIC4 bridged into wazuh-host, Suricata install and Wazuh integration, the JSON decoder field-limit blocker, the discovery that the physical SPAN was blind to east-west traffic, the hypervisor tc mirroring fix and its live verification | Functionally complete, tc mirror persistence across reboot still open |
| 06 | [`logs/Phase 06 Attack Surface/`](./logs/Phase%2006%20Attack%20Surface/) | Kali moved to RANGE30 and isolation confirmed live | In progress: Kali isolation done; pfSense log forwarding and the WireGuard VPN not started |
| 07 | `logs/Phase 07 AD Expansion/` | AD/DC build (`dc01`), directory structure and RBAC, deliberate misconfigurations, GPO deployment and shares, Linux victim and web app, attacker tooling on Kali, the full kill chain, LAPS fix and retest, writeup | Not started |
| 08 | `logs/Phase 08 Hybrid Identity/` | Entra Connect sync, Exchange Online and phishing simulation, Conditional Access and cloud identity enumeration | Not started |
| 09 | `logs/Phase 09 AI Triage Layer/` | Governed Wazuh triage agent (`wazuh-triage-01`) | Not started |
| 10 | [`logs/Phase 10 Local AI and Routing/`](./logs/Phase%2010%20Local%20AI%20and%20Routing/) | `pve-ai` host setup and vfio-pci GPU passthrough binding, `ai-vm` build with the RTX 3070 passed through and verified via nvidia-smi | Step 1 (GPU passthrough) complete; Step 2 (Ollama, Open WebUI, n8n routing) not started |
| 11 | `logs/Phase 11 SOAR/` | Shuffle deployment, Wazuh webhook integration, first enrichment/notify playbook | Not started |
| 12 | `logs/Phase 12 Case Management/` | TheHive and Cortex | Deferred |

Empty phase folders (07, 08, 09, 11, 12) and empty step folders inside started phases (Phase 6 Steps 2 to 3, Phase 10 Step 2) are placeholders in the folder structure only, no files exist in them yet. They get their first file the day that work actually starts.

## On ordering

Files are numbered by phase, not strictly by calendar date. Phase 10's GPU passthrough work is dated 2026-07-12 and 2026-07-13, earlier than most other phases, because the `pve-ai` node was originally tracked as its own separate project before being merged into this one. That is expected, not an error.

See `LAB-BLUEPRINT.md` for the full historical mapping note and reasoning behind the 2026-09-06 renumbering.
