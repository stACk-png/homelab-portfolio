# Isolated Lab Network (Red/Blue Team Lab)

## What it is

A separate, locked-down network on the Proxmox box where I run intentionally vulnerable machines to attack them. It's sealed off from my real home network and from the internet, with exactly one exception: the vulnerable boxes are allowed to reach the Wazuh SIEM so their agents can report. Nothing else gets in or out.

The whole point is to be able to attack a box and then immediately go look at what that attack looked like in Wazuh, without any risk of a compromised machine touching the rest of my network.

## Why I built it

This is the blue/red team lab I mentioned wanting to build in the main README. Running deliberately vulnerable machines is dangerous if they can reach anything real — the first thing an attacker does after popping a box is look around for what else they can get to. So before putting any vulnerable target on the network, I wanted a segment that physically and logically can't pivot anywhere.

The second requirement was that I still need to *see* what's happening on those boxes. A vulnerable machine I can attack but can't monitor only teaches me the red side. One that's attackable *and* watched by my SIEM teaches both — I attack it, then I write detections for what I did. That attack-then-detect loop is the actual goal.

Those two requirements pull against each other (isolation says "cut it off from everything," monitoring says "let it talk to the SIEM"), and the whole design is about resolving that with one narrow, deliberate hole.

## How it's laid out

```text
Home LAN (192.168.0.0/24)
   |
   +-- Proxmox host (192.168.0.2)
   +-- Docker VM / Wazuh (192.168.0.248)
   +-- ThinkPad / Kali (192.168.0.162)   attacker, reaches the lab over Tailscale
   |
   +-- OPNsense WAN (192.168.0.61)
             |
       [ OPNsense firewall ]   <- the only gateway in or out
             |
       OPNsense LAN (10.10.10.1)
             |
       Isolated lab (10.10.10.0/24)   vmbr1, no uplink
             |
             +-- Lin.Security target (10.10.10.20)
```

The lab lives on its own Proxmox bridge, `vmbr1`, which has **no physical uplink** — it's a software-only switch. That alone means at the switching layer it's already cut off from everything. Then OPNsense sits between it and the real network as the only router, and that's where the actual policy lives.

There's no DHCP on the lab on purpose, so every box gets a static IP I assign by hand. Addressing convention I'm using: `.1` gateway, `.10-.19` infra, `.20-.29` targets, `.30-.39` attackers.

## The firewall (OPNsense)

OPNsense runs as its own VM (ID 102) with two network cards — one leg on the home LAN (its "WAN"), one leg on the lab (`10.10.10.1`, the lab's gateway). A firewall routes between two networks by having a foot in each, and this is that.

One thing that confused me at first: OPNsense calls the home-LAN-facing side "WAN" even though it's not the internet. To OPNsense, "WAN" just means the upstream/untrusted side, which in my setup is my own home LAN.

The containment policy is default-deny with a short list of allows, ordered (order matters — first match wins):

| From the lab, to... | Rule |
|---|---|
| Wazuh manager (192.168.0.248, ports 1514/1515) | **allow** |
| Tailnet (100.64.0.0/10) | **block** |
| Anything else on the home LAN (192.168.0.0/24) | **block** |
| Internet | allow |

So the lab can report to the SIEM and pull updates, and that's it. It can't reach my home LAN and it can't reach my other Tailscale devices.

I tested every one of these from inside the lab rather than assuming they worked:

| From the lab, to | Expected | Result |
|---|---|---|
| 10.10.10.1 (gateway) | reach | works |
| 192.168.0.248:1514/1515 (SIEM) | reach | works |
| 192.168.0.2 (home LAN) | block | blocked |
| 100.94.110.39 (a Tailscale device) | block | blocked |
| 8.8.8.8 (internet) | reach | works |

(One gotcha: ping to the SIEM is blocked even though the agent ports aren't — the allow rule is TCP 1514/1515 only, and ping is ICMP. That's correct, not a bug. I test the ports with `nc`, not `ping`.)

## The target (Lin.Security)

The first vulnerable box on the lab is Lin.Security, an Ubuntu 18.04 machine built specifically for privilege-escalation practice. I imported it from an OVA (verified the checksum first, since I'm about to run a box that's built to be exploitable — I want to be sure it's the real one), converted the disk, and put it on the lab network at `10.10.10.20`.

I originally planned to use Metasploitable 3 here, but switched — the MS3 Ubuntu build is 14.04, which is too old to run a modern Wazuh agent cleanly, and it needs a finicky build process. Lin.Security is 18.04, ships ready to import, and the agent installs fine. That mattered because the whole point is monitoring it, and a target I can't put an agent on only does half the job. (DVWA, which I already run, covers the web-app attack side.)

It's got a Wazuh agent installed, version-matched to my 4.9.0 manager, enrolled across the firewall. The agent's running FIM, SCA (the CIS Ubuntu 18.04 benchmark), rootcheck, and log collection, all reporting back. So anything I do to this box shows up in the SIEM.

## Reaching it remotely (Tailscale subnet router)

I'm on campus a lot, and the lab is sealed off from my home LAN, so even when I'm home my laptop can't just reach the lab. I wanted to attack it from my ThinkPad's Kali VM from anywhere, without poking a hole in the isolation.

The way I did this is to make OPNsense a **Tailscale subnet router**. It already routes the lab, so I installed the Tailscale plugin on it and had it advertise `10.10.10.0/24` into my tailnet. My ThinkPad accepts that route and can now reach the lab through OPNsense over the tunnel.

The important part is that this is **one-directional on purpose**:

```text
ThinkPad / Kali  --- IN -->  OPNsense  --->  lab     (allowed)
      lab        <-- OUT --  (tailnet)               (blocked)
```

Opening a way *in* for my attacker box could accidentally become a way *out* for a compromised target — a popped box could try to ride that same Tailscale connection to reach my phone, my Proxmox host, whatever else is on my tailnet. The block rule in the firewall table above (lab -> `100.64.0.0/10`, the Tailscale address range) is what stops that. I confirmed it: my ThinkPad can reach into the lab, but the lab box cannot reach back out to any Tailscale device.

A detail I didn't expect: Tailscale NATs subnet-router traffic to the router's own subnet IP by default, so the target sees my attacks coming from `10.10.10.1` (the gateway), not my ThinkPad's Tailscale address. That's actually what keeps the one-directional block from breaking my own attack traffic, since the replies go back to the gateway, not to a blocked tailnet address.

## Problems I ran into

A few things broke in ways worth remembering:

- **A firewall rule that was correct but did nothing.** My allow rule for the lab-to-Wazuh traffic had every field right, but the traffic was being dropped and the rule showed zero matches. I ran `tcpdump` on the Wazuh host and saw nothing arriving at all, which told me it was the firewall dropping it, not the SIEM — and internet traffic still worked, so routing was fine. The rule had been saved but never actually committed into the running ruleset. Hitting "Apply Changes" loaded it and traffic passed instantly. Lesson: a saved rule and a live rule aren't the same thing.
- **"Block private networks" silently overriding my rule.** OPNsense has a built-in WAN protection that drops private-source traffic, and it runs at a higher priority than my own rules. It assumes WAN faces the internet, but mine faces my home LAN, so it was dropping legitimate traffic before my rule ever got looked at. Had to turn it off for that interface. The bigger lesson stuck: in a first-match firewall, rule precedence beats rule correctness.
- **The target booted with no IP.** The OVA's network config was written for `enp0s3` (the interface name it had under VirtualBox), but under Proxmox/KVM the same card comes up as `ens18`. So it was trying to configure an interface that didn't exist. Wrote a static netplan config for the real interface name and disabled cloud-init from regenerating it.
- **DNS resolved to IPv6 only.** The Wazuh package host came back with only IPv6 addresses, and my lab is IPv4-only, so downloads stalled until I forced IPv4 with `curl -4`.

## What's left

- Lock inbound lab access down to only my ThinkPad (right now the firewall allows the whole home LAN subnet in for management — I widened it while debugging and want to tighten it back)
- Actually start the attack -> detect loop on Lin.Security and write custom detection rules for whatever the default ruleset misses, the same way I did with the SQL-injection rule on DVWA
- Add a Windows target with Sysmon, since most real SOC work is Windows
- Add a network IDS (Suricata) feeding the SIEM, so I get network-level detection alongside the host agents
- Upgrade the EliteDesk's RAM — I'm constantly shutting one VM down to run another, and 32 GB would let the attacker, target, SIEM, and firewall all run at once
