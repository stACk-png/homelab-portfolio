# Homelab

This is my self-hosted homelab, running on a single Proxmox box at home. I started it to get real hands-on experience to gain cyber skills also to completely understand what i was learning in my classes,deploying and hardening actual services, running a SIEM, and dealing with the kind of things that break in ways no textbook warns you about. Along with future projects like setting up a blue/red team lab on the server to mimick attacks and learn what they do to get a little experience from it. I also used AI (Claude) as a guide throughout this project to explain concepts I didn't know yet, walk through unfamiliar tools, and help me think through problems but every command was run by me, and every fix was something I understood and applied myself, not copy-pasted blind. 

I'm a sophomore cybersecurity major at Robert Morris University. This repo is where I document what I've built and, more importantly, what's gone wrong and how I figured it out.

## What's running

- **[Wazuh](services/wazuh.md)** — SIEM. Log monitoring, file integrity monitoring, security config checks.
- **[Caddy](services/caddy.md)** — reverse proxy in front of everything.
- **[Prometheus](services/prometheus.md)** — metrics collection.
- **[Grafana](services/grafana.md)** — dashboards for those metrics.
- **[Portainer](services/portainer.md)** — Docker container management UI.
- **[Vaultwarden](services/vaultwarden.md)** — self-hosted password manager.
- **[Scrutiny](services/scrutiny.md)** — hard drive health monitoring.
- **[Pi-hole](services/pihole.md)** — DNS-level ad/tracker blocking, on a separate Raspberry Pi.

Also running Jellyfin and Nextcloud on the same host, but those aren't security-related so I haven't written them up here.

## Architecture

```text
Internet
   |
Home Router -> TP-Link travel router -> HP EliteDesk 800 G2 Mini (Proxmox VE)
   |                                          |
Raspberry Pi 4               +-------------------------+-------------------------+
(Pi-hole)                    |                         |                         |
                        Docker VM                 Jellyfin (LXC)             Nextcloud
                              |
   +----------------+----------------+------------------+
   |                |                |                  |
Caddy         Grafana/Prometheus   Vaultwarden      Wazuh (manager,
(reverse      (metrics)            (secrets)         indexer, dashboard)
 proxy)
```

I access everything remotely over Tailscale, usually from my laptop tethered to my phone. That remote-access setup actually caused one of the trickier problems I ran into see the SSH incident below.

## Incidents

These are writeups of actual problems I ran into and how I tracked them down. I'm keeping these separate from the service docs because they tell the "how I actually solved it" story, not just how something is configured.

- [SSH Agent Lockout](incidents/ssh-agent-lockout.md) — SSH just hung, no error, for a while I had no idea why.
- [Wazuh Disk Exhaustion](incidents/wazuh-disk-exhaustion.md) — thought I'd run out of disk for real, turned out to be a single misbehaving module.
- [Proxmox Memory Reading](incidents/proxmox-memory-reading.md) — Proxmox said the VM was almost out of RAM. It wasn't.
- [Wazuh Agent Enrollment](incidents/wazuh-agent-enrollment.md) — my first agent wouldn't connect, for two completely unrelated reasons.

## Stack

Proxmox VE, Docker, Tailscale, Caddy, Wazuh, Grafana, Prometheus, Vaultwarden, Portainer, Scrutiny

## Status

Still actively working on this. Right now I'm setting up SCA and FIM in Wazuh and adding more agents.
