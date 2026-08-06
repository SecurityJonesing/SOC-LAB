# Agent Registry

Tracks every AI agent operating in this lab: its scope, credentials, and
audit trail. The goal is that no agent in this lab acts under my own
identity or with more privilege than its specific job requires — every
agent gets its own credential, scoped narrowly, with its actions logged
separately from mine.

No agents exist yet — this will be populated for real starting in Phase D.
The entries below are the planned scope for each, decided ahead of time
so the access model is settled before any code runs, not bolted on after.

---

## Planned entries

**Relationship between the two planned Phase D/E agents:** `wazuh-triage-01` is the specialist — the only thing with actual Wazuh alert access, scoped narrowly to that one job. `triage-router-01` is general routing plumbing underneath it — not Wazuh-specific, since it also routes other lab automation through n8n. `wazuh-triage-01` calls into `triage-router-01` to decide local vs. cloud for a given alert; they're not redundant, and merging them would mean a general-purpose router sitting on Wazuh API access it doesn't need for most of its job.

### `wazuh-triage-01` (Phase D)

- **Purpose:** Pulls new Wazuh alerts, suggests a verdict, hands the
  suggestion to me for review/confirm/override.
- **Credential:** Dedicated Wazuh API user — not the admin account,
  not my own session.
- **Scope:** `alerts:read`, `agent:read` only. No delete, no config
  changes, nothing outside the Infra alert pipeline.
- **Audit trail:** Every action logs as `agent=wazuh-triage-01`. My own
  review/override actions log separately as `user=analyst` — the two
  trails must stay distinguishable from each other. (Exact log-identifier
  convention to be finalized when Phase D is actually built — this is a
  placeholder label, not a name.)
- **Status:** Not built yet. Sequenced after Phase C.6/C.7 in the current
  blueprint — a full kill chain gives this agent a much richer, more
  realistic alert stream to triage than isolated detection tests alone
  would. Also deliberately sequenced *before* `triage-router-01` — this
  is the simpler case, meant to prove the governance model (own
  credential, least privilege, separate audit trail) before Phase E adds
  routing complexity on top.

### `triage-router-01` (Phase E)

- **Purpose:** Routes SOC triage requests (from `wazuh-triage-01`) and
  general lab automation requests between local inference (Ollama on
  `pve-ai`) and cloud escalation (Claude Code via `claude -p`, or Grok),
  based on a severity/ambiguity threshold I define. **Scoped to lab/SOC
  and general lab automation only — the `pve-ai` node is not used for
  personal AI requests, so this agent has no personal-use routing role.**
- **Credential:** TBD — decided when Phase E is actually built.
- **Scope:** TBD.
- **Audit trail:** TBD — same principle as above: its routing decisions
  need to be distinguishable from my own manual choices.
- **Status:** Not built yet. The underlying `pve-ai`/`ai-vm` GPU
  infrastructure is already built and verified (see `build_log.md`,
  Phase E entries); what remains is the Ollama/Open WebUI stack itself,
  then this router. **Execution model note:** the now-merged AI node
  build used a model where Claude Code executed commands directly over
  SSH after my confirmation, rather than my typing every command myself.
  This was a deliberate choice, not a shortcut around learning — I'd
  already done a Proxmox setup and an Ubuntu Server install once before
  this build, and had done GPU passthrough at work previously as well,
  so I already had hands-on familiarity with the concepts involved and
  delegated the repetitive execution to Claude Code while retaining
  review/confirmation authority over every step. The remaining Phase E
  work (Ollama, Open WebUI) reverts to this project's standard model
  instead — I run every command myself — for consistency with the rest
  of this build, since that part is new territory for me here.

### Shuffle playbook identity (Phase F) — new

- **Purpose:** Executes the Wazuh → Shuffle → enrichment → notify
  playbook. Registered here even before Phase F starts, since Shuffle
  fits this file's own definition of a non-human decision-maker taking
  action, not just an AI model. If the optional active-response step
  (blocking an IP via pfSense) is ever turned on, this becomes the
  single highest-stakes non-human identity in the lab — the one thing
  capable of autonomously modifying network state.
- **Credential:** Its own Wazuh API credential, separate from
  `wazuh-triage-01`'s. Its own VirusTotal/AbuseIPDB API keys for
  enrichment. If/when active-response is chosen: a pfSense API
  credential scoped to nothing but adding entries to one specific
  block-list alias — nothing else.
- **Scope:** Read Wazuh alerts relevant to its playbook trigger; call
  enrichment APIs; post notifications. No broader Wazuh access, no
  pfSense access at all unless/until active-response is explicitly
  decided (per the two-tier rule — a separate, deliberate firewall
  decision, not bundled into standing up Shuffle itself).
- **Audit trail:** `agent=shuffle-playbook-01`, distinguishable from
  both `wazuh-triage-01`'s and my own actions.
- **Status:** Not built yet. Sequenced after Phase C.6, per the
  blueprint — a real, varied detection stream from the full kill chain
  gives the first playbook something substantial to react to.

### Entra Connect sync account (Phase C.7) — new

- **Purpose:** Not an AI agent, but a non-human, scheduled, privileged
  identity that fits this file's governing principles — the account
  `entra-connect-01` uses to read from on-prem AD and push to the
  Entra ID tenant. Registered here deliberately: in real breaches, the
  AD Connect/Entra Connect sync account is a well-known, high-value
  target, because it typically holds broad directory access to do its
  job. Documenting it here, scoped as tightly as possible, is also a
  direct, deliberate echo of the DCSync misconfiguration planned
  elsewhere in this build — the sync account is a legitimate real-world
  example of exactly that risk pattern.
- **Credential:** Dedicated service account, not a Domain Admin
  account — scoped to the minimum AD replication/read permissions
  Entra Connect actually requires, not broader.
- **Scope:** Read access to synced attributes/objects only. No write-back
  beyond what the hybrid identity design in `LAB-BLUEPRINT.md` Phase C.7
  explicitly calls for (password writeback, if ever enabled — currently
  not part of the plan).
- **Audit trail:** Sync activity should be distinguishable in both AD's
  own logs and Entra ID's sign-in/audit logs from any interactive
  admin action.
- **Status:** Not built yet. Comes online as part of Phase C.7's Entra
  Connect step.

---

## Governing principles (apply to every future agent added here)

1. **Own identity, not mine.** Every agent gets its own dedicated
   credential — never runs under my personal login or an admin account.
2. **Least privilege.** Scope is decided *before* the agent is built,
   not expanded after the fact because it turned out to be convenient.
3. **Separate audit trail.** An agent's actions and my own reviews of
   those actions must be distinguishable in logs — never merged into
   one undifferentiated trail.
4. **This file stays current.** Any change to an agent's scope gets
   reflected here in the same session it's made, the same discipline
   as `build_log.md`.
5. **No agent identity uses my name or any other identifying detail.**
   Log labels, credential usernames, and audit-trail identifiers are
   generic role labels (e.g. `wazuh-triage-01`, `user=analyst`), consistent
   with the rest of this documentation set.
