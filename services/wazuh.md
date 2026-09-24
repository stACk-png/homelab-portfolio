# Wazuh

## What it is

Wazuh is the SIEM (security monitoring platform) running in my homelab. It's made up of three pieces that all run together in Docker:

- **manager** — takes in data from agents, runs it against detection rules, generates alerts
- **indexer** — OpenSearch under the hood, stores all the events and alerts
- **dashboard** — the web UI, for looking at alerts and managing agents

Right now it's deployed all-in-one (all three pieces on one host) since I only have the one machine to run it on. Eventually I want to move it to its own dedicated VM once I upgrade the RAM on the EliteDesk.

## Why I set it up

This is the centerpiece of the security side of the homelab. The goal is log monitoring, file integrity monitoring (FIM), and security configuration checks (SCA) the kind of thing a SOC analyst actually looks at day to day.

## Where it lives

Docker VM, in `~/wazuh-docker/single-node/`. Deployed from Wazuh's official repo, pinned to version 4.9.0 so it doesn't randomly change out from under me.

## Setting it up

The install itself wasn't just `docker compose up`. A few things had to happen first:

- Generated the internal TLS certificates using Wazuh's own cert-generator tool (a one-off Docker container built from `indexer-certs-creator/`)
- Changed every default password before first launch the indexer's `admin` and `kibanaserver` accounts, and the manager's API password. Wazuh ships with well-known default creds, which is a bad look for a security tool, so this was non-negotiable.
- Moved the dashboard off port 443 to 8443, since Caddy already had 443 for the reverse proxy

Five other demo accounts in the indexer (`kibanaro`, `logstash`, `readall`, `snapshotrestore`) still have default passwords. Nothing currently uses them, but it's on my list to clean up.

## Ports

| What | Host port | Purpose |
|---|---|---|
| Manager | 1514 | agent data |
| Manager | 1515 | agent enrollment |
| Manager | 514/udp | syslog |
| Manager | 55000 | manager API |
| Indexer | 9200 | indexer API |
| Dashboard | 8443 | web UI |

## Access

`https://<tailscale-ip>:8443`, over Tailscale only. Not exposed to the internet. Browser will complain about the certificate that's expected, it's signed by Wazuh's own internal CA, not a public one.

## Problems I've run into

**Manager wouldn't start — `Error 5007 - Insecure user password provided`.** The API password I picked didn't meet Wazuh's complexity rules (needs upper, lower, a number, and a special character). Fixed the password, then had to `docker compose down` and `up` again not just restart, since it had already half-crashed.

**Disk filled up completely within two days, with zero agents even connected.** Full writeup in [the incident doc](../incidents/wazuh-disk-exhaustion.md), but short version: the vulnerability-detection module was retrying a failed download every hour and leaving junk behind each time. Disabled it for now.

**First agent wouldn't connect manager address stayed as the literal placeholder text `MANAGER_IP`.** The `WAZUH_MANAGER` environment variable I passed at install time didn't get picked up. Had to edit `/var/ossec/etc/ossec.conf` directly and swap in `127.0.0.1`.

**Agent installed but manager never saw it version mismatch.** The apt repo installs whatever the newest 4.x release is, which was ahead of my 4.9.0 manager. Agents can't be newer than the manager. Had to `apt-cache madison wazuh-agent` to find the exact 4.9.0 build and install that specific version instead.

## Recovery

Config and certs are all in `~/wazuh-docker/single-node/`. Data lives in named Docker volumes. Neither of these is backed up anywhere yet — that's a gap I know about and need to fix before I'd trust this with anything I couldn't afford to lose.

To wipe it and start over: `docker compose down -v` (the `-v` deletes the volumes too, so only do that on purpose).

## What's left

- Get FIM actually configured, not just running
- Same for SCA
- Set up alerting for authentication events
- Deal with those five leftover default accounts
- Add more agents beyond just the Docker VM itself
- Figure out backups
