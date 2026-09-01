# Detection: Local Account/Group Enumeration via net.exe (T1087.001)

**Rule ID:** 100006
**MITRE ATT&CK:** T1087.001
**Tactic:** Discovery

## The Technique

Before moving laterally or trying to escalate privileges, attackers usually want to know who else has access on a box, other local accounts, who's in the local admin group, what groups even exist. It's basic recon but it's a reliable early signal. net.exe commands like `net user` and `net localgroup` answer these questions instantly using tools that are already built into Windows, which is exactly why they show up so often in real intrusions, and also why they're just as common in normal day-to-day admin work.

## The Test

I validated this using Atomic Red Team tests that run `net user`, `net localgroup Users`, and `net localgroup`, three separate enumeration commands covering the common ways this technique gets used.

## What I Built

A rule chained onto the same Sysmon Event ID 1 dispatch rule that 100002 uses, watching for net.exe as the process with a command line matching account or group enumeration. I kept it narrow on purpose, matching exactly what I tested instead of a broad catch-all for any net command. I set it to severity level 8, not 14, since `net user` and `net localgroup` are routine admin activity and tagging them at the same severity as credential dumping would just create noise I'd start ignoring.

## What Went Wrong, and How I Found It

I caught this one before it went live, during my own review of the draft. The first version of the rule used four backslashes in the image path field, which is the same escaping I used for 100004 and 100005, but those match against a different field, targetObject, the registry path. I compared the draft against 100002's own already-working image field pattern and the convention didn't match, image path fields use two backslashes, not four. The two field types don't share the same escaping and I'd carried the wrong one over out of habit.

## The Fix

I corrected the image field to the two-backslash convention, matching 100002's pattern exactly. I deployed it, confirmed a clean restart with no parse errors, then reran all three atomic tests. I checked the raw alert log directly and the rule fired three times, once for each command I tested, each with the right MITRE tag.

## What This Would Catch, and What It Would Not

This catches the specific net.exe enumeration commands I tested. It would not catch the same recon done through PowerShell cmdlets like Get-LocalUser or Get-LocalGroupMember, WMI queries, or other built-in tools like dsquery or wmic, since each of those has a completely different process and command line footprint and would need its own rule. It also wouldn't catch enumeration run through a renamed copy of net.exe.
