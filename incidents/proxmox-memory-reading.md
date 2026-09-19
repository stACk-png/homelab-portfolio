# Incident: Proxmox Memory Reading Looked Like a Crisis, Wasn't

## Summary

Right after getting Wazuh running, I checked the Docker VM's stats in the Proxmox web UI and saw memory usage at 94.83% (7.59 GiB of 8.00 GiB). That looked like the VM was about to run out of memory outright, and given I'd just added a whole SIEM stack on top of the existing services, it seemed like a plausible explanation. I almost started planning around it reallocating RAM, or holding off on installing agents until I upgraded hardware.

## Symptoms

- Proxmox's own VM summary page showed memory usage pinned at 94.83%
- No errors or actual performance problems happening at the same time the VM was responsive, nothing was crashing

## Investigation

Before assuming the number was accurate, I checked memory from inside the VM itself, not just from Proxmox's outside view:

```
free -h
```

This showed a completely different picture: `used` was around 3.3-3.4 GiB, `available` was 4.4-4.5 GiB, and swap usage was effectively zero (1 MiB, just rounding noise).

## Root Cause

Proxmox's memory percentage reflects total memory the guest OS has touched, including anything Linux is using for disk cache — and Linux aggressively grabs spare RAM for caching because it's free performance, not because it needs to keep it. That cached memory shows up as "used" from Proxmox's point of view, but it's instantly reclaimable the moment something else actually needs it. `free -h` inside the VM breaks this apart explicitly (`used` vs `buff/cache` vs `available`), which is why it told a different story than the outside view.

The real signal for actual memory pressure is `available` shrinking toward zero **and** swap usage climbing at the same time neither was happening here.

## Fix

No fix needed — there was no actual problem. I just stopped treating Proxmox's dashboard percentage as the ground truth and used `free -h` inside the guest instead.

## Prevention / Lessons

- Host-level dashboards (Proxmox, and probably others like it) often report memory in a way that includes reclaimable cache as "used." That number alone isn't enough to tell if a VM is actually under pressure.
- The real check is inside the guest: `available` memory and swap usage together, not a single percentage from outside.
- Worth checking this before making any resource decision (upgrading hardware, killing services, reallocating RAM) a scary-looking dashboard number is not the same thing as a real resource constraint.
