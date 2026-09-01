# Detection: Network Share Removal via net.exe (T1070.005)

**Rule ID:** 100007
**MITRE ATT&CK:** T1070.005
**Tactic:** Defense Evasion

## The Technique

After using a network share to stage tools or pull data out, an attacker might delete that share to get rid of evidence it was ever there. It's a small anti-forensics step. Removing a share only takes one command, doesn't need any special tooling, and is easy to miss since share management is routine admin work too.

## The Test

I validated this using an Atomic Red Team test that runs `net share` on a test share I created for this, followed by delete.

## What I Built

A rule chained onto the same Sysmon Event ID 1 dispatch rule as 100002 and 100006, watching for net.exe with a command line matching the share deletion syntax. Same narrow approach as 100006, it matches exactly the tested pattern instead of a broad `net share` catch-all. I set it to level 8 for the same reason, net share administration is routine, not inherently malicious, so it needed a severity that matches real-world base rate instead of worst-case intent.

## What Went Wrong, and How I Found It

Same escaping mistake as 100006, since I drafted both rules together off the same starting point. The first draft used four backslashes, copied from the targetObject-based rules, instead of the two backslashes image path fields actually need.

## The Fix

I corrected the image field to two backslashes, matching 100002 and the now-fixed 100006. I deployed it, confirmed a clean restart, reran the atomic test, and checked the raw alert log directly. The rule fired once with the correct description and MITRE tag.

## What This Would Catch, and What It Would Not

This catches direct net.exe share removal matching the tested syntax. It would not catch the same result done through PowerShell with Remove-SmbShare, WMI, or the Computer Management GUI, since none of those go through net.exe at all and would need their own separate rule.
