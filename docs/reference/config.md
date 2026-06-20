# Configuration Reference

## server.json

`hivemind-core` reads `~/.config/hivemind-core/server.json` at startup. `hivemind-core listen` takes no command-line flags; edit this file to change any setting.

### Full default configuration

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

An MQTT transport (`hivemind-mqtt-protocol`) is planned/experimental and not confirmed published.

### policy.chain

List of policy plugin modules applied in order to each incoming message. The default chain contains a single entry: `hivemind-ovos-agent-policy`, which enforces per-client skill and intent blacklists from `Client.metadata`.

### database

| Key | Default | Description |
|---|---|---|
| `module` | `"hivemind-sqlite-db-plugin"` | Database backend plugin |

Per-plugin settings are nested under the plugin name key. See [Database Backends](../concepts/databases.md).

---

## Identity file

`~/.config/hivemind/_identity.json` — written by `hivemind-client set-identity`.

| Field | Description |
|---|---|
| `access_key` | Access credential assigned by the hub |
| `password` | Used for PBKDF2 key derivation |
| `default_master` | Hub host address (ws:// or wss://) |
| `default_port` | Hub port (default: 5678) |
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
