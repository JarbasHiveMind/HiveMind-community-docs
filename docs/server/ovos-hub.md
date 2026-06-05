# OVOS Skills Hub

The canonical `hivemind-core` hub, backed by an OpenVoiceOS instance. Satellites connect to it and route utterances to OVOS skills.

## Prerequisites

`ovos-core` and `ovos-messagebus` must be running on the same machine before you start `hivemind-core`. For a minimal OVOS install:

```bash
pip install ovos-core ovos-messagebus
```

See the [OVOS documentation](https://openvoiceos.github.io/community-docs) for a complete OVOS setup guide.

## Install

```bash
pip install hivemind-core
```

## Configuration

Server configuration lives at `~/.config/hivemind-core/server.json`. You can also pass all settings as command-line flags to `hivemind-core listen`. The file takes precedence over flags for settings it defines.

Default configuration:

```json
{
  "agent_protocol": {
    "module": "hivemind-ovos-agent-plugin",
    "hivemind-ovos-agent-plugin": {
      "host": "127.0.0.1",
      "port": 8181
    }
  },
  "binary_protocol": {
    "module": null
  },
  "network_protocol": {
    "hivemind-websocket-plugin": {
      "host": "0.0.0.0",
      "port": 5678
    },
    "hivemind-http-plugin": {
      "host": "0.0.0.0",
      "port": 5679
    }
  },
  "policy": {
    "chain": [
      {"module": "hivemind-ovos-agent-policy"}
    ]
  },
  "database": {
    "module": "hivemind-sqlite-db-plugin",
    "hivemind-sqlite-db-plugin": {
      "name": "clients",
      "subfolder": "hivemind-core"
    }
  }
}
```

## Starting the server

```bash
hivemind-core listen
```

Or with explicit flags (override `server.json`):

```bash
hivemind-core listen \
  --host 0.0.0.0 \
  --port 5678 \
  --ssl false \
  --db-backend sqlite
```

## Managing clients

```bash
# Add a new satellite
hivemind-core add-client --name "living-room"

# List all registered satellites
hivemind-core list-clients

# Remove a satellite
hivemind-core delete-client --node-id 2

# Rename a client
hivemind-core rename-client "new-name" --node-id 2
```

See [CLI Reference](../reference/cli.md) for all commands and [Security](../concepts/security.md#permissions) for permission management.

## Adding server-side audio processing

To support mic-satellite and voice-relay, install the audio binary protocol plugin:

```bash
pip install hivemind-audio-binary-protocol
```

Then configure `server.json` to enable it. See [Audio Binary Protocol](audio-binary-protocol.md).

## Systemd service

To run `hivemind-core` as a system service:

```ini
# /etc/systemd/system/hivemind-core.service
[Unit]
Description=HiveMind Core
After=network.target ovos-messagebus.service
Requires=ovos-messagebus.service

[Service]
Type=simple
User=ovos
ExecStart=/home/ovos/.venvs/ovos/bin/hivemind-core listen
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable hivemind-core
sudo systemctl start hivemind-core
```
