# Detection: Registry Run Key Persistence (T1547.001)

**Rule ID:** 100004
**MITRE ATT&CK:** T1547.001
**Tactic:** Persistence

## The Technique

Attackers who get code execution on a machine often want to survive a reboot. One of the oldest, and simplest ways to do this is to write a value into a registry "Run" key. Anything there launches automatically every time that specific user logs in. It doesn't need admin rights and it's still around because it works.

## The Test

I validated this using Atomic Red Team test T1547.001-1, which writes a value to `HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run` via reg.exe. I confirmed the write, then checked Sysmon and saw it correctly logged as Event ID 13 (registry value set), tagged by the SwiftOnSecurity ruleset as T1060,RunKey.

Wazuh already had a built-in rule (92302) that catches registry writes via reg.exe in general, but it's broad, not specific to persistence keys. I wanted a rule that calls out this exact pattern by name, at a severity that reflects how serious it is.

## What I Built

A custom rule watching Sysmon's registry-write events for any value written under a CurrentVersion\Run or RunOnce key, using a case-insensitive regex on the registry path that matches. I set the severity to level 14 (high) since a legitimate reason to write directly to this key outside of an installer is rare.

## What Went Wrong, and How I Found It

The rule didn't fire. No errors anywhere and it passed Wazuh's own syntax check, the manager restarted clean, but another test run showed nothing.

I confirmed the event was fine (the registry value was correct in the raw log) so the problem was how the rule was written, not in the data. I ended up tracing it back to a gap in how Wazuh's rule engine works: it evaluates rules in load order and stops at the first rule that matches a given event. My rule and Wazuh's own built-in rule (92300) were both listening for the same broad category of event. It grabbed the event before my rule got a chance to run because the built-in rule loads first and it is broad enough to match mine too. My rule didn't fail, it just never got evaluated.

## The Fix

I rewrote the rule to chain directly onto the specific built-in rule ID that was intercepting the event, instead of matching on the same broad category itself. This makes my rule a child of the one that was already claiming these events, so now it runs as a sibling check rather than competing for the same slot.

After that change, I re-ran the atomic test and the rule fired correctly. I verified it directly against the raw alert data, which showed the right rule ID, the right severity, and the correct MITRE tag.

## What This Would Catch, and What It Would Not

This catches the classic version of the technique: a value written straight to a Run key via the registry. It would not catch more evasive variants: encoded or obfuscated key paths, WMI-based persistence, or scheduled-task-based persistence, which all use different mechanics and would need their own separate rule.
