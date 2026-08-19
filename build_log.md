## 2026-08-18 — FileDeleteDetected (Event 26) config attempt — unresolved, reverted

**Phase:**       C (Detection Engineering)
**Goal:**        Enable Sysmon Event ID 26 (FileDeleteDetected) on Win11-LTSC-Victim to close the
                  known FileDelete telemetry gap and unblock future T1070.004-family detection work.
**Transcript:**  Long session, heavy diagnostic detour.

### What happened
Added a new `<RuleGroup name="FileDeleteDetection" groupRelation="or">` containing an empty
`<FileDeleteDetected onmatch="exclude">` block to `sysmonconfig-export.xml`, placed immediately
after the existing (commented-out) Event 23 FileDelete section. Verified via schema dump
(`Sysmon64.exe -s`) that `FileDeleteDetected` (event value 26) is the correct, version-confirmed
element name for this build (v15.21, schema 4.91). XML validated cleanly via PowerShell's `[xml]`
parser and reloaded with no parse errors (`Sysmon64.exe -c`).

**Verification never succeeded.** Across two full VM reboots (required since Sysmon blocks
standard service-stop requests as tamper protection — confirmed expected behavior, not a bug),
Event ID 26 never appeared in the Windows Event Log for a deliberately generated test file
deletion, and `FileDeleteDetected`/`FileDelete` never appeared in the live `-c` config dump —
even after renaming the `RuleGroup` from an empty `name=""` to `name="FileDeleteDetection"` on
the theory that an unnamed rule group might be silently dropped.

**Root cause not isolated.** Substantial diagnostic effort went into what turned out to be a red
herring: PowerShell's console/pipeline handling of this Sysmon binary's output introduced null-byte
corruption and premature pipeline truncation (`Select-Object -First`/`-Path`-based `Select-String`
searches against captured Sysmon output were unreliable all session), which repeatedly produced
false "not found" results and cost significant time before being identified as a diagnostic-tooling
issue rather than a config issue. Once ruled out (via `notepad`'s Ctrl+F on a clean file), the
absence of Event 26 was confirmed genuine — the config change is not taking effect in the live
driver, for a reason not yet identified.

**Reverted for cleanliness.** The `RuleGroup` block was wrapped in a comment (matching the existing
commented-out Event 23 section's style) and the config reloaded clean, so the running config now
matches the file with no orphaned/half-working rule group in place.

### Outcome
- **FileDelete/FileDeleteDetected telemetry gap remains open** — file-deletion-based detections
  (T1070.004 and similar) are still not possible on Win11-LTSC-Victim.
- Config file is back in a clean, consistent state (Event 26 section commented out, matching Event 23).
- No lasting change to the VM's detection posture from tonight's session.

### Next-steps (logged, not built)
- Investigate whether PPL/tamper-protection or Defender is silently blocking Event 26 registration,
  similar to the LSASS PPL interaction hit during the T1003.001 session — worth checking
  `Get-MpThreatDetection` and Defender exclusions specifically for this event type before the next attempt.
- Consider testing Event 23 (FileDelete, full archive) instead, to see if the same failure occurs —
  would help isolate whether this is FileDelete-family-specific or a broader Sysmon driver issue on this VM.
- Consider a from-scratch reinstall of Sysmon (`-u` then `-i -c <config>`) rather than a `-c` reload,
  in case the driver itself has a stale registration that a simple reload/reboot doesn't clear.

### Lessons
1. **PowerShell's handling of native console output from this specific Sysmon binary is unreliable
   in this environment** — `Select-String -Path`, `Select-Object -First` piped directly onto a live
   external command, and even `[Console]::OutputEncoding` mismatches all produced silent false
   negatives tonight. When search tools give suspiciously consistent "nothing found" results against
   data you can visually confirm exists, verify with the simplest possible method (`.Contains()` on a
   directly-loaded string, or just `notepad` + Ctrl+F) before trusting the search tool's output.
2. **An empty `name=""` attribute on a Sysmon `RuleGroup` was a reasonable, testable hypothesis but
   was not the actual cause** — worth remembering so it isn't re-tried as a fix next time without new
   evidence.
3. **This is a genuine, currently-unresolved gap** — not a documentation or config-syntax problem.
   Future sessions should start from Defender/PPL interaction as the next hypothesis, not repeat
   tonight's XML/encoding diagnostics.
