# Vaultwarden

## What it is

A self-hosted, lightweight implementation of the Bitwarden password manager server.

## Why I set it up

Rather than trusting a third-party password manager's cloud, I keep this on my own infrastructure. Used it to actually generate and store the new credentials during the Wazuh credential rotation (indexer admin password, manager API password, etc.) instead of writing them down anywhere less secure.

## Notes

This one holds genuinely sensitive data, so it's a good candidate to prioritize once I actually set up backups for anything on this homelab.
