# Incident: Wazuh Disk Exhaustion from a Misbehaving Feed Updater

## Summary

Within roughly 48 hours of standing up a fresh Wazuh SIEM deployment — before any agents had even been installed the Docker host's root filesystem filled to 100%, causing an unrelated container (cAdvisor) to fail with `no space left on device`. The initial hypothesis was that the host had simply reached its hardware limits and needed a RAM/disk upgrade before the project could continue. Investigation showed this wasn't a hardware problem at all it was a single misconfigured module retrying a failed operation without cleaning up after itself.

## Symptoms

- `OCI runtime exec failed ... no space left on device` from an unrelated container
- `df -h /` showing 100% usage, 0 bytes available
- Manager logs full of cascading errors: `SQLite: database or disk is full`, `Bad response from database`, `Cannot save Syscollector`
- Disk usage climbed again within an hour even after a first round of cleanup

## Investigation

1. Confirmed the disk was genuinely full (`df -h /`), not a false reading like an earlier, unrelated Proxmox memory-percentage scare that turned out to just be reclaimable disk cache being counted as "used."
2. Ran `docker system df` total Docker-reported usage didn't add up to the actual root filesystem usage, indicating space was being consumed outside Docker's own accounting (this turned out to be a red herring in one sense and a real clue in another see below).
3. Checked `/var/lib/docker` vs `/var/lib/containerd` directly — found ~12 GB sitting in containerd's own storage, distinct from Docker's reported numbers. This was accumulated build/image layer data and was cleaned up with `docker system prune -a`, recovering about 1 GB not enough to explain the full gap, so the investigation continued.
4. Checked for orphaned Docker volumes (`docker volume ls -f dangling=true`). This flagged several volumes as unused, including ones belonging to actively running Grafana and Prometheus containers **a false positive**, confirmed by directly inspecting the live containers' actual mounts (`docker inspect <container> --format '{{ range .Mounts }}...'`). This was an important check: blindly trusting the dangling flag and running `docker volume prune` would have deleted live dashboard and metrics data. Only volumes independently confirmed to be unreferenced by any container (stopped or running) were removed.
5. After that cleanup, the real remaining hog was found using `docker system df -v`, which broke down per-volume sizes: a volume named `wazuh_queue` was 19+ GB, far larger than expected for a fresh deployment with zero connected agents.
6. Broke that volume down further (`du` per subdirectory) and found the bulk of it under `vd` and `vd_updater` — Wazuh's **vulnerability-detection module** and its feed-download cache (backed by an embedded RocksDB store).
7. Correlated with manager logs showing `Failed writing received data to disk/application` for `vulnerability_feed_manager` — the module was attempting to download a CVE feed database, failing partway through, and not cleaning up the partial data before retrying on its configured interval (`<feed-update-interval>60m</feed-update-interval>`). Each failed retry left more orphaned data behind.

## Root Cause

Wazuh's vulnerability-detection module was configured to retry a failing feed download every 60 minutes, and the failure mode did not clean up partial/orphaned data before retrying. With zero agents connected and no other explanation for such rapid, sustained growth, this single module accounted for the entire incident.

## Fix

1. Stopped the manager container cleanly before touching its data (`docker compose stop wazuh.manager`) to avoid corrupting a file it was actively writing
2. Removed the orphaned feed-cache data directly from the Docker volume's backing directory on the host
3. Disabled the module in the manager's configuration (`<enabled>no</enabled>` under `<vulnerability-detection>`)
4. **Important gotcha:** a simple `docker compose restart` did *not* actually stop the module from running, despite the config file showing the correct value both on disk and inside the running container disk usage grew again within an hour. Confirming via logs (`docker compose logs -f wazuh.manager | grep -i vuln`) showed the module was still active. A full container recreate (`stop` → `rm` → `up -d`, rather than an in-place `restart`) was required before the manager's startup log finally confirmed: `Vulnerability scanner module is disabled.` Disk usage was flat on every subsequent check afterward.

## Prevention / Lessons

- A resource symptom (disk full) doesn't necessarily mean a resource problem (need more disk/RAM) it's worth ruling out a runaway process before reaching for a hardware upgrade.
- `docker volume ls -f dangling=true` is not fully reliable — it flagged actively-mounted volumes as unused. Always verify with `docker inspect <container>` against every relevant container before deleting anything it flags.
- Shell glob expansion happens *before* `sudo` takes effect — `sudo du /root-owned-path/*` can silently fail to match anything if the invoking user's shell can't read the directory to expand the glob. Wrapping the whole command in `sudo bash -c '...'` avoids this.
- A configuration change that looks correct on disk and inside a running container is not proof it's actually been applied some daemons don't fully re-read all settings on an in-place `restart`. Confirm via runtime logs, and prefer a full container recreate when a setting genuinely needs to change process behavior.
- `ps aux` inside a container is not always the right tool to check whether a specific feature is active some modules (like this one) run as internal threads inside a parent daemon rather than as separate OS processes. Application-level logs are the more reliable signal.
