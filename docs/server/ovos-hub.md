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

All server configuration lives at `~/.config/hivemind-core/server.json`. `hivemind-core listen` takes no command-line flags — edit the file to change any setting. Run `hivemind-core print-config` to dump the effective configuration.

Default configuration (abbreviated):

```json
{
  "binarize": false,
  "allowed_encodings": [
    "JSON-B64", "JSON-URLSAFE-B64", "JSON-B91",
    "JSON-Z85B", "JSON-Z85P", "JSON-B32", "JSON-HEX"
  ],
  "allowed_ciphers": ["CHACHA20-POLY1305", "AES-GCM"],
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
      "port": 5678,
      "ssl": false,
      "cert_dir": "~/.local/share/hivemind",
      "cert_name": "hivemind"
    },
    "hivemind-http-plugin": {
      "host": "0.0.0.0",
      "port": 5679,
      "ssl": false,
      "cert_dir": "~/.local/share/hivemind",
      "cert_name": "hivemind"
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

TLS is enabled per network protocol by setting `ssl` to `true` and pointing `cert_dir`/`cert_name` at the certificate to serve.

## Starting the server

```bash
hivemind-core listen
```

All behaviour comes from `server.json`; there are no flags to pass here.

## Managing clients

Client commands take the node ID as a **positional** argument (omit it and the CLI prompts you to pick one from a table):

```bash
# Add a new satellite
hivemind-core add-client --name "living-room"

# List all registered satellites
hivemind-core list-clients

# Remove a satellite (node ID 2)
hivemind-core delete-client 2

# Rename a client (new name via --name, node ID positional)
hivemind-core rename-client --name "new-name" 2
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
