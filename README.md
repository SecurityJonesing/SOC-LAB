## About me

I'm a systems administrator with a background in IAM and SOC operations. Early in my career I worked security analyst-side, and what I took away from that was a clear sense of where I wanted to build my career going forward: administration and engineering. Understanding how systems are actually built, configured, and maintained gives me a level of depth I couldn't get staying purely on the analyst side, and it's the direction I've been deliberately growing in ever since. This lab is part of that — a way to keep building real, hands-on infrastructure skill on my own time, especially useful right now while I'm between roles after a layoff.

My background includes time at Arctic Wolf doing SOC triage, the U.S. Air Force, Wells Fargo, Medtronic, and Thomson Reuters, and most recently administering Entra ID, Intune, and Conditional Access policy for an environment of about 350 users. I hold CompTIA A+, Network+, Security+, CySA+, and PenTest+ certifications.

# Home SOC Lab — Segmented Network, Firewall Policy, and Detection Engineering

I took a flat, unmanaged home network and rebuilt it from scratch into something closer to how a real environment is actually run: segmented, firewalled, logged, and — eventually — watched by a SIEM with AI-assisted alert triage on top. I built and documented all of it myself, from the switch configuration up through the hypervisor networking, the firewall policy, and the detection stack.

## Why this exists

I'm a Technology Systems Administrator with an IAM/SOC background, and I wanted to build hands-on infrastructure and security-operations depth outside of work rather than just read about it. So this lab is a real network, not a diagram in a slide deck. Every decision in it is logged as it happened — what I built, what broke, why, and how I fixed it. The `build_log.md` file in this repo is that actual running record, not a cleaned-up summary I wrote after the fact to look good.

## What this includes

The network itself has four VLANs — Management, Infra, an isolated Range network for running attacks against on purpose, and a dedicated monitoring path that mirrors traffic for inspection. A pfSense firewall sits between all of them. The switch's only job is tagging traffic into the right VLAN; pfSense's job is deciding what's actually allowed to cross between them. Keeping those two jobs separate, rather than treating a switch and a firewall as interchangeable, is a deliberate design choice and something I can explain in detail, not just something I copied from a tutorial.

Everything runs on a Proxmox VE virtualization host — the firewall, the attacker and target VMs, and the SIEM stack all live there. That meant real hypervisor networking work: designing the bridges, mapping physical NICs to them correctly, and running into (and fixing) the operational quirks that come with that kind of setup.

None of it got built carelessly. Every configuration change — a firewall rule, a VLAN, a bridge reassignment — gets planned out, gets a rollback point (a VM snapshot, an exported config, a documented recovery path) before it's touched, and gets logged with the actual command output afterward. Not "it worked, moving on."

## Why this matters beyond just security

This isn't really a security showpiece. The skills underneath it are general infrastructure administration skills, just applied with the kind of discipline a production environment actually demands.

Rebuilding this network from a factory-reset switch into a segmented, trunked, firewalled topology is the same category of work as standardizing or migrating systems during an org transition or acquisition — building something correctly from the ground up, not patching around an existing mess.

I also did all of it solo. There was no team to split the work across — planning, execution, troubleshooting, and documentation all fell to me, and I built the process specifically so nothing depended on memory. That's the same posture as being the sole onsite technical resource for something.

A few examples of real troubleshooting worth mentioning, pulled straight from the build log rather than cleaned up for effect:

A network card that looked like a dead piece of hardware — no link, failed every cable and BIOS check — turned out to simply be administratively down. I caught it by checking the interface's actual state before escalating to hardware-level diagnosis, which saved a lot of wasted time.

A duplicate static IP address across two virtual network bridges caused pings to silently fail with no error message anywhere. I found it by comparing the live state of both interfaces side by side instead of trusting either one in isolation.

A firewall setup wizard's default setting would have blocked the network's own legitimate upstream traffic, because this build's internet connection comes from a private home network rather than a real ISP. I caught it by understanding *why* that default setting exists in the first place, not just clicking through what it does.

## Tech stack

Proxmox VE, pfSense CE, Cisco Catalyst switching (IOS), Windows Server and Windows 11, Wazuh, Docker, Git.


