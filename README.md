# Home SOC Lab — Segmented Network, Detection Engineering, and AI-Assisted Triage

I took a flat, unmanaged home network and rebuilt it from scratch into something closer to how a real environment is actually run — segmented, firewalled, logged, and eventually watched by a SIEM with AI-assisted triage on top. This repo is the running record of that build: what I built, why, what broke, and how I fixed it.

## About me

I'm a systems administrator with a background in IAM and SOC operations. Early in my career I worked analyst-side, and what I took away from that was a clear sense of where I wanted to build my career going forward: administration and engineering. Understanding how systems are actually built, configured, and maintained gives me a level of depth I couldn't get staying purely on the analyst side, and it's the direction I've been deliberately growing in ever since. This lab is part of that — a way to keep building real, hands-on infrastructure skill on my own time, especially useful right now while I'm between roles after a layoff.

My background includes time at Arctic Wolf doing SOC triage, the U.S. Air Force, Wells Fargo, Medtronic, and Thomson Reuters, and most recently administering Entra ID, Intune, and Conditional Access policy for an environment of about 350 users. I hold CompTIA A+, Network+, Security+, CySA+, and PenTest+(expired) certifications.

## What this is

A physical home lab — a Dell PowerEdge R710, a dedicated GPU inference box, and a managed Cisco switch — rebuilt into a segmented network with an intentionally isolated attack range, a SIEM correlating host and network telemetry, and (in progress) SOAR automation and AI-assisted triage. Every step is documented as it actually happened, including the mistakes — a previous configuration attempt caused a full lockout requiring a rebuild, which is why this repo exists in the first place: a real, timestamped record of every change and how to undo it.

**This is not a tutorial clone.** Every rule, every detection, and every playbook here is either written by me or explicitly reviewed and finalized by me — AI assistance drafts references and explains options; it does not make the security decisions.

## Architecture, at a glance

```
Home network → pfSense (firewall) → Cisco switch (VLAN trunk)
                                          │
                    ┌─────────────────────┼─────────────────────┐
              Management VLAN       Infra VLAN              Range VLAN
              (admin access)   (Wazuh, Suricata,        (attacker + victim,
                                 Shuffle, AI stack)        fully isolated)
```

- **Range VLAN** — an attacker and a target machine, fenced by a default-deny firewall rule. Real attack techniques run here safely because nothing gets out except one explicit logging path.
- **Infra VLAN** — Wazuh (SIEM) correlates host-based telemetry (Sysmon) with network-based telemetry (Suricata, fed by a switch SPAN port) — two independent layers of detection for one incident.
- **Management VLAN** — where I actually administer everything.
- **AI layer** — a local GPU inference node (Ollama) handles routine triage and personal use for free; harder cases escalate to Claude or Grok on a threshold I set. Every AI agent with autonomous access is scoped, credentialed, and logged separately from my own actions — see `agent-registry.md`.

Full architecture detail, including the build phases and every hardware/network specific, is in [`LAB-BLUEPRINT.md`](./LAB-BLUEPRINT.md).

## Status

| Phase | What | Status |
|---|---|---|
| A | Network rebuild — VLANs, trunk, pfSense | ✅ Complete |
| A.5 | Range VLAN isolation firewall rule | 🔜 Next |
| B | Docker/Git substrate + Wazuh SIEM | Planned |
| C | Detection engineering (Atomic Red Team + custom rules) | Planned |
| C.5 | Network visibility (SPAN + Suricata) | Planned |
| D | AI triage layer + agent-identity governance | Planned |
| E | Local AI (Ollama) + n8n routing | Planned |
| F | SOAR automation (Shuffle) | Planned |
| G | Case management (TheHive + Cortex) | Deferred |

Live detail, including actual command output and what went wrong, is in [`build_log.md`](./build_log.md).

## Skills this demonstrates

- **Network segmentation & firewall policy** — VLAN design, 802.1Q trunking, default-deny rule writing
- **Detection engineering** — custom Wazuh rules written and tuned against real attack simulations (MITRE ATT&CK-mapped)
- **Multi-layer correlation** — host-based (Sysmon/agent) and network-based (Suricata/SPAN) detection tied to one incident
- **SOAR / automation** — Shuffle playbooks taking alert-to-enriched-case from ~40 minutes manual to seconds automated
- **AI agent governance** — least-privilege scoping, dedicated credentials, and separated audit trails for every autonomous AI agent in the stack
- **Infrastructure-as-code discipline** — rules, playbooks, and configs are version-controlled, not just clicked into a GUI and forgotten
- **Incident documentation** — every investigation write-up follows a consistent format: what happened, what I saw, my verdict, and why

## Repo structure

```
LAB-BLUEPRINT.md      — what's being built, in what order, with commands
build_log.md          — the running, append-only history: real output, real decisions
agent-registry.md      — every AI agent's scope, credential, and owner
investigations/        — individual incident write-ups (the analytical portfolio)
Docx Files/            — printable workbook versions of the above (optional reading)
```

## Tech stack

`Proxmox VE` · `pfSense` · `Cisco IOS` · `Wazuh` · `Suricata` · `Shuffle (SOAR)` · `Docker` · `Ollama` · `Git`

---

*This lab is actively under construction. Check `build_log.md` for the most current status — it's updated every session, not retroactively.*
