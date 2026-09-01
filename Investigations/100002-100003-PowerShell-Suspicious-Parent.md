# Detection: PowerShell Spawned by a Suspicious Parent Process (T1059.001 / T1027)

**Rule ID:** 100002 / 100003
**MITRE ATT&CK:** T1059.001, T1027
**Tactic:** Execution

## The Technique

PowerShell is a favorite tool for attackers because it's already installed on the machine, trusted by most security tools, and it can do almost anything once it's running. A common giveaway isn't PowerShell itself, it's what launched it. A user doesn't usually open PowerShell by double clicking cmd.exe or a script host like wscript.exe. When that shows up it's usually a macro, a dropped script, or a phishing payload handing off to PowerShell. Attackers make it worse by encoding the command in Base64, which hides what it's actually doing from a quick look at the process list.

## The Test

I validated this using Atomic Red Team test T1059.001-17, which spawns PowerShell from cmd.exe with a Base64 encoded command. I confirmed a real example event first, powershell.exe launched by cmd.exe, with the parent command line showing `cmd.exe /c powershell.exe -e` followed by the encoded command, which is exactly the pattern I needed to catch.

## What I Built

Two rules that work together. 100002 is the base rule, PowerShell spawned by a suspicious parent process like cmd.exe or wscript.exe. 100003 is a child rule that only checks once 100002 already matched, and adds one more condition on top, an encoded command in the parent's command line. Plain PowerShell from cmd is worth watching, but PowerShell from cmd with an encoded payload is worth a higher severity alert.

## What Went Wrong, and How I Found It

The rule didn't fire, and it took three separate bugs before it actually worked, each one hiding behind the last.

First, the rule was decoding the event as json, but real Sysmon events actually get tagged windows_eventchannel by Wazuh's decoder. I fixed that but the rule still didn't fire.

Second, I had copied the rule file into the Wazuh container using `docker cp`, which keeps the host's file ownership instead of switching to the container's. Wazuh's own process couldn't read a file it didn't own, and it failed with no error at all. I found this by checking file ownership directly inside the container instead of trusting that no error meant it was fine.

Third, and this was the real blocker, every field in the rule used a `data.win.eventdata` path. That's how fields are named in the exported alert JSON you see in the dashboard, but it's not the actual namespace Wazuh's rule engine checks against when it evaluates a rule. Every condition was silently failing to find its field and coming back false, even though the logic itself was right. I caught this by comparing my field paths against a working built-in rule, which used the same fields without the `data.` prefix.

One more thing came up while I was checking this worked. I kept searching for rule 100002 by itself in the dashboard and getting nothing, which looked like it was still broken. Turns out Wazuh only shows the more specific child rule as the alert when both a parent and child rule match the same event, so 100002 matching is implied by 100003 firing, it just doesn't show up on its own.

## The Fix

I stripped the `data.` prefix off every field path and rechained the rule to route through the same Sysmon event group the built-in rule uses. I re-ran the atomic test and both 100002 and 100003 fired correctly, confirmed straight from the raw alert data.

## What This Would Catch, and What It Would Not

This catches the parent process plus encoding pattern from this specific atomic test. It would not catch PowerShell launched from a parent process outside the list I'm watching, PowerShell invoked in less common ways like Invoke-Expression piped in instead of a command line flag, or obfuscation methods other than Base64 encoding. Those would all need their own rule or a broader pattern.
