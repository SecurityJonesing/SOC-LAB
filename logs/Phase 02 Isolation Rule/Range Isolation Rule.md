# Phase 2: Isolation Rule

## 2026-07-30, Range isolation rule written and applied

**Phase:** 2
**Goal:** Write and apply the pfSense Range-VLAN isolation rule, deny all outbound from Range, with one explicit allow to Infra's Wazuh ingest port.
**Rollback:** Proxmox snapshot `pre-isolation-rule` (VM 102) taken from the pve01 shell. pfSense config also exported as XML and saved locally.
**Transcript:** session recording on throughout.

### What happened

Gate items confirmed before any firewall change: fresh Proxmox snapshot `pre-isolation-rule` taken on VM 102, pfSense XML config exported and saved locally, session recording confirmed on.

Interface labels renamed for clarity (cosmetic only): LAN to MGMT10, OPT1 to INFRA20, OPT2 to RANGE30, applied once after all three renames.

Rule 1, the allow, built on the RANGE30 tab: Pass, IPv4, TCP, source RANGE30 subnets, destination INFRA20 subnets (placeholder, Wazuh doesn't exist until Phase 3, to be tightened to a single host IP once built), port 1514, description "allow range -> wazuh agent ingest ONLY".

Rule 2, the default deny, built second: Block, RANGE30 subnets to any, description "default deny - range is isolated".

Rules were built and read back field by field using the Claude in Chrome extension in a strict point/describe only mode, the extension highlighted each field and stated the value to enter, every field was typed in by hand and confirmed via screenshot before Save.

Reordering gotcha, caught live: dragged Rule 2 to confirm its position relative to Rule 1. Discovered that a drag-to-reorder is not committed by the drag alone, clicking Apply Changes immediately after a drag, without an intervening Save on the rule list, triggered pfSense's own "unsaved changes" warning. Canceled out of that warning, clicked Save first (which committed the new order), then clicked Apply Changes.

Applied successfully: pfSense confirmed the rules were reloading, order preserved on reload. Post-apply sanity check: confirmed continued GUI access from MGMT10 immediately after applying.

### Outcome

Both rules live on the RANGE30 interface: Pass (RANGE30 to INFRA20 subnets, TCP 1514) above Block (RANGE30 to any, any). pfSense GUI access from MGMT10 confirmed unaffected.

Rule exists and is correctly scoped, but isolation is not yet acceptance-tested live, the target VMs are still on the flat network, not yet moved to the Range VLAN. That move is Phase 3. The real "no internet, no home net, no MGMT, only Wazuh:1514" test can't run until a VM actually sits on RANGE30.

Destination on Rule 1 (INFRA20 subnets) is intentionally broader than the final design (a single Wazuh host), revisit once Wazuh is deployed.

### Lessons

1. A drag-to-reorder in pfSense's rule list is not committed until Save on the rule list itself is clicked, sequence going forward: drag, Save, Apply Changes.
2. Renaming interface descriptions is purely cosmetic but meaningfully reduces the chance of picking the wrong tab under pressure.
3. A rule being applied cleanly is not the same as the isolation being proven. Don't mark this phase fully complete until the acceptance check has actually run against a real VM.

Note: this rule's live isolation test against a real VM did not happen until Phase 3 Step 3 (2026-08-04), see "Victim to Range VLAN and Agent Enrollment."
