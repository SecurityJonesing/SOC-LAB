# Detection: LSASS Credential Dumping via Silent Process Exit (T1003.001 / T1546.012)

**Rule ID:** 100005
**MITRE ATT&CK:** T1003.001, T1546.012
**Tactic:** Credential Access

## The Technique

LSASS holds credential material in memory, password hashes, Kerberos tickets, sometimes plaintext credentials. Dumping LSASS is one of the most valuable things an attacker can do on a box because it can hand over credentials for lateral movement without needing to exploit anything else.

The direct way of doing this, attaching a debugger or a memory dumping tool right to LSASS, is well monitored and a lot of the time blocked outright by newer Windows protections. This technique gets around that by abusing a legitimate Windows feature called Silent Process Exit, set up through the registry's Image File Execution Options key. By setting a GlobalFlag value against lsass.exe in the registry, an attacker can register a helper process to run automatically whenever LSASS exits, including a dump tool, without ever attaching to the live process.

## The Test

I validated this using an Atomic Red Team test that sets the IFEO GlobalFlag registry value against lsass.exe, which simulates the setup step an attacker would do before actually triggering the dump. I confirmed the registry write happened and checked Sysmon, which logged it correctly as a registry value set event.

There wasn't a built-in Wazuh rule already covering this specific IFEO/LSASS pattern. I checked the built-in ruleset for it before writing anything new.

## What I Built

A rule chained onto Sysmon's real registry write dispatch rule, watching specifically for the `Image File Execution Options\lsass.exe\GlobalFlag` registry path. I set it to severity level 14, high, since there's really no legitimate reason for this specific value to get set outside of this attack. I tagged it with two MITRE techniques instead of one, T1003.001 for the credential access goal and T1546.012 for the actual mechanism being abused, since they're describing two different things about the same event and I wanted both represented.

## What Went Wrong, and How I Found It

This one wasn't really a broken rule, it was a broken way of checking that made a working rule look broken.

wazuh-logtest, fed the raw event as JSON, only matched a generic catch-all rule instead of mine. I had already run into this exact problem with an earlier rule: raw JSON fed into wazuh-logtest goes through a different decoder than real production traffic does, so it can't reliably show how Sysmon events actually get matched. A logtest non-match doesn't mean the rule is broken, it means the test isn't trustworthy for this kind of event.

I checked the manager's own logs and confirmed the rule was written correctly and loaded clean, but I still couldn't get a solid live-fire confirmation. It turned out the actual problem was simpler than I thought, the test event I was checking against had been generated before the manager's last restart, meaning the rule genuinely wasn't live yet when that event happened. It wasn't a matching problem at all, just bad timing.

## The Fix

I ran a fresh atomic test after confirming the rule was actually loaded, then checked the raw alert log directly instead of trusting wazuh-logtest. The rule fired correctly, confirmed with the right rule ID, both MITRE tags, and a timestamp well after the rule went live.

## What This Would Catch, and What It Would Not

This catches the specific IFEO/GlobalFlag setup step for Silent Process Exit against LSASS. It would not catch other credential dumping methods, direct memory access tools when they aren't blocked by Protected Process Light, comsvcs.dll MiniDump abuse, or LSASS dumping through a different mechanism that doesn't use IFEO. Each of those leaves a different footprint and would need its own rule.
