# Migrating agents off the DeviceHomeHub hub

The point of this service is that agents can move one at a time, with no window where a device is
uncontrollable. Both hubs can drive the same machine at once: an agent is a plain process holding
one outbound websocket, so a second one against the new hub costs nothing and changes nothing
about the first.

## 1. Bring the new hub up

DNS `service-controller.jannekeipert.de` → `195.201.171.111` (**A only** — an AAAA record breaks
the ACME HTTP-01 challenge on this IPv4-only cluster). Then check:

```bash
curl https://service-controller.jannekeipert.de/        # Service Controller online
```

## 2. Add a second agent instance on the device

Keep the existing unit exactly as it is. Add a parallel one pointed at the new hub — same device
id, new URL:

```ini
# /etc/systemd/system/service-controller-cloud.service
[Unit]
Description=Service controller agent (cluster hub)
After=network-online.target

[Service]
User=<same user as the existing unit>
WorkingDirectory=/home/<user>/service-controller
Environment=SERVICE_CONTROLLER_WEBSOCKET_URL=wss://service-controller.jannekeipert.de/service_controller?token=<agent token>
ExecStart=/usr/bin/python3 main.py <device-id>
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now service-controller-cloud
```

Confirm the hub sees it:

```bash
curl "https://service-controller.jannekeipert.de/agents?password=<control password>"
```

`connections` counts websockets, not devices — during the overlap a device that also talks to the
old hub still shows `1` here, because the old hub's connection is not ours to see.

## 3. Move the callers

Repoint whatever issues commands (Homebridge scenes, iOS shortcuts, cron) from
`https://valorantcrazyclips69.com/execute_action` to
`https://service-controller.jannekeipert.de/execute_action`. Same parameters. If the control
password differs between the two hubs, update it at the same time.

Check `deliveredTo` in the response: it is the number of agent connections that actually got the
command. The old hub always answered "Action Send" whether or not anything was listening.

## 4. Retire the old path

Once every caller is on the new host and nothing has regressed, disable the old agent unit on each
device. The old hub can then stop serving `/service_controller`; `/audio` and `/deviceV2` stay
where they are, along with everything that has to reach a device on the home LAN.

## Rotating the agent token

Rotating disconnects every agent until each one's URL is updated, so do it deliberately: reseal
the secret (see the README), let ArgoCD sync, then update each unit and restart it.
