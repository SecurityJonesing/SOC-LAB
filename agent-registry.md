# agent-registry.md

Every AI agent operating in this lab: what it is, what it may touch, who owns it, and whether it's still supposed to exist.

**Why this file exists.** An AI agent that calls APIs and reads alert data is a **non-human identity**. It gets governed like one — its own credential, least-privilege scope, its own audit trail, a named owner, and a rotation schedule. Not because it's tidy, but because an agent running on a human's personal credential is indistinguishable from that human in every log it touches. That's an accountability gap, not a style preference.

**Maintenance rules:**
- An agent that isn't in this file **should not be running.** No exceptions, including "just testing."
- Update on **every** change: new agent, rescoped, credential rotated, retired.
- Reviewed on a schedule (below). A registry that's never wrong on inspection isn't being inspected — see `SOC-Lab-Lesson-Plans.docx`, Lesson Plan 2, Module 5.5.
- **Never put a credential in this file.** Record *where* it lives and *when* it rotates — never the value. This repo is public.

---

## Schema

| Field | Meaning |
|---|---|
| `id` | Unique, stable name. Appears verbatim in log lines so agent actions are attributable. |
| `purpose` | One sentence. If it takes a paragraph, the agent does too much. |
| `may access` | Explicit allow-list. Not "Wazuh" — the exact API scopes. |
| `may NOT` | The things it must be *provably* unable to do. Each one should be testable. |
| `owner` | A human name. Not a team, not "IT." One person. |
| `credential type` | Where the credential lives. Never the credential itself. |
| `rotation` | Interval + last rotated + next due. |
| `status` | `planned` / `active` / `retired`. |
| `audit trail` | Where its actions are logged, and how they're distinguished from human actions. |
| `last review` | Date this entry was last checked **against reality**, not just read. |

---

## Agents

### `wazuh-triage-01` — status: **planned** (Phase D, not yet created)

| | |
|---|---|
| **purpose** | First-pass triage of new Wazuh alerts. Suggests a verdict; decides nothing. |
| **may access** | Wazuh API on the Infra VLAN — `alerts:read`, `agent:read`. A model API for inference. |
| **may NOT** | Delete agents · modify detection rules · change Wazuh config · reach anything outside the Infra VLAN alert pipeline · act on its own verdict |
| **owner** | Michael Mathews |
| **credential type** | Wazuh internal user, dedicated. **Not** the admin account. **Not** Michael's session. |
| **rotation** | Every 90 days · last: `____-__-__` · next due: `____-__-__` |
| **status** | planned |
| **audit trail** | Every action logs `agent=wazuh-triage-01`. Michael's review/override logs `user=michael`. The two are never ambiguous in a single log line. |
| **last review** | `____-__-__` |

**Scope test — must pass before this agent goes active:**
```bash
# authenticate as the agent user
curl -k -u <agent-user>:<pass> -X POST \
  "https://<wazuh>:55000/security/user/authenticate?raw=true"

# MUST succeed — reading alerts is its job
curl -k -H "Authorization: Bearer <tok>" "https://<wazuh>:55000/agents?pretty"

# MUST return 403 Forbidden — this is the actual test
curl -k -H "Authorization: Bearer <tok>" -X DELETE \
  "https://<wazuh>:55000/agents?pretty"
```
If the DELETE succeeds, the scope is wrong. Fix it before the agent runs. **A test that cannot fail proves nothing.**

---

### `triage-router-01` — status: **planned** (Phase E, not yet created)

| | |
|---|---|
| **purpose** | Decides whether an alert is handled by the local model or escalated to the cloud model. |
| **may access** | Ollama API on `ai-vm` · cloud model API · the triage service's alert queue |
| **may NOT** | Modify alerts · write verdicts directly · reach the Wazuh API at all |
| **owner** | Michael Mathews |
| **credential type** | `____________________` (TBD — depends on how `ai-vm` ends up exposed after the VLAN decision) |
| **rotation** | `____________________` · last: `____-__-__` · next due: `____-__-__` |
| **status** | planned |
| **audit trail** | Logs `agent=triage-router-01` with the routing decision **and the reason**. The reason is the point — it's what makes the routing rule reviewable. |
| **last review** | `____-__-__` |

**Why this is an agent and not plumbing:** it makes an autonomous decision — which alerts a human-supervised model sees versus which get handled locally and cheaply. A bad routing rule silently downgrades real incidents. That's a decision with consequences, so it gets governed like one.

---

## Review log

Log every review, including the clean ones. A review that found nothing still proves the review happened — *if* you record how you verified it.

| Date | Reviewed | Findings | Actions taken |
|---|---|---|---|
| | | | |

**Review checklist** (per `SOC-Lab-Lesson-Plans.docx`, Module 5.5):
- [ ] Does each agent's credential still match what's recorded here?
- [ ] Is actual access still limited to `may access`? Tested, not assumed.
- [ ] Do the `may NOT` items still fail? Re-run the scope tests.
- [ ] Are agent log entries still cleanly separable from human actions?
- [ ] Has each credential rotated on schedule?
- [ ] Is anything `active` here that shouldn't be running anymore?
- [ ] Is anything running that **isn't in this file**?

---

## Why this is worth doing in a home lab

Non-human identities — service accounts, API keys, AI agents — already outnumber human accounts at most organisations, often by a wide margin. Governance hasn't caught up: a large majority of leaders call agent governance critical, while a minority have implemented any policy for it.

Current job postings ask for this directly. One example, verbatim from a 2026 Principal-level posting: *"build and maintain a centralized AI agent registry that supports agent onboarding, unique agent identification, metadata management, ownership tracking, lifecycle status, versioning, and auditability"* — listed alongside agentic AI, RAG, and OAuth/OIDC scoped tokens.

That is this file. In miniature, on real infrastructure, maintained over time — with a review log proving it was actually inspected rather than written once and forgotten.
