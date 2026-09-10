# Moving agents and callers onto this hub

Both hubs can drive the same machine at once: an agent is a plain process holding one outbound
websocket, so a second one against the new hub costs nothing and changes nothing about the first.
Nothing has to be cut over in a single window.

## 1. Get access to the UI

Add your Authentik user to the group **`service-controller-users`**, then open
https://service-controller.jannekeipert.de. Anyone not in that group is stopped at the outpost.

## 2. Register the agent, then point a second unit at it

Create an **AGENT** client in the UI with the device id the machine should answer to. The token is
shown **once**, together with the websocket URL to paste. If you lose it, rotate — there is no
recovery.

Keep the existing systemd unit exactly as it is. Add a parallel one.

Two things about where the token goes. It must not be an inline `Environment=` line: `systemctl
cat` is readable by any local user. It must not be `argv[2]` either, for the same reason via `ps`.
Put it in a root-only file:

```bash
sudo install -m 600 -o root -g root /dev/null /etc/service-controller-cloud.env
echo 'SERVICE_CONTROLLER_WEBSOCKET_URL=wss://service-controller.jannekeipert.de/service_controller?token=<token>' \
  | sudo tee /etc/service-controller-cloud.env > /dev/null
```

The agent also has to be a version that reads that variable — `home-audio-detection`'s
`service-controller/main.py` does, and redacts the token when it logs the endpoint. Older copies
on the Pis take the URL as an argument only and print it verbatim, so copy the current one
alongside the existing script rather than replacing it:

```bash
scp service-controller/main.py <pi>:/home/<user>/service-controller/main_cloud.py
```

```ini
# /etc/systemd/system/service-controller-cloud.service
[Unit]
Description=Service Controller Listener (cluster hub)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
# Match the existing unit — it needs to run systemctl, so it is usually root.
User=root
WorkingDirectory=/home/<user>/service-controller
EnvironmentFile=/etc/service-controller-cloud.env
Environment=PYTHONUNBUFFERED=1
ExecStart=/usr/bin/python3 /home/<user>/service-controller/main_cloud.py <device-id>
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now service-controller-cloud
```

Confirm the journal shows `token=<redacted>` and not the token itself:

```bash
sudo journalctl -u service-controller-cloud -n 5
```

The agent should appear as connected in the UI within a couple of seconds. To prove the round
trip without disturbing anything, send **start** for a unit that is already running — systemd
treats that as a no-op, and the agent's journal will still show it executing the command. The device id it sends
is ignored in favour of the one its token belongs to, so a typo in `ExecStart` is now harmless —
but the device id you typed when creating the client is not, and that one still has to match what
you expect to command.

## 3. Move the callers

Each thing that *issues* commands gets its own **CALLER** client, so a leaked credential can be
traced and revoked without touching anything else. Old callers keep working against the old hub
until you move them; when you move one, it goes straight onto its own token:

```bash
curl -X POST -H "Authorization: Bearer <token>" \
  "https://service-controller.jannekeipert.de/execute_action?deviceId=led-matrix&service=led-matrix&action=restart"
```

Callers that cannot set a header can use `?token=<token>` instead. Same parameters as the old hub
otherwise.

Check `deliveredTo` in the response: it is the number of agent connections that actually got the
command. The old hub always answered "Action Send" whether or not anything was listening.

## 4. Retire the old path

Once every caller is on the new host, disable the old agent unit on each device. The old hub can
then stop serving `/service_controller`; `/audio` and `/deviceV2` stay where they are, along with
everything that has to reach a device on the home LAN.

## Rotating and revoking

Both take effect immediately and disconnect the agent — its old credential is gone, and leaving
the session open would let a revoked token keep working until the next reconnect. Revoked clients
stay in the list on purpose: the audit trail outlives the credential.
