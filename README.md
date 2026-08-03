# Home SOC Lab

A home-built Security Operations Center lab — network segmentation, detection
engineering, SOAR automation, and AI-assisted triage, built from scratch on
retired enterprise hardware. Built as a hands-on portfolio piece and working
lab, in support of a transition from Technology Systems Administrator II into
a SOC Analyst / IAM Analyst role.

**Author:** Michael Mathews — U.S. Air Force veteran, IT/security professional
(8+ years), CompTIA A+ / Network+ / Security+ / CySA+ / PenTest+, BS in IT
Networking and Security.

> **Note:** This repo is intentionally framed to present a security-focused
> engineering trajectory — the goal is a working SOC stack that demonstrates
> real detection engineering, automation, and governed AI-assisted triage,
> not just administration.

---

## Architecture

```
Home Router (192.168.0.1)
        |
   [dumb switch] -- Management PC (dedicated USB-Ethernet link)
        |
     pfSense VM (on pve01)
        |
   Cisco Catalyst Switch (802.1Q trunk)
   +--------+----------+-------------+
 MGMT10   INFRA20    RANGE30      SPAN monitor
 (mgmt)   (Docker/    (Kali +      (mirrors Range
          Wazuh/       Win11-       traffic -> Suricata,
          Suricata/     victim)      passive/no IP)
          Shuffle/n8n,
          + pve-ai inference)
```

**Two layers of detection**, as in a real SOC:
- **Host-based** — Sysmon on the victim VM -> Wazuh agent -> Wazuh
  (crosses Range->Infra on TCP 1514 only)
- **Network-based** — SPAN port mirrors Range VLAN traffic -> Suricata
  (passive) -> alerts into Wazuh

[Link To Lab Diagrams](https://securityjonesing.github.io/SOC-LAB/Diagrams/SOC-Lab-Diagrams.html)

## Hardware

| Node | Spec | Role |
|---|---|---|
| `pve01` | Dell PowerEdge R710, Xeon X5675, 64GB RAM | Primary Proxmox host — pfSense, Wazuh, Suricata, Shuffle, n8n, Range VMs, service UIs |
| `pve-ai` | i9-10900KF, RTX 3070 (8GB) | GPU inference only (Ollama) — separate project, see below |
| Cisco Catalyst WS-C2960X-48FPS-L | IOS 15.2(7)E9 | VLAN trunk + management access port |
| QNAP TS-869 Pro | — | Excluded from the lab as storage; future Proxmox QDevice candidate only |

## VLAN design

| VLAN | Purpose | Access |
|---|---|---|
| **MGMT10** | Management PC | Full access to all VLANs for admin (Proxmox, Wazuh, Shuffle, n8n, pfSense, iDRAC) |
| **INFRA20** | Docker/Wazuh host, `pve-ai` | Receives Range logs on Wazuh's port; scoped outbound internet (DNS/HTTP/HTTPS/ICMP) |
| **RANGE30** | Kali, Win11 victim | Default-deny; single explicit allow to Infra's Wazuh ingest port only |

## Stack

- **Firewall/Routing:** pfSense CE 2.8.1
- **Detection:** Wazuh (host-based) + Suricata (network-based, via SPAN)
- **SOAR:** Shuffle — SOC incident response automation
- **AI orchestration:** n8n — local-vs-cloud routing (`triage-router-01`),
  general automation
- **AI inference:** Ollama on `pve-ai` (RTX 3070), local-first; Claude Code
  (`claude -p`, subscription-covered) or Grok as cloud escalation on a
  threshold
- **Chat UI:** Open WebUI, points at Ollama with Claude/Grok selectable
- **Case management (planned, deferred):** TheHive + Cortex

## Phase status

| Phase | Status | Notes |
|---|---|---|
| A — Network Rebuild | ✅ Complete | VLANs, trunk, mgmt access port, pfSense VM + interfaces. Snapshot `pfsense-clean-install`. |
| A.5 — Isolation Rule | ✅ Complete | Range default-deny + explicit Wazuh-port allow, applied and reviewed. |
| **B — Docker/Git/Wazuh Substrate** | 🔄 In progress | **Steps 1-4 done:** Ubuntu Server 24.04.4 LTS VM (`wazuh-host`, `10.10.20.100`) built, SSH confirmed; host prepped for Wazuh's indexer; Docker CE + Compose installed; Git configured with a dedicated SSH deploy key, repo cloned to `~/soc-lab`. INFRA20 now has 5 outbound rules (DNS/HTTP/HTTPS/ICMP/SSH), each added and live-tested as real work hit each gap. **Step 5 next:** deploying Wazuh via Compose. |
| C — Detection Engineering | ⏳ Not started | Atomic Red Team + custom Wazuh rules |
| C.5 — Network Visibility | ⏳ Not started | SPAN + Suricata |
| D — AI Triage Layer | ⏳ Not started | Governed Wazuh triage agent (`wazuh-triage-01`) |
| E — Local AI + n8n Routing | ⏳ Not started | Integrates with the separate `ai-node` project |
| F — SOAR (Shuffle) | ⏳ Not started | Slots in after Phase C |
| G — Case Management (TheHive + Cortex) | ⏸ Deferred | Requires Cassandra + Elasticsearch; attempted after F and E are stable |

## Documentation

- [`LAB-BLUEPRINT.md`](./LAB-BLUEPRINT.md) — what's being built and in what order
- [`build_log.md`](./build_log.md) — running, append-only record of every change and its actual output
- [`agent-registry.md`](./agent-registry.md) — scope/owner/lifecycle for every AI agent in the lab (in progress)
- [`PROJECT-INSTRUCTIONS.md`](./PROJECT-INSTRUCTIONS.md) — environment reference used to drive Claude sessions

## Related, separate project

`pve-ai` / `ai-vm` (Ollama + Open WebUI base) is built and tracked under its
own plan — this repo's Phase E only *integrates* with it and adds the n8n
routing layer; it does not rebuild the Ollama install.
