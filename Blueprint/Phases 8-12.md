# Blueprint — Phases 8 through 12

Part of `LAB-BLUEPRINT.md`'s "Build Phases" section, split out 2026-09-07 to stay under the ~16,000-character tool-read limit. See `LAB-BLUEPRINT.md` for the full purpose, environment, and phase index; see `Blueprint/Phases 1-6.md` and `Blueprint/Phase 7 AD Expansion.md` for the rest.

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
- The `Write-Up ....md` files embedded in `Build Logs/` serve as case documentation in the meantime.

---
