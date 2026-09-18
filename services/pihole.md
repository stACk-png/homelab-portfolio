# Pi-hole

## What it is

DNS-level ad and tracker blocking, running on a Raspberry Pi 4 — separate hardware from the main EliteDesk/Proxmox box.

## Why it's on its own hardware

Mostly just how it started out — the Pi was already sitting around, and DNS is the kind of thing you don't want going down because something else on the same machine crashed. Keeping it on separate hardware means a Docker restart or a Proxmox reboot doesn't take DNS down with it.

## Status

Still more of an experiment than a finished setup. It blocks ads/trackers for devices pointed at it, but I haven't done much beyond the basic install — no redundancy, no real tuning of blocklists yet.

## What's left

- Look into running a second instance (even just Pi-hole in Docker on the main box as a backup resolver) so DNS doesn't have a single point of failure
- Actually review and customize the blocklists instead of using defaults
- Point more of the network at it consistently instead of it being half set up
