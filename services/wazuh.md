---
type: service
category: security monitoring
status: online
host: Docker VM
dashboard: https://100.94.110.39:8443
---

# Wazuh

> **Purpose:** Centralized security monitoring — SCA, FIM, and log/authentication event monitoring
> **Status:**  Operational (deployed, agents not yet installed)
> **Category:** Security / Monitoring
> **Dashboard:** `https://100.94.110.39:8443` (Tailscale only, not proxied through Caddy yet)

---

##  Overview

Wazuh is an open-source security monitoring platform made up of three components running together in this deployment:

- **Wazuh manager** — receives data from agents, applies detection rules/decoders, generates alerts
- **Wazuh indexer** — OpenSearch-based datastore that holds all events/alerts
- **Wazuh dashboard** — web UI (Kibana-style) for viewing alerts, configuring rules, and managing agents

Deployed as an all-in-one, single-node stack in its own isolated Docker Compose project on the Docker VM — separate from the existing Grafana/Prometheus/Caddy monitoring stack.

---

##  Purpose

- Central security monitoring across homelab hosts
- Security Configuration Assessment (SCA)
- File Integrity Monitoring (FIM)
- Authentication and log event monitoring/alerting
- Core piece of the security hardening + Wazuh project phase
- Portfolio project demonstrating centralized security monitoring

---

##  Architecture

```text
Homelab hosts (agents — none installed yet)
        |
        | agent data (1514/1515)
        v
Docker VM
        |
        +-- wazuh.manager   (rules, alerts, API)
        |         |
        |         v
        +-- wazuh.indexer   (OpenSearch — stores events/alerts)
        |         |
        |         v
        +-- wazuh.dashboard (web UI, port 8443)
```

Isolated into its own Docker network — not sharing `monitoring_default` or `proxy` with the existing stack.

---

## Host

**Docker VM** — deployed via the official `wazuh/wazuh-docker` repo, pinned to tag `v4.9.0`.

Location on disk: `~/wazuh-docker/single-node/`

---

## Status

- **Deployed:** all-in-one, single-node, in Docker
- **Isolated from existing stack:** own Compose project, own network
- **Migration plan:** intended to move to a dedicated VM/LXC after a hardware upgrade. Data lives in named Docker volumes specifically to make that migration straightforward later (stop container → copy volume data → start on new host)
- **Agents:** first agent installed and active — the Docker VM itself, monitoring its own host. Confirmed enrolled, connected, and visible in the manager (`manage_agents -l`) and dashboard.

**Resource status (confirmed after the vulnerability-detection incident above):** this incident initially looked like genuine resource exhaustion and raised the question of whether to pause the project until the planned RAM upgrade. Once root-caused and fixed, confirmed this was *not* the case:
- Docker VM disk: 12 GB available, holding steady (post-fix)
- Docker VM RAM: ~4.5 GB available, 0 swap pressure — unaffected throughout, never actually the bottleneck
- Proxmox host RAM: ~4.1 GB available across everything sharing the EliteDesk's 16 GB total — tighter, but not currently blocking anything

Conclusion: no pause and no hardware upgrade were needed to resolve this. The eventual RAM upgrade + migrating Wazuh to its own VM/LXC is still the right long-term plan for headroom, but it's a planned improvement, not something the project is currently blocked on.

---

## Configuration

- Internal TLS certs generated via Wazuh's official `indexer-certs-creator` tool (built as a local Docker image, run as a one-off container against `single-node/config/certs.yml`, output written to `single-node/config/wazuh_indexer_ssl_certs/`)
- Default credentials replaced before first launch:
  - Indexer `admin` account — password + bcrypt hash changed (hash regenerated using the indexer image's own `hash.sh` tool, kept in sync between `internal_users.yml` and `docker-compose.yml`)
  - Indexer `kibanaserver` account — same process
  - Manager `API_PASSWORD` — plaintext value in `docker-compose.yml`, no separate hash file; must meet Wazuh's complexity rule (8–32 chars, upper + lower + number + special character) or the manager container fails to start with `Error 5007 - Insecure user password provided`
- **Known outstanding item:** five demo indexer accounts (`kibanaro`, `logstash`, `readall`, `snapshotrestore`) still have default password hashes in `internal_users.yml`. Not referenced by this compose file, so not currently exploitable via this deployment, but flagged for future cleanup.

---

## Dependencies

- Docker / Docker Compose on the Docker VM
- Sufficient free RAM/disk on the Docker VM (checked before deployment: ~6.4 GB available, 0 swap pressure, 19G free on root — comfortable headroom for a single-node deployment)

---

##  Ports

| Service | Host Port | Container Port | Purpose |
|---|---|---|---|
| Manager | 1514 | 1514 | Agent data (encrypted) |
| Manager | 1515 | 1515 | Agent enrollment |
| Manager | 514/udp | 514/udp | Syslog |
| Manager | 55000 | 55000 | Manager API |
| Indexer | 9200 | 9200 | Indexer API |
| Dashboard | **8443** | 5601 | Web UI (HTTPS) |

**Note:** Dashboard was remapped from the default host port 443 → 8443, because Caddy already owns 443 on this VM for the `.home.lab` reverse-proxy setup. Avoided a port-binding conflict at launch.

---

## How it's accessed

- Directly via Tailscale: `https://100.94.110.39:8443`
- Browser will show a certificate warning — expected, since the cert chain is signed by Wazuh's own internally-generated root CA, not a public CA. Not exposed to the public internet.
- Not yet proxied through Caddy — could add a `wazuh.home.lab` Caddy site block later for a friendlier internal URL.

---

## How it's monitored

- Not yet instrumented into the existing Prometheus/Grafana stack
- Container health checked with `docker compose ps` inside `~/wazuh-docker/single-node/`

---

##  Common Failure Modes / Troubleshooting

- **`Error 5007 - Insecure user password provided`** (manager container exits on startup): the `API_PASSWORD` value in `docker-compose.yml` doesn't meet Wazuh's complexity requirements. Fix the password, then `docker compose down && docker compose up -d` (don't just `up -d` again on top of a crashed container — start clean)
- **Port bind failure on dashboard startup:** something else on the host already holds port 443 (in this case, Caddy). Remap the dashboard's host port in `docker-compose.yml`
- **Indexer won't start / cert errors:** certs missing or filenames don't match what's referenced in `docker-compose.yml`. Regenerate via `indexer-certs-creator` and confirm filenames against the compose file's volume mounts
- Insecure file permission warnings for indexer JDK binaries on startup (`should be 0600`) are expected noise from the base image and not a real problem
- **Root disk fills up within a day or two of a fresh deployment, with `database or disk is full` errors in `ossec.log` and containers failing with `no space left on device`:** caused by the **vulnerability-detection module**'s feed updater (`vd_updater` / `vd` inside the `wazuh_queue` volume, using an embedded RocksDB store). On this deployment, the feed download/write repeatedly failed (`Failed writing received data to disk/application`) but didn't clean up after itself — each retry (every `<feed-update-interval>`, default 60m) left more orphaned data behind, growing to ~19 GB in under 48 hours with zero agents connected. This looked at first like genuine resource exhaustion (disk *and* RAM were both suspected), but was actually a single misbehaving module, unrelated to agent count or real indexed data.
  - **Diagnose:** `docker system df -v | grep -i wazuh` to check per-volume sizes; if `wazuh_queue` is unexpectedly large, break it down further with `sudo bash -c 'du -sh /var/lib/docker/volumes/single-node_wazuh_queue/_data/*/ 2>&1' | sort -rh` (note: `sudo du ... *` alone fails silently because the shell glob expands before `sudo` takes effect on a root-owned directory — must wrap in `sudo bash -c '...'`)
  - **Fix:** stop the manager (`docker compose stop wazuh.manager`), clear the runaway data (`sudo bash -c 'rm -rf .../vd_updater/* .../vd/*'`), restart it
  - **Prevent recurrence:** disabled the module by editing `single-node/config/wazuh_cluster/wazuh_manager.conf`, setting `<enabled>no</enabled>` inside the `<vulnerability-detection>` block (leave the separate `<indexer><enabled>yes</enabled>` block untouched — different setting, controls the indexer connection itself).
    - **Important:** `docker compose restart wazuh.manager` was NOT sufficient to apply this — the config file showed `<enabled>no</enabled>` correctly both on disk and inside the container, but the module kept running anyway and `wazuh_queue` grew again (~5GB in under an hour). Confirmed fix required a full container recreate instead: `docker compose stop wazuh.manager && docker compose rm -f wazuh.manager && docker compose up -d wazuh.manager`. After that, manager logs explicitly confirmed: `INFO: Vulnerability scanner module is disabled.` `ps aux` inside the container is not a useful check for this — the scanner isn't a separate OS process, it runs inside `wazuh-modulesd`, so check the logs instead: `docker compose logs -f wazuh.manager | grep -i vuln`
  - Revisit re-enabling this once agents are in place and the feed-download issue is understood (may be version-specific; worth checking Wazuh's GitHub issues before re-enabling)
- **Agent fails to start: `ERROR (4112): Invalid server address found: 'MANAGER_IP'`, `ERROR (1215): No client configured`:** the `WAZUH_MANAGER='127.0.0.1'` environment variable passed at `apt install` time wasn't picked up correctly, leaving the literal placeholder text `MANAGER_IP` in `ossec.conf` instead of the real address. Fix: edit the config directly rather than relying on the install-time variable — `sudo sed -i 's/MANAGER_IP/127.0.0.1/' /var/ossec/etc/ossec.conf`, then `sudo systemctl start wazuh-agent`.
- **Agent starts locally (`active (running)`) but never appears on the manager — log shows a version-mismatch message ("agent version should be older or equal to the manager"):** the Wazuh apt repo (`https://packages.wazuh.com/4.x/apt/`) serves the *latest* 4.x release by default, not the version matching this manager (4.9.0). A plain `apt install wazuh-agent` grabs whatever's newest, which will be newer than the 4.9.0 manager — and Wazuh agents can't be newer than their manager. Fix: purge the mismatched agent (`apt remove --purge wazuh-agent -y`), find the exact available version string with `apt-cache madison wazuh-agent | grep 4.9.0`, then install that exact pinned version: `apt install wazuh-agent=4.9.0-1` (adjust suffix to match what madison shows). Re-enroll and start as normal afterward.

---

## Recovery

- Config and certs live in `~/wazuh-docker/single-node/` — back this whole directory up (not yet included in a formal backup routine as of this writing — known gap)
- Data lives in named Docker volumes (`wazuh-indexer-data`, `wazuh_logs`, etc.) — also not yet in a backup routine
- To fully tear down and rebuild from scratch: `docker compose down -v` ( the `-v` flag deletes the named volumes too — only use this if you intend to lose all indexed data and start over)

---

##  Security Controls

- Not exposed to the public internet — reachable only via Tailscale (or LAN) to the Docker VM
- Default indexer and manager API credentials replaced before first launch
- Internal component-to-component communication secured via generated TLS certificates
- Isolated into its own Docker Compose project/network, separate from the existing monitoring/reverse-proxy stack — a problem here shouldn't cascade into Grafana/Prometheus/Caddy

---

##  Next Steps

- [x] Install a Wazuh agent (Docker VM itself) — done, confirmed active and reporting
- [ ] Verify FIM (File Integrity Monitoring) is active and tune watched directories
- [ ] Verify SCA (Security Configuration Assessment) is running and review findings
- [ ] Install agents on additional homelab hosts
- [ ] Configure authentication/security event monitoring and alerting
- [ ] Rotate/disable the remaining default demo indexer accounts
- [ ] Add Wazuh data/config directory to the backup routine
- [ ] Consider a Caddy site block for a friendlier internal dashboard URL
- [ ] Investigate why the vulnerability-detection feed update fails, and re-enable the module once resolved (currently disabled — see Troubleshooting)
