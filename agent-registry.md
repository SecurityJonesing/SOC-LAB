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

### `wazuh-triage-01` (Phase D)

- **Purpose:** Pulls new Wazuh alerts, suggests a verdict, hands the
  suggestion to me for review/confirm/override.
- **Credential:** Dedicated Wazuh API user — not the admin account,
  not my own session.
- **Scope:** `alerts:read`, `agent:read` only. No delete, no config
  changes, nothing outside the Infra alert pipeline.
- **Audit trail:** Every action logs as `agent=wazuh-triage-01`. My own
  review/override actions log separately as `user=michael` — the two
  trails must stay distinguishable from each other.
- **Status:** Not built yet.

### `triage-router-01` (Phase E)

- **Purpose:** Routes both SOC triage and my own personal AI requests
  between local inference (Ollama on `pve-ai`) and cloud escalation
  (Claude Code via `claude -p`, or Grok), based on a severity/ambiguity
  threshold I define.
- **Credential:** TBD — decided when Phase E is actually built.
- **Scope:** TBD.
- **Audit trail:** TBD — same principle as above: its routing decisions
  need to be distinguishable from my own manual choices.
- **Status:** Not built yet.

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
