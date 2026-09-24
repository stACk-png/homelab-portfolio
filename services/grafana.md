# Grafana

## What it is

The dashboard layer on top of Prometheus. Prometheus collects the numbers, Grafana is where I actually look at them.

## Why I set it up

Same reason as Prometheus, really a real dashboard is a lot faster to glance at than SSHing in and running commands every time I want to check on something.

## Notes

Dashboards are mostly the default ones that come with the exporters I'm using. Haven't built anything custom yet. Would like to eventually pull in Wazuh alert data here too, so I have one place to check instead of switching between the Wazuh dashboard and Grafana.
