# Homelab — Security Monitoring & Infrastructure Project

A self-hosted homelab built on Proxmox, focused on hands-on defensive security skills: SIEM deployment, log/FIM/SCA monitoring, network segmentation, and infrastructure hardening. Built as both a learning environment and a portfolio project supporting a SOC Analyst career path (HTB Academy SOC Analyst path → CDSA certification).

## Highlights

- Deployed and hardened a **Wazuh SIEM stack** (manager, indexer, dashboard) in Docker, including generating internal PKI/TLS certificates, rotating all default credentials, and resolving a port conflict with an existing reverse proxy
- Diagnosed and resolved a **disk-exhaustion incident** caused by a misbehaving security module — ruled out RAM/hardware limits, isolated the true root cause to a specific misconfigured feed updater, and fixed it without adding hardware
- Diagnosed and resolved an **SSH lockout incident** — traced a silent authentication hang through client/server logs, network path analysis (Tailscale relay vs. direct P2P), and ultimately to a stale cached credential, without ever weakening the account's security posture
- Hardened SSH access (key-only auth, `IdentitiesOnly`, per-host key scoping) and verified server-side hardening (password auth disabled, root login checks)
- Built and operates a segmented Docker environment: reverse proxy (Caddy), metrics stack (Prometheus/Grafana), secrets management (Vaultwarden), and the SIEM stack, isolated into separate Docker networks
- Maintains structured documentation for every service (architecture, configuration, failure modes, recovery steps) as living infrastructure docs, not just setup notes

## Architecture

```text
Internet
   |
Home Router → TP-Link travel router → HP EliteDesk 800 G2 Mini (Proxmox VE)
                                              |
                    +-------------------------+-------------------------+
                    |                         |                         |
              Docker VM                 Jellyfin (LXC)             Nextcloud
                    |
   +----------------+----------------+------------------+
   |                |                |                  |
Caddy         Grafana/Prometheus   Vaultwarden      Wazuh (manager,
(reverse      (metrics)            (secrets)         indexer, dashboard)
 proxy)
```

Remote access via **Tailscale** (direct peer-to-peer), used from an off-network laptop over a mobile hotspot during this project — see [incidents/ssh-agent-lockout.md](incidents/ssh-agent-lockout.md) for a real-world case where that remote-access path caused a genuinely tricky diagnostic problem.

## Services

Each service has its own documentation page covering purpose, configuration, dependencies, ports, common failure modes, and recovery steps:

- [Wazuh (SIEM)](services/wazuh.md)
- More services to be added as documentation is ported over

## Incidents

Real troubleshooting writeups — symptom, investigation, root cause, fix, and prevention. Kept separate from service docs because these tell the "how I actually solved it" story that a static config reference doesn't capture:

- [SSH Agent Lockout](incidents/ssh-agent-lockout.md) — a silent SSH hang traced through client logs, server logs, and network-path analysis to a stale cached passphrase
- [Wazuh Disk Exhaustion](incidents/wazuh-disk-exhaustion.md) — a full disk that looked like a hardware limitation, actually caused by a single misbehaving module

## Stack

Proxmox VE · Docker · Tailscale · Caddy · Wazuh · Grafana · Prometheus · Vaultwarden · Portainer · Scrutiny

## Status

Actively maintained, in progress. Current focus: Wazuh SCA/FIM configuration and expanding agent coverage to additional hosts.
