# OVOS Skills Server

This is the setup that turns your hive into a real voice assistant — weather, timers,
music, home control, hundreds of skills. Behind hivemind-core sits a full
[OpenVoiceOS](https://openvoiceos.github.io/community-docs) install, and every satellite
that connects gets the whole thing: ask any of them a question and an OVOS skill answers.
It's the default flavour, and the one most people want when they picture "a private
smart speaker I actually own." hivemind-core is the gateway; OVOS is the brain behind it.

!!! abstract "In a nutshell"
    - A `hivemind-core` server whose agent talks to a local `ovos-core` / `ovos-messagebus` on `127.0.0.1:8181`.
    - Every setting lives in `~/.config/hivemind-core/server.json` — `hivemind-core listen` takes no flags.
    - Manage satellites with the `hivemind-core` CLI (node IDs are positional).

---

## Prerequisites

`ovos-core` and `ovos-messagebus` must be running on the same machine before you start `hivemind-core`. For a minimal OVOS install:

```bash
pip install ovos-core ovos-messagebus
```

See the [OVOS documentation](https://openvoiceos.github.io/community-docs) for a complete OVOS setup guide.

---

## Install

```bash
pip install hivemind-core
```

---

## Configuration

All server configuration lives at `~/.config/hivemind-core/server.json`. `hivemind-core listen` takes no command-line flags — edit the file to change any setting. Run `hivemind-core print-config` to dump the effective configuration.

Don't be daunted by the block below — for a standard OVOS server you barely touch it. It's
printed in full so you can see the defaults, but the one line that matters for this setup
is the `agent_protocol` block pointing at your OVOS messagebus. Everything else already
does the right thing:

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
      "port": 8181,
      "connection_timeout": 10
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

!!! warning "The HTTP port `5679` is a deliberate override"
    The WebSocket plugin's source default port is **5678** (correct above). The HTTP
    plugin's *source* default port is **also 5678** — the `5679` you see in this
    `server.json` is an explicit override so the two listeners don't collide. If you
    copy the `hivemind-http-plugin` block but drop the `port` key, HTTP falls back to
    its `5678` default and clashes with the WebSocket listener. Always keep `port: 5679`
    (or any free port) on the HTTP block.

The `agent_protocol` block points at the OVOS agent plugin. Its keys are `host`
(`127.0.0.1`, the OVOS messagebus host), `port` (`8181`, the messagebus port), and
`connection_timeout` (`10` seconds, how long to wait for the messagebus before giving
up). The entry-point names above — `hivemind-ovos-agent-plugin`,
`hivemind-ovos-agent-policy`, `hivemind-websocket-plugin`, `hivemind-http-plugin`, and
`hivemind-sqlite-db-plugin` — are the strings `hivemind-core` resolves at startup.

TLS is enabled per network protocol by setting `ssl` to `true` and pointing `cert_dir`/`cert_name` at the certificate to serve.

!!! tip "More transports than WebSocket + HTTP"
    The two `network_protocol` entries above are the defaults, but the carrier is
    pluggable — MQTT and an experimental Usenet transport also exist. See
    [Transports](transports.md) for the full status table.

---

## Starting the server

```bash
hivemind-core listen
```

All behaviour comes from `server.json`; there are no flags to pass here.

---

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

---

## Adding server-side audio processing

To support mic-satellite and voice-relay, install the audio binary protocol plugin:

```bash
pip install hivemind-audio-binary-protocol
```

Then configure `server.json` to enable it. See [Audio Binary Protocol](audio-binary-protocol.md).

---

## Systemd service

Once it works from the terminal, you'll want it to come back on its own after a reboot.
The unit below does that — the one detail worth copying carefully is the ordering: OVOS's
messagebus has to be up before hivemind-core tries to reach it, which the `After=` /
`Requires=` lines guarantee.

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

---

## Other agent flavors

Beyond the OVOS, [persona](persona-server.md), and [A2A](a2a-server.md) agent plugins, two
more agent-plugin flavors exist for advanced or experimental setups. Both are
**alpha / unpublished — install from the repo and read its README**.

??? note "Advanced: other agent flavors"
    **localhive** ([`hivemind-localhive-agent-plugin`](https://github.com/JarbasHiveMind/hivemind-localhive-agent-plugin))
    — an "isolated-skills" agent. It runs OVOS skills *in-process* (no separate
    `ovos-core` / `ovos-messagebus`), but each skill only ever sees its own messages.
    Here `skill_id` is purely a routing label, not a credential — one skill cannot
    eavesdrop on another's traffic. Useful when you want OVOS skills without a full
    messagebus and with strict per-skill isolation.

    **multimind** ([`hivemind-multimind-agent-plugin`](https://github.com/JarbasHiveMind/hivemind-multimind-agent-plugin))
    — a per-access-key agent *multiplexer*. Every access key gets its own isolated
    agent instance (by default a bundled MiniCroft brain), and each key's
    `{module, config}` is stored in that client's DB metadata. This is the
    multi-tenant story: one server, many independent assistants, one per satellite key.

---

## Source

Validated against the HiveMind source:

- [`hivemind_ovos_agent_plugin/__init__.py`](https://github.com/JarbasHiveMind/hivemind-ovos-agent-plugin/blob/HEAD/hivemind_ovos_agent_plugin/__init__.py) — the `host` / `port` / `connection_timeout` agent keys and the `hivemind-ovos-agent-plugin` / `hivemind-ovos-agent-policy` entry points
- [`hivemind_websocket_protocol/__init__.py`](https://github.com/JarbasHiveMind/hivemind-websocket-protocol/blob/HEAD/hivemind_websocket_protocol/__init__.py) — the WebSocket plugin's default port `5678` and TLS keys
- [`hivemind_core/config.py`](https://github.com/JarbasHiveMind/HiveMind-core/blob/HEAD/hivemind_core/config.py) — the default `server.json` (encodings, ciphers, network/agent/database blocks)
