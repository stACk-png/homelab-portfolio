# Prometheus

## What it is

Metrics collection. Prometheus scrapes stats from a handful of exporters running on the Docker VM and stores them as time-series data.

## What it's watching

- **node-exporter** — host-level stats (CPU, memory, disk)
- **cAdvisor** — per-container resource usage
- **proxmox-exporter** — stats from the Proxmox host itself

## Why I set it up

Before this, "is the VM okay" meant SSHing in and running `free -h` and `df -h` by hand — which is exactly what I did during the disk-exhaustion incident, actually. Prometheus plus Grafana means I can see this stuff without logging in every time, and eventually get alerted before something fills up rather than after.

## Notes

This is the piece that would have caught the disk-exhaustion incident earlier if I'd had alerting rules set up already — right now it's collecting data but I haven't wired up any alerts on top of it. That's the next real gap to close here.
