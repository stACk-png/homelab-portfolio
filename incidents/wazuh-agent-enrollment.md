# Incident: First Wazuh Agent Wouldn't Enroll — Two Separate Bugs, Not One

## Summary

Once the Wazuh manager was stable, I installed my first agent on the Docker VM itself, monitoring its own host, as a first test before rolling out to other machines. The agent installed and its local service showed as running, but the manager never saw it. Getting it actually connected took finding and fixing two unrelated problems, one after the other, that both happened to produce the same "not connected" symptom.

## Symptoms

- `sudo systemctl status wazuh-agent` showed `active (running)` — the local agent process looked healthy
- The manager's own agent list (`manage_agents -l`) never showed the new agent at all

## Investigation Bug 1

Checked the agent's own log for what it was actually doing:

```
sudo tail -n 20 /var/ossec/logs/ossec.log
```

This showed the real problem immediately:

```
ERROR: (4112): Invalid server address found: 'MANAGER_IP'
ERROR: (1215): No client configured. Exiting.
```

The literal placeholder text `MANAGER_IP` was sitting in the config where a real IP address should have been. I'd passed `WAZUH_MANAGER='127.0.0.1'` as an environment variable at install time, expecting the installer to write that value into `ossec.conf` it didn't take.

### Fix

Edited the config directly instead of relying on the install-time variable:

```
sudo sed -i 's/MANAGER_IP/127.0.0.1/' /var/ossec/etc/ossec.conf
sudo systemctl start wazuh-agent
```

## Investigation — Bug 2

Agent started locally without error this time, but the manager still didn't show it. Checked the agent log again and found a different message: something to the effect of the agent's version needing to be older than or equal to the manager's version.

### Root Cause

The apt repo I'd added (`https://packages.wazuh.com/4.x/apt/`) serves whatever the *latest* 4.x release currently is not specifically version 4.9.0, which is what my manager was running. A plain `apt install wazuh-agent` had pulled in a newer agent version than the manager, and Wazuh enforces that agents can't be ahead of their manager (they can be older, just not newer).

### Fix

```
sudo systemctl stop wazuh-agent
sudo apt remove --purge wazuh-agent -y
apt-cache madison wazuh-agent | grep 4.9.0
sudo apt install wazuh-agent=4.9.0-1
```

(the exact version string came from what `apt-cache madison` actually listed, not guessed)

Enrolled and started cleanly after that — confirmed both from the agent's own log and from the manager's `manage_agents -l` finally showing it.

## Prevention / Lessons

- An install-time environment variable isn't guaranteed to land in the config the way you'd expect worth checking the actual config file after install rather than assuming it worked.
- A package repo pinned to a major version range (`4.x`) doesn't mean it's pinned to *your* version it'll serve the newest release in that range. Manager and agent versions need to match (or agent ≤ manager), which means checking `apt-cache madison` and installing an exact version explicitly, not just `apt install <package>`.
- Two different bugs producing the same visible symptom ("agent installed but manager doesn't see it") is a good reminder to actually read the agent's own log each time, rather than assuming a fix that solved the first problem also solved the second.
