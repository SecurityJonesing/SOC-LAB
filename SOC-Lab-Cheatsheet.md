# SOC Lab, Cheat Sheet & Checklist

*Two halves: the checklist (what to do, in order) and the lookup (what's the command/IP/path again). Ctrl+F is my friend.*

# Part 1, The Checklist

## Before I start ANY session

- [ ] Turn on session recording, pick the one that matches where I'm working (see Part 2 → Session recording).
- [ ] Open `build_log.md` (it's now a short index, the real narrative lives under `Build Logs/`), I'm logging as I go, not reconstructing later.
- [ ] Know my recovery path before I need it, see the Recovery Ladder below. Confirm it works, don't assume.
- [ ] One change at a time. Decide right now what the single next change is. Not three.

## Risk tiers, how much prep before I touch it

| Tier | Examples | What's required first |
| --- | --- | --- |
| LOW | Install a package, read a config, `ip addr show`, check a dashboard | Just log it |
| MEDIUM | VM config change, Docker deploy, editing a detection rule, a Shuffle playbook | Snapshot the VM first |
| HIGH | pfSense rules, VLAN/trunk config, switch config, bridge/NIC changes | Console access confirmed + VM snapshot + config backup + session recording ON |

**Everything network-related is HIGH. No exceptions. That's the category that cost me a full Proxmox reinstall.**

## During the work

- [ ] Understand before running. If Claude Code proposes something I don't follow, ask.
- [ ] Compare actual output to expected output. Not "it didn't error", does it say what it should say?
- [ ] If output doesn't match: STOP. Do not stack the next change on top of an unverified one.
- [ ] Log it, command, actual output, decision, why.

## After each phase

- [ ] Run the phase's acceptance check from `LAB-BLUEPRINT.md`. Not "it seems fine", the actual check.
- [ ] Explain it back. Can I say what I built and why, out loud?
- [ ] Commit to Git, rules, configs, notes.
- [ ] Update the relevant file under `Build Logs/` with the phase outcome and any rollback point created.
- [ ] Stop the transcript.

## The Recovery Ladder, when something breaks

*Work down the list until something works. Know this cold before I need it.*

| If I lose… | Fall back to… |
| --- | --- |
| Proxmox web UI (192.168.0.201:8006) | SSH to pve01 |
| SSH to pve01 | iDRAC at 192.168.0.100 → or monitor + keyboard on the server |
| iDRAC web console (Java .jnlp pain) | Monitor + keyboard directly on the R710, always works, skip the Java fight |
| pfSense web UI | Proxmox console into the pfSense VM → or roll back the VM snapshot |
| Cisco switch access | PuTTY over USB-serial console, physical, can't be locked out |
| A VM's network | Check its bridge assignment in the Hardware tab |
| Shuffle / n8n / Open WebUI unreachable | Check the service container: `docker compose ps` on its host; confirm the host VM itself is up in Proxmox first |
| Everything, total mess | Restore VM snapshot / restore pfSense XML config backup |

**The rule this table encodes: never make a change that could cut off my only access path.**

# Part 2, The Lookup

## Addresses & access

| What | Where | Notes |
| --- | --- | --- |
| pve01 Proxmox web UI | 192.168.0.201:8006 | Management on NIC1 / vmbr1 |
| pve01 iDRAC | 192.168.0.100 | DHCP reservation on home router. User: root |
| pve-ai Proxmox host | 192.168.0.202 | Separate node, GPU inference only |
| ai-vm (Ubuntu on pve-ai) | 192.168.0.203 | Ollama target. Port 11434. |
| Home router | 192.168.0.1 | DHCP/routing for everything today |
| pfSense GUI (Management) | https://10.10.10.1 | VLAN 10, MGMT subnet |
| pfSense GUI (Infra) | https://10.10.20.1 | VLAN 20, INFRA subnet |
| pfSense GUI (Range) | https://10.10.30.1 | VLAN 30, RANGE subnet |
| Wazuh dashboard | https://\<wazuh-host-ip\> | Port 443, on the Ubuntu SOC host (Infra) |
| Suricata (Phase 5) | runs on wazuh-host, bound to ens19 | No web UI, alerts flow through eve.json into Wazuh |
| Ollama API | http://192.168.0.203:11434 | /api/tags lists models; /api/generate for a prompt |
| Open WebUI | http://\<pve01-host-ip\>:3000 | Personal chat UI, points at Ollama; Claude/Grok selectable |
| n8n | http://\<pve01-host-ip\>:5678 | Router workflows (triage-router-01) + general automation |
| Shuffle | https://\<shuffle-vm-ip\> | Dedicated VM on Infra, SOAR playbooks, receives Wazuh webhook |

## SSH

```
# KexAlgorithms flag is REQUIRED, Windows OpenSSH 9.5p2 vs Proxmox 9 mismatch
ssh -o "KexAlgorithms=curve25519-sha256" root@192.168.0.201
```

- Reuse the existing `id_ed25519` key, don't generate a new one.
- `.ssh/config` needs correct `icacls` permissions on Windows or the key is silently rejected.

## The workflow

**I run the command. I paste the output into chat. Claude reads it, tells me if it worked and why, and gives me the next step.**

- Claude CANNOT see my terminal. What I paste is all it gets.
- Claude in Chrome CAN see browser tabs: Proxmox, pfSense, Wazuh, Shuffle, n8n, Open WebUI dashboards. Use it for "where is that button", not for clicking things on networking screens.
- Claude in Chrome CANNOT see: PuTTY, BIOS, iDRAC console, VM consoles. Copy and paste for those.
- Project instructions carry my hardware, IPs and gotchas into every new chat, paste `PROJECT-INSTRUCTIONS.md`'s contents in once.
- Core security logic (firewall rules, detection rules, Shuffle playbooks, n8n routing thresholds), Claude explains and drafts, I write the final.

## Session recording

**PuTTY (Cisco switch), set BEFORE opening the session:**

Session → Logging → "All session output" → log file: `C:\Users\micha\SOC-Lab 7-15-2026\Build-Transcripts 7-16-2026\Session Logging\switch-&Y&M&D-&T.log`

**PowerShell 7 (Windows):**

```powershell
Start-Transcript -Path "C:\Users\micha\SOC-Lab 7-15-2026\Build-Transcripts 7-16-2026\Session Logging\session-$(Get-Date -Format 'yyyy-MM-dd-HHmm').txt"
# ...work...
Stop-Transcript
```

**Linux (pve01, Kali, Ubuntu VM):**

```bash
script ~/session.log      # start recording
script -a ~/session.log   # ...or append to existing
exit                       # stop recording
```

## Proxmox, network commands (the ones that saved me)

```bash
ip addr show vmbr1        # does the bridge have an IP?
ip link show              # all interfaces + UP/DOWN + NO-CARRIER
ip link show eno1         # one specific interface
brctl show vmbr1          # which physical NIC is in this bridge?
bridge link                # ...if brctl isn't installed
ethtool eno1               # "Link detected: yes/no", cable really live?
ip route show              # confirm a default gateway exists
ip addr flush dev vmbr0    # nuke an IP off a bridge immediately
nano /etc/network/interfaces  # the actual config file
ifreload -a                 # apply changes live (Proxmox = ifupdown2)
systemctl restart networking  # fallback if ifreload unavailable
```

### The bridge/network gotchas I already hit

- Two bridges claiming the same IP = silent ARP failure, even if one has no cable. Always check both.
- Moving a cable/bridge orphans any VM still pointed at the old one. Check every VM's Hardware tab after.
- A NIC showing no link may just be administratively down, not dead hardware, check `ip link show` / `ip link set <iface> up` before chasing cables, BIOS, or iDRAC logs.
- A physical switch SPAN can be blind to VM-to-VM traffic if the VMs share a Linux bridge on the same hypervisor (hit in Phase 5), the bridge switches locally in software and never sends the frames to the physical switch. Fix at the hypervisor layer with `tc mirred` mirrors on the relevant tap interfaces if this happens again.

## Console access paths

| Target | How |
| --- | --- |
| pve01 BIOS | F2 during POST |
| pve01 iDRAC6 config | Ctrl+E during POST, separate from F2, easy to miss (5-second window) |
| pve01 local console | Monitor into VGA + USB keyboard, back of the server. No Java, no fuss. Preferred. |
| Cisco switch | PuTTY → USB-serial console |
| VM consoles | Proxmox web UI → VM → Console (inline, no popout needed) |

*R710 is legacy BIOS only, it predates UEFI on PowerEdge. Don't go looking for UEFI options.*

## Git (Phase 3 onward)

```bash
git status              # what's changed?
git add .               # stage everything
git commit -m "message" # snapshot it
git push                 # send to GitHub
git log --oneline        # history, compact
```

## Docker (Phase 3 onward)

```bash
docker ps                     # what's running?
docker ps -a                  # ...including stopped
docker compose up -d          # start the stack, detached
docker compose down           # stop the stack
docker logs <container>       # why is this thing unhappy?
docker compose logs -f        # follow logs live
```

*Note: `docker compose` (v2, space) vs `docker-compose` (v1, hyphen), check which my install uses.*

## Suricata / SPAN visibility (Phase 5)

```bash
lsusb                    # what chipset is a USB adapter (historical, superseded by tc mirroring)
ip link show             # did a new enx* interface appear?
bridge vlan show         # confirm VLAN tags on bridge ports/taps
tcpdump -i ens19 -n -e   # is anything actually arriving on the mirrored interface?
```

*The USB-Ethernet adapter path was superseded, Phase 5 ended up using NIC4/`vmbr4` plus hypervisor `tc mirred` mirroring instead, since the physical switch SPAN proved blind to east-west VM-to-VM traffic on the same bridge.*

## AI stack (Phase 10)

```bash
curl http://192.168.0.203:11434/api/tags     # is Ollama alive? lists models
curl http://192.168.0.203:11434/api/generate -d '{
  "model": "<your model>",
  "prompt": "say ok",
  "stream": false
}'   # -> expect a JSON response with the model's reply
```

### Claude Code headless (subscription, no API billing)

```bash
claude -p "your prompt here"                                    # basic headless call
claude -p "..." --output-format json                            # structured, has total_cost_usd
claude -p "..." --permission-mode bypassPermissions --max-turns 3   # unattended, guardrailed
```

**WATCH THIS:** Headless Claude Code draws from my subscription's usage limits, same pool as interactive use. `--max-turns` is a safety guardrail so a background loop doesn't burn my daily quota in one go.

## SOAR stack (Phase 11)

```bash
docker compose ps   # Shuffle containers up?
```

Wazuh → Shuffle webhook, added to `ossec.conf`:

```xml
<integration>
  <name>shuffle</name>
  <hook_url>https://<shuffle-host>/api/v1/hooks/<id></hook_url>
  <rule_id>100001</rule_id>
  <alert_format>json</alert_format>
</integration>
```

```bash
docker compose restart wazuh.manager
```

## Snapshots & backups

| What | How |
| --- | --- |
| VM snapshot | Proxmox web UI → VM → Snapshots → Take Snapshot. Near-instant, do it before anything HIGH risk. |
| pfSense config | Built-in XML config export. Do this before every rule change. |
| Switch config | You're starting from `write erase`, after config: `copy running-config startup-config` |

## Files & where they live

```
C:\Users\micha\SOC-Lab 7-15-2026\Build-Transcripts 7-16-2026\   <- project folder (git repo root)
├── PROJECT-INSTRUCTIONS.md      <- paste into Project instructions
├── LAB-BLUEPRINT.md             <- shared reference: what + what order (includes merged ai-node/pve-ai build, Phase 10)
├── SOC-Lab-Cheatsheet.md        <- this file
├── build_log.md                 <- short index into Build Logs/, created Phase 1, maintained throughout
├── Build Logs\                  <- the actual build narrative, one numbered folder per phase (01 through 12)
│   ├── Phase 01 Network Rebuild\
│   ├── Phase 02 Isolation Rule\
│   ├── Phase 03 Wazuh Substrate\
│   ├── Phase 04 Detection Engineering\
│   ├── Phase 05 Network Visibility\
│   ├── Phase 06 Attack Surface\
│   ├── Phase 07 AD Expansion\        <- not started, no files yet
│   ├── Phase 08 Hybrid Identity\     <- not started, no files yet
│   ├── Phase 09 AI Triage Layer\     <- not started, no files yet
│   ├── Phase 10 Local AI and Routing\
│   ├── Phase 11 SOAR\                <- not started, no files yet
│   └── Phase 12 Case Management\     <- not started, no files yet
├── agent-registry.md            <- created Phase 9, wazuh-triage-01 + triage-router-01
├── investigations\               <- my writeups = the portfolio
└── Session Logging\              <- raw session transcripts (PowerShell `Start-Transcript`, PuTTY switch logs) and pfSense config exports, lives IN this repo folder, declared in .gitignore as local-only
```

*Each phase folder splits into Step subfolders when the phase has more than one step or is still in progress; a complete phase with a single entry gets one flat file directly in the phase folder instead (Phase 2 is the current example of that).*

## The one-line version

**Log it, snapshot it, confirm my fallback, change one thing, verify it, explain it back.**
