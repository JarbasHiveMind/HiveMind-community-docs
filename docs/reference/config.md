# Configuration Reference

One file decides how your server behaves: `~/.config/hivemind-core/server.json`. There
are no start-up flags to hunt for — every port, every cipher, every plugin choice lives
here. This page lays out that file in full, block by block, with every default spelled
out, plus the satellite identity file and the ports everything listens on.

!!! abstract "In a nutshell"
    - `hivemind-core` reads `~/.config/hivemind-core/server.json` at startup; `hivemind-core listen` takes no flags, so all settings live in this file.
    - Configurable blocks: `binarize`, `allowed_encodings`/`allowed_ciphers`, `agent_protocol`, `binary_protocol`, `network_protocol`, `policy.chain`, and `database`.
    - The database backend and TLS certificates are auto-selected on first run.
    - Satellites store credentials in `~/.config/hivemind/_identity.json`, written by `hivemind-client set-identity`.

---

## server.json

`hivemind-core` reads `~/.config/hivemind-core/server.json` at startup. `hivemind-core listen` takes no command-line flags; edit this file to change any setting.

### Full default configuration

```json
{
  "binarize": false,
  "allowed_encodings": [
    "JSON-B64", "JSON-URLSAFE-B64",
    "JSON-B91",
    "JSON-Z85B", "JSON-Z85P",
    "JSON-B32", "JSON-HEX"
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

The `database` block above is selected automatically on first run: a fresh install defaults to `hivemind-sqlite-db-plugin`, while an existing JSON deployment keeps `hivemind-json-db-plugin` (see [database](#database) below). `cert_dir` defaults to `<xdg_data_home>/hivemind` (e.g. `~/.local/share/hivemind`).

### binarize

| Key | Default | Description |
|---|---|---|
| `binarize` | `false` | Enables the HiveMind binary framing protocol once it has been negotiated with the client. Defaults to `false` to stay compatible with older `hivemind-bus-client` versions that mishandle binary frames. |

### allowed_encodings / allowed_ciphers

| Key | Default | Description |
|---|---|---|
| `allowed_encodings` | the 7 values below | Permitted message encodings, ordered by preference. The first encoding both peers support is used. |
| `allowed_ciphers` | `["CHACHA20-POLY1305", "AES-GCM"]` | Permitted symmetric ciphers for payload encryption, ordered by preference. |

The supported encodings are: `JSON-B64`, `JSON-URLSAFE-B64`, `JSON-B91`, `JSON-Z85B`, `JSON-Z85P`, `JSON-B32`, `JSON-HEX`. Trim either list to restrict what the server will negotiate.

### agent_protocol

| Key | Default | Description |
|---|---|---|
| `module` | `"hivemind-ovos-agent-plugin"` | Agent backend plugin |
| `hivemind-ovos-agent-plugin.host` | `"127.0.0.1"` | OVOS messagebus host |
| `hivemind-ovos-agent-plugin.port` | `8181` | OVOS messagebus port |

Set `module` to `"hivemind-persona-agent-plugin"` and configure accordingly for LLM/persona mode.

### binary_protocol

| Key | Default | Description |
|---|---|---|
| `module` | `null` | Binary data handler plugin. Set to `"hivemind-audio-binary-protocol-plugin"` to enable server-side audio |

When set to the audio binary protocol plugin, configure STT, TTS, VAD, and wakeword sub-keys. See [Audio Binary Protocol](../server/audio-binary-protocol.md).

### network_protocol

Multiple network protocol plugins can be active simultaneously. Each key is the plugin name; its value is the plugin-specific config.

| Plugin | Default port | Protocol |
|---|---|---|
| `hivemind-websocket-plugin` | 5678 | WebSocket |
| `hivemind-http-plugin` | 5679 | HTTP polling |

An MQTT transport is available as an alpha: package `hivemind-mqtt-protocol`,
plugin name `hivemind-mqtt-plugin`, broker port 1883 (config keys `broker_host`,
`broker_port`, `broker_username`, `broker_password`, `tls`, `topic_prefix`, `qos`,
`idle_timeout`). It provides the server-side listener only. An experimental
Usenet wormhole transport (`hivemind-usenet`, plugin `hivemind-usenet-wormhole`)
also exists but is unpublished.

Each network protocol plugin accepts the following TLS keys for serving over `wss://` / `https://`:

| Key | Default | Description |
|---|---|---|
| `ssl` | `false` | Serve over TLS. When `true`, a certificate is loaded from `cert_dir`/`cert_name` (generated if absent). |
| `cert_dir` | `<xdg_data_home>/hivemind` | Directory holding the TLS certificate and key. |
| `cert_name` | `"hivemind"` | Base filename for the `.crt`/`.key` pair in `cert_dir`. |

See [Security & Permissions](../concepts/security.md) for the full TLS setup.

### policy.chain

List of policy plugin modules applied in order to each incoming message. The default chain contains a single entry: `hivemind-ovos-agent-policy`, which enforces per-client skill and intent blacklists from `Client.metadata`.

!!! note "Removed / ignored key: `policy.fail_open`"
    The policy chain is **unconditionally fail-closed** — any error in a policy denies the message, and there is no knob to change this. If an old `server.json` still carries a `policy.fail_open` key, `hivemind-core` **strips it on load and logs a warning**; the key has no effect. Operators upgrading a server should delete it.

### database

| Key | Default | Description |
|---|---|---|
| `module` | `"hivemind-sqlite-db-plugin"` | Database backend plugin |

Per-plugin settings are nested under the plugin name key. See [Database Backends](../concepts/databases.md).

**Auto-selection:** when `server.json` does not already define a `database` block, the backend is chosen automatically on first run. If a legacy `clients.json` exists (in `<xdg_data_home>/hivemind-core/`) and no SQLite `clients.db` is present, the JSON backend (`hivemind-json-db-plugin`) is kept so an upgrade never strands an existing credential store. Otherwise SQLite (`hivemind-sqlite-db-plugin`) is the default. Move an existing JSON store to SQLite with `hivemind-core migrate-db --to sqlite`.

---

## Identity file

`~/.config/hivemind/_identity.json` — written by `hivemind-client set-identity`.

| Field | Description |
|---|---|
| `access_key` | Access credential assigned by hivemind-core |
| `password` | Used for PBKDF2 key derivation |
| `default_master` | hivemind-core host address (ws:// or wss://) |
| `default_port` | hivemind-core port (default: 5678) |
| `site_id` | Location identifier injected into OVOS context |
| `public_key` | RSA public key string |
| `secret_key` | Path to the RSA private key (PEM) file |

---

## Default ports

| Service | Port | Protocol |
|---|---|---|
| HiveMind WebSocket | 5678 | WebSocket (ws:// / wss://) |
| HiveMind HTTP | 5679 | HTTP |
| OVOS messagebus | 8181 | WebSocket (internal) |

---

**Next:** [CLI Reference](cli.md) · [Security & Permissions](../concepts/security.md)

---

## Source

Validated against the HiveMind source:

- [`hivemind_core/config.py`](https://github.com/JarbasHiveMind/HiveMind-core/blob/HEAD/hivemind_core/config.py) — the `_DEFAULT` config, database auto-selection, and the `policy.fail_open` strip-and-warn
- [`hivemind_core/policy.py`](https://github.com/JarbasHiveMind/HiveMind-core/blob/HEAD/hivemind_core/policy.py) — the fail-closed policy chain and `MessageTypeACLPolicy`
