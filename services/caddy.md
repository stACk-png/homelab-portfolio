# Caddy

## What it is

Reverse proxy sitting in front of the services on the Docker VM. Everything that needs a clean URL goes through Caddy instead of people (or me) having to remember which port each service runs on.

## Why I set it up

Handles HTTPS automatically and keeps me from having to expose a dozen different ports directly. One entry point, one place to manage TLS.

## Ports

80 and 443 on the Docker VM.

## Notes

When I added Wazuh, its dashboard originally wanted port 443, which conflicted with Caddy. I moved Wazuh's dashboard to 8443 instead of touching Caddy's config, since Caddy is already fronting everything else.

Haven't yet set up a Caddy site block for Wazuh's dashboard, so right now it's reached directly on 8443 instead of through a clean `wazuh.home.lab`-style URL. On the list.
