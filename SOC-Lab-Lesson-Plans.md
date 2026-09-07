# SOC Lab — Lesson Plans

What I read. Two brains: the engineer who builds it, the analyst who runs it.

## Which document do I actually open?

Short answer: this one, and the printed checklists. Nothing else.

| File | Who reads it | When |
|---|---|---|
| `LAB-BLUEPRINT.md` | Claude Code | Never open it. It's the spec Claude Code builds from. |
| `PROJECT-INSTRUCTIONS.md` | Claude Code | Never open it. Auto-loads every session. |
| Phase Checklists (.md) | ME, printed | Beside the keyboard, every build session. |
| Cheatsheet (Word) | ME, printed | When I need a command or an IP. |
| This document | ME | Before/during each phase, and once things are live. |
| `agent-registry.md` | ME + Claude Code | Phase 9 onward. Keep it true. |
| `build_log.md` | Claude Code writes | Read it when something broke and I need to know what changed. |

I never need VS Code for this project. The `.md` files exist for Claude Code to read, the Word docs exist for me.

## How to use this document

- Read a module when its phase is about to start, or right after. Don't binge ahead, concepts land better attached to something I just watched happen.
- Lesson Plan 1 (engineer) explains what got built and why. Lesson Plan 2 (analyst) starts once something is running.
- Every module ends with a checkpoint. The bar is: say it out loud, cold, without notes. That's the only honest test.
- Diagrams throughout. If a diagram makes a module unnecessary, the diagram wins, look at it and move on.

### Why the checkpoints matter more than the lab

Claude Code can build this whole thing while I watch. I'd end up with a working lab and nothing to say about it. The checkpoints are the difference between "I built a SOC lab" (a claim anyone can make) and being able to explain, cold, why a SPAN port has no IP address. One of those gets me hired.

## The Big Picture

Before any module, this is what I'm building and why it hangs together.

Phases 1 through 5, plus 9 and 10, are covered in this document (the phases built and taught so far). Red = lockout risk. The two riskiest phases teach the most.

The lab network. Every packet between VLANs crosses pfSense, that's the whole design.

Two ideas carry the entire build:

- The switch decides which VLAN a port belongs to. pfSense decides whether traffic may cross between VLANs. Those are different jobs, that's why I need both.
- The Range VLAN is fenced so I can attack it on purpose. Everything downstream, running real attack code, disabling Defender, generating real malicious traffic, is only safe because that fence exists.

## Lesson Plan 1 — The Builder

What got built, and why. Read alongside each phase.

### Module 0 — Before Phase 1: bridges, and why Kali went dark

I already lived this one, which makes it the best place to start.

Proxmox turns one physical server into many virtual ones. The part that trips everyone: a VM's network card isn't plugged into anything physical. It's plugged into a bridge, a software switch inside Proxmox. The bridge is then tied to a real NIC.

The bridge is the wall socket. The NIC is the wire behind the wall. The VM is the lamp.

Move the wire behind the wall to a different socket, and the lamp plugged into the old socket goes dark, even though nothing about the lamp changed. That is exactly what happened to Kali when I moved the cable from NIC2 to NIC1: Proxmox's own management followed to `vmbr1`, but Kali was still plugged into `vmbr0`, which now had no uplink.

**The two gotchas this explains:**
1. Two bridges can claim the same IP simultaneously, the kernel can't decide which should answer ARP, so pings silently fail even though everything "looks" configured.
2. Moving a cable or bridge orphans every VM still pointed at the old one, with no error message. Check every VM's Hardware tab after any bridge change.

**Checkpoint, say it out loud. Can't? Re-read this module.**
In my own words: why did Kali lose network after the NIC1/NIC2 cable swap? Use the socket idea.

### Module 1 — Phase 1: VLANs, and why pfSense sits in the middle

A VLAN makes one physical switch behave like several separate switches. Devices on different VLANs can't talk to each other by default, even plugged into the same box, the switch tags traffic with a VLAN ID and keeps it apart.

Why this matters here: Kali is about to attack a Windows VM on purpose. Without VLANs, that traffic shares a broadcast domain with my management PC and my home network. VLANs are the fence.

**Trunk vs access port**
- An access port belongs to exactly one VLAN. My PC's lab port is an access port on VLAN 10, full stop.
- A trunk port carries multiple VLANs at once, each packet tagged. NIC3 is a trunk because it has to carry Management, Infra and Range to pfSense over one wire.

```
# the two commands that tell me if it actually worked:
show vlan brief          # do my VLANs exist?
show interfaces trunk    # is the trunk actually trunking?
```

**Why pfSense specifically**
VLANs only control which broadcast domain traffic lives in. They don't decide whether traffic may cross between domains. That's a firewall's job. pfSense has a leg in each VLAN and enforces what's allowed across.

**Checkpoint, say it out loud. Can't? Re-read this module.**
What's the difference between a VLAN and a firewall rule, what does each actually control, and why do I need both?

### Module 2 — Phase 2: default-deny

Two philosophies:
- Default-allow, block known-bad: everything passes unless specifically blocked. Fast to set up, fragile, I only block what I thought of.
- Default-deny, allow known-good: nothing passes unless explicitly allowed. Slower, but an attacker (or a misbehaving test) can't reach anywhere I didn't name.

The Range VLAN rule is default-deny: block everything, then poke exactly one hole, to Wazuh's log port. I'm not trying to anticipate every bad outcome. I'm only trusting what I explicitly named.

```
# pfSense rules evaluate TOP-DOWN. First match wins. Order is everything.
Rule 1:  PASS   RANGE net -> <wazuh IP>:1514      # the one hole
Rule 2:  BLOCK  RANGE net -> any                  # everything else dies
```

**If block is above pass**
Nothing gets through, ever. And if I write a Block rule on the interface I manage pfSense from, I lock myself out, that is the single most common self-inflicted outage in this whole build. Read the rule twice before applying.

**Checkpoint, say it out loud. Can't? Re-read this module.**
If Atomic Red Team tried to reach an external IP mid-simulation, what happens under this rule, and why?

### Module 3 — Phase 3: containers vs VMs

A VM virtualises an entire computer, kernel, drivers, the lot. That's why VMs are heavy: each one boots a full OS. A container shares the host's kernel and packages only the application plus what it directly needs.

That's why a container starts in seconds and a VM takes minutes: a container isn't booting an operating system, it's launching a process in an isolated pocket of one that's already running.

So Wazuh's manager, indexer and dashboard run as three containers on one Ubuntu VM, not three VMs. They're three isolated processes, not three computers.

```
# Docker Compose = a text file describing the desired end state
docker compose up -d       # build it all, detached
docker compose ps          # what's actually running?
docker compose logs -f     # why is it unhappy?
```

**The volume mount that matters**
Wazuh's rules live inside the manager container by default, which means a rebuild wipes them, and I can't commit them to Git. Mounting `./config/rules/local_rules.xml` from the host is what makes my detections survive rebuilds and become a portfolio artifact. That one line in `docker-compose.yml` is the difference between a lab and a portfolio.

**Checkpoint, say it out loud. Can't? Re-read this module.**
Why does Wazuh's manager/indexer/dashboard make more sense as three containers on one VM than three separate VMs?

### Module 4 — Phase 3: what Git is actually for

Git tracks changes over time, not just copies. Every commit is a snapshot with a message explaining what changed and why.

The value isn't recovery. It's history I can point at in an interview: "here's the detection rule I wrote, here's why I changed it two days later when I found it was too noisy." That sentence is worth more than any bullet on a resume, because it shows judgment over time rather than a one-off.

```
git status                    # what's changed?
git add config/rules/local_rules.xml
git commit -m "detect: T1059.001 - tighten to reduce FP on admin scripts"
git push
git log --oneline             # my story, compressed
```

Read that commit message again. It says what, why, and what I learned. That's the artifact.

**Before my first push**
Is there anything in these files I'd rather not publish? Credentials, keys, internal IPs? Check now. Git history is permanent, and this repo is my portfolio, which means strangers will read it.

**Checkpoint, say it out loud. Can't? Re-read this module.**
Why is "here's my Git history, the rule, the false positive I found, the fix" a stronger interview story than "I wrote a detection rule"?

### Module 5 — Phase 4: what a SIEM actually does

Wazuh does three things: collects logs from many sources, normalises them into a common format so different log types can be compared, and matches them against rules to generate alerts.

A log is data. An alert is a log that matched a rule. The rule is the only thing in between.

Atomic Red Team exists so I can generate a known attack pattern on demand and confirm my rule actually catches it. Attacking my own lab isn't the point, proving my detection works is.

```
# on the Win11 victim (ART runs ON the box I want telemetry from):
Invoke-AtomicTest T1059.001 -ShowDetailsBrief   # read before I run
Invoke-AtomicTest T1059.001 -GetPrereqs
Invoke-AtomicTest T1059.001
Invoke-AtomicTest T1059.001 -Cleanup            # always
```

**Defender will quarantine the atomics folder**
That's Defender working correctly, it's real attack tooling. I'll add an exclusion for `C:\AtomicRedTeam`, which is only acceptable because that VM is fenced on the Range VLAN. Phase 2 is what makes Phase 4 safe. If I ever find myself disabling AV on a box that isn't fenced, stop.

**Checkpoint, say it out loud. Can't? Re-read this module.**
What's the difference between a log and an alert, and what has to happen for one to become the other?

### Module 6 — Phase 5: two kinds of visibility

Sysmon tells me what happened on a machine, a process ran, a registry key changed. Suricata, fed by a mirrored (SPAN) port, tells me what happened on the wire, connections, beaconing, tunnelling, regardless of whether the endpoint is logging anything at all, or even has an agent.

**Why mirror instead of just plugging Suricata into the network**
A SPAN destination port only receives a copy of traffic. It doesn't participate in the network otherwise. That's why it gets no IP address, it's a passive tap, not a network citizen. This is exactly how enterprise SOCs get network visibility without an agent on every device.

```
monitor session 1 source vlan 30
monitor session 1 destination interface <port>
show monitor session 1
```

```
# on the Wazuh host, the tap gets NO IP:
sudo ip link set <iface> up
sudo ip link set <iface> promisc on
ip addr show <iface>        # MUST show no inet address
```

**The skill this builds**
Correlating both layers. "Sysmon says this process ran" AND "Suricata says that host then called out to a strange IP" is a far stronger signal than either alone. Recognising that two alerts are one story is what separates an analyst who reads alerts from one who builds an incident picture.

**The lesson that almost didn't make it into this module, and matters more than the SPAN config itself**

The switch SPAN, as configured above, looked correct and still failed to see the traffic that mattered. Kali and the Win11 victim both live on `pve01`, attached to the same Linux bridge (`vmbr3`). A Linux bridge switches frames between VMs on the same bridge entirely in software, in RAM, on the hypervisor. That traffic never travels down the wire to the physical Cisco switch at all, so a SPAN session configured on the switch has nothing to mirror. It faithfully mirrored ARP and broadcast noise, and mirrored nothing of the actual attack traffic, with no error anywhere to say so.

The fix had to move to where the traffic actually was: `tc mirred` ingress mirrors applied directly on the hypervisor's tap interfaces (`tap100i0` for Kali, `tap101i0` for the victim), redirecting a copy of their traffic to the tap feeding Suricata. Verified by capturing a real SYN/SYN-ACK exchange arriving on the Suricata-facing interface. The physical switch SPAN is still correct and still useful, it's exactly right for north-south traffic, traffic that actually leaves the host. It was never going to see traffic that stays inside one hypervisor's own virtual switch.

This is the real answer to "what went wrong, how did I fix it" for this phase, better than anything above it in this module, because the failure was silent, plausible-looking, and only found by testing the assumption rather than trusting the config.

**Checkpoint, say it out loud. Can't? Re-read this module.**

1. If an attacker compromised a device with no Sysmon and no agent at all, how would Suricata still have a chance of catching it?
2. Why did a correctly configured switch SPAN session still fail to see traffic between two VMs on the same hypervisor, and where did the fix actually have to happen?

### Module 7 — Phase 9: what AI triage actually means

An AI reading a Wazuh alert and producing a plain-English verdict is not the same as the AI deciding anything. I am the analyst of record. The AI's output is a first draft of judgment, not a replacement for it.

This mirrors exactly where the SOC analyst job is going: reviewing and confirming machine-generated verdicts rather than reading every raw log by hand. Tier 1 is becoming a verdict queue. The person who can supervise the machine, and catch it when it's wrong, is the one who stays employed.

**Checkpoint, say it out loud. Can't? Re-read this module.**
Why does "the AI flagged it as malicious" need a human verdict logged next to it, rather than standing alone as the final answer?

### Module 7.5 — Phase 9: governing the agent as an identity

I do this for humans every day. An Entra ID account gets specific permissions, not everything, just what the job needs, and a log of what it did. Apply the same discipline to the triage agent instead of a human, and I get agent-identity governance.

The agent has its own credential, its own scope, and its own line in the audit log.

**The four pieces, mapped to what I already know**
- Its own credential, never mine. Same idea as a service account: a non-human identity with its own name and access.
- Least privilege, it can read alerts and write verdicts, nothing broader. The same RBAC principle I already apply at work.
- Its own audit trail, every agent action logs as the agent, never blended with my actions. Test it: read one log line. Can I tell who acted?
- A registry entry, what it is, what it may touch, who owns it, when the credential rotates, whether it's still active. An identity governance platform in miniature.

**A test that can actually fail**
Get a token as the agent user and try to DELETE an agent via the Wazuh API. It should return 403 Forbidden. If it succeeds, my scope is wrong. A test that can't fail proves nothing, this one can.

**Checkpoint, say it out loud. Can't? Re-read this module.**
In terms of least privilege and audit trails, why would the agent using my personal Claude login be a governance gap, not just a style preference?

### Module 8 — Phase 10: local vs cloud, and why route between them

A local model on the RTX 3070 costs electricity, not per-request fees, and keeps data on my network, good for routine, high-volume triage. A larger cloud model costs more per call but reasons better about ambiguous or novel situations.

The router's job is deciding which alerts are routine enough for the local model and which deserve the better model's attention. That "escalate only when needed" pattern is itself an architecture decision worth explaining in an interview, it shows I think about cost and capability, not just whether something works.

```
curl http://192.168.0.203:11434/api/tags     # is Ollama alive?
```

**The router is a second agent**
It makes autonomous decisions, which alerts escalate. That's an agent, not plumbing. It gets its own entry in `agent-registry.md`: scope, owner, credential, lifecycle, review date. Same treatment as the first one.

**Checkpoint, say it out loud. Can't? Re-read this module.**
Give one example of an alert I'd trust the local model to triage alone, and one I'd want escalated, and explain what makes the difference.

## Lesson Plan 2 — The Operator

Starts once something is running. This is where I practise being the analyst.

**The writeup is the point**
Every module here ends with a short investigation writeup committed to the repo. Over weeks those become a running log of real analytical work, exactly what a hiring manager wants linked from a resume, and exactly what "I built a home lab" alone can't demonstrate. The lab proves I can build. The writeups prove I can think.

**Writeup format, every time. Half a page is plenty.**

```
What happened   — plain language
What I saw      — the actual alert / log evidence
My verdict      — malicious / benign / needs more info, and WHY
What I'd do next— contain / escalate / tune the rule / ignore
```

`investigations/2026-07-15-atomic-t1059.md`

### Module 1 — Reading raw logs before I trust an alert (after Phase 3)

Before touching the dashboard, look at a few raw Sysmon events directly. Get comfortable with what a process-creation event actually contains: parent process, command line, user context, timestamp.

This matters because an alert is somebody else's interpretation of a log. I want to be able to check that interpretation, not just trust it.

Exercise: find one boring process-creation event and one that looks unusual, even if nothing flagged it. One sentence on what made the second one stand out.

No writeup. This is a warm-up.

### Module 2 — My first triage (after Phase 3 acceptance)

Wazuh will generate baseline alerts from normal Windows activity. That's real life, not a flaw. Most alerts in any SOC are benign noise, and telling noise from signal is the actual Tier 1 skill.

Walk through: what triggered it, what the underlying log looks like, whether the activity makes sense given what I know is running, and whether I'd escalate, dismiss, or need more data.

Writeup: one real alert Wazuh generated on its own. Full format.

**A modern addition worth practising here:** not every alert that looks like an attack technique is one. Wazuh's own agent periodically runs commands like `net user` for its own inventory collection, and a detection rule written to catch that exact command pattern will happily fire on its own monitoring tool. Check the parent process before I trust the verdict, "what process actually launched this" is often the fastest way to tell a real technique from routine tooling.

### Module 3 — Catching a known attack (after Phase 4)

This is the heart of detection engineering: run a known technique, see if my rule catches it, and if it doesn't, work out why not. Attack, observe, tune, re-attack. That loop is what "writing detections" means day to day, far more than writing a rule from scratch.

Exercise: run the same atomic twice, once before my custom rule exists, once after. Compare what Wazuh shows.

Writeup: the technique (name the ATT&CK ID), what my rule looks for, and the before/after difference.

### Module 4 — Correlating host and network (after Phase 5)

Once Suricata is live I'll sometimes get two alerts for one incident, one host-based, one network-based. Recognising "these are the same story" is what separates an analyst who reads alerts in isolation from one who builds an incident picture.

```
# from Kali, a real network attack, not an atomic:
nmap -sS -p- <win11-victim-ip>
```

Writeup: both alerts side by side, and why they're one incident rather than two.

### Module 5 — Reviewing an AI verdict, not accepting it (after Phase 9)

Practise deliberately: read the AI's suggested verdict, then form my own opinion before reading its full reasoning, then compare. Sometimes I'll agree. Sometimes I'll catch something it missed, or it catches something I missed. Both are useful data about where the tool is strong.

**If I never disagree, something is wrong**
Find at least one alert where my read differs from the AI's, even slightly. If I can't after several, that's worth noting too, either the tool is well-calibrated for that alert type, or I'm anchoring on its answer instead of forming my own first. Be honest about which.

Writeup: the alert, the AI's verdict, my independent verdict, and, if they differed, which was actually right and why.

### Module 5.5 — Auditing the agent, not just the alerts (after Phase 9)

Module 5 reviews what the AI concluded. This one reviews what the AI is, and what it's allowed to be.

An agent's real behaviour drifts from its scope, through a config change, an expanded permission, a credential that should have rotated and didn't. Catching that drift is its own analyst skill, and it's the thing most teams aren't doing yet for AI agents.

This is where my IAM background becomes a security-operations skill on paper: I'm running a miniature access review against a non-human identity. The same discipline I'd apply to a service account at work, now framed as a SOC control, not an IT chore.

**Exercise, check `agent-registry.md` against reality**
- Does the credential still match what's recorded?
- Is actual access still limited to what the registry claims, read alerts, write verdicts, nothing more?
- Are agent log entries still cleanly separate from my human actions?
- Has the credential rotated on the schedule I wrote down?

**A registry that's never wrong isn't being inspected**
Try to find at least one thing that's drifted, is vague, or was never truly verified. If everything genuinely checks out, document HOW I verified each item, so the writeup proves the review happened rather than just asserting "looks fine".

Writeup: treat any mismatch as a finding, what the registry claimed, what I found, why the gap matters, how I'd close it.

### Module 6 — Working with a local model's limits (after Phase 10)

Notice where the local model is confident and fast versus where it hedges or gets a low-severity alert wrong. That's not a flaw to hide, it's exactly the judgment a security-analyst-plus-AI role requires: knowing when to trust automation and when a case needs escalation.

Exercise: deliberately send it one alert type it handles well and one it struggles with. I may need to try several to find the struggle case.

Writeup: describe the routing rule I'd write from this observation, what stays local, what escalates, why.

### Ongoing — the standing habit

After any real triage, if it's the least bit interesting, write it up. The lesson modules are training wheels. The goal is doing this instinctively once the lab is just... running.

## The final test

For any phase, out loud:

1. What did I build?
2. What problem does it solve?
3. What went wrong while building it?
4. How did I fix it?

**Question 3 is the one that matters**
Interviewers lean on it hardest, and it's the one candidates smooth over. Don't. The pfSense lockout that cost a full reinstall. The duplicate-IP ARP failure that made pings vanish with no error. The orphaned bridge that took Kali offline silently. The switch SPAN that looked configured correctly and mirrored nothing that mattered, because the traffic never left the hypervisor's own virtual switch. Those are the answers that sound like somebody who has actually done the work, because I have.

If I can do that for every phase, the lab has done its job, whether or not it's still running six months from now.
