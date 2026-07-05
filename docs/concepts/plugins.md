# Plugin Architecture

hivemind-core is less a monolith than a socket board. Almost nothing about it is
welded in place: the way it talks to devices, which AI actually answers a question, how
it deals with raw audio, where it files the guest list, and how it decides what each
device may do — every one of those is a part you can pull out and replace. Want the LLM
brain instead of the skills brain? Swap one part; the satellites never notice. Want to
store credentials in Redis instead of a file? Swap another. Five families of parts,
one board they all plug into, and a manager that snaps the right ones in at startup.

!!! abstract "In a nutshell"
    - Five families of swappable parts: **network protocol** (how bytes arrive), **agent protocol** (who answers), **binary data handler** (raw audio), **database** (the guest list), and **policy** (who may do what).
    - Out of the box you get the WebSocket + HTTP transports, the OVOS brain, a SQLite guest list, and the OVOS permission policy. No audio handler loads until you ask for one.
    - You wire the parts together in `~/.config/hivemind-core/server.json`. Heads-up: the name you `pip install` isn't always the name you write in config — the package and the plugin can differ.

![HPM diagram](../hpm.png)

## Plugin categories

Five families, five jobs. The tables below list what's available in each — but as you read
them, the useful question isn't "what are all these?" so much as "which one would I swap to
change *this*?" Each family answers exactly one such question.

### Network protocol plugins

**The question they answer: how do bytes get in and out?** These control how HiveMind
listens for and connects to satellites.

| Plugin | Transport | Default port | Status |
|---|---|---|---|
| `hivemind-websocket-plugin` | WebSocket (ws:// / wss://) | 5678 | stable, default |
| `hivemind-http-plugin` | HTTP (polling) | 5679 | stable, default |
| `hivemind-mqtt-plugin` | MQTT (broker) | 1883 | alpha (PyPI `hivemind-mqtt-protocol`) |
| `hivemind-usenet-wormhole` | Usenet store-and-forward | — | experimental, unpublished |

WebSocket and HTTP are enabled by default in `server.json`. The MQTT plugin
(package `hivemind-mqtt-protocol`) is published as an alpha and ships the server-side
listener; a satellite-side MQTT client is planned. The Usenet wormhole
(package `hivemind-usenet`) is an experimental covert/fallback transport — not a
real-time channel, not on PyPI (it pulls git-only carrier libraries).

### Agent protocol plugins

**The question they answer: who actually thinks?** This is the one swap that changes what
your assistant *is* — skills, an LLM, or someone else's agent — while the satellites never
notice.

| Plugin | Back-end |
|---|---|
| `hivemind-ovos-agent-plugin` | OpenVoiceOS (default) |
| `hivemind-persona-agent-plugin` | ovos-persona / LLM solvers |
| `hivemind-a2a-agent-plugin` | Google A2A agents (bridges the hive to A2A) |

### Binary data handler plugins

**The question they answer: what happens to raw audio?** This is the ear on the server —
the piece a thin satellite leans on to have its speech transcribed.

| Plugin | Function |
|---|---|
| `hivemind-audio-binary-protocol-plugin` | Server-side wakeword, STT, VAD, TTS for mic-satellite and voice-relay |

No binary plugin is loaded by default. Install and configure one when you need server-side audio processing.

### Database plugins

**The question they answer: where does the guest list live?** A file for one server, Redis
when several share it.

| Plugin | Backend |
|---|---|
| `hivemind-sqlite-db-plugin` | SQLite (default for new installs) |
| `hivemind-json-db-plugin` | JSON file |
| `hivemind-redis-db-plugin` | Redis (from the `hivemind-redis-database` package) |

### Policy plugins

**The question they answer: who's allowed to do what?** Admission control — the gate every
message passes before it reaches the agent. Policy plugins are loaded as an ordered **chain** (group `hivemind.policy`) and are consulted for every inbound message; they configure only via the `policy.chain` JSON block (there is no separate `module` selector like the other families).

| Plugin | Function |
|---|---|
| `hivemind-ovos-agent-policy` | Per-client skill/intent blacklists for the OVOS agent (default) |

The built-in `allowed_types` ACL (`MessageTypeACLPolicy`) is always force-prepended to the chain and cannot be removed. See [Security — How the policy chain works](security.md#how-the-policy-chain-works) and [Writing Plugins — Policy plugins](../developers/writing-plugins.md#5-policy).

---

## Configuration

All five families meet in one place: `server.json`. Reading the default file is the
fastest way to see how the parts name each other — every `module` key is one part snapping
into its socket:

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

---

## Installing plugins

Plugins are installed as Python packages via pip:

```bash
pip install hivemind-audio-binary-protocol
pip install hivemind-redis-database
```

After installation, update `server.json` to reference the new plugin (entry-point) name — note that the installed package name may differ from the plugin name used in config (e.g. the `hivemind-redis-database` package provides the `hivemind-redis-db-plugin` plugin, and `hivemind-audio-binary-protocol` provides `hivemind-audio-binary-protocol-plugin`).

---

## Developing plugins

Use the `hivemind-plugin-manager` package to discover and instantiate plugins programmatically:

```python
from hivemind_plugin_manager import find_plugins, HiveMindPluginTypes

# Discover all installed database plugins
print(find_plugins(HiveMindPluginTypes.DATABASE))

# Discover all installed agent protocol plugins
print(find_plugins(HiveMindPluginTypes.AGENT_PROTOCOL))
```

Factory classes handle instantiation:

```python
from hivemind_plugin_manager import DatabaseFactory, AgentProtocolFactory

db = DatabaseFactory.create("hivemind-redis-db-plugin",
                            password="secret", host="localhost", port=6379)

agent = AgentProtocolFactory.create("hivemind-ovos-agent-plugin")
```

To author your own plugin, see [Writing Plugins](../developers/writing-plugins.md).

---

## Source

Validated against the HiveMind source:

- [`hivemind_plugin_manager/__init__.py`](https://github.com/JarbasHiveMind/hivemind-plugin-manager/blob/HEAD/hivemind_plugin_manager/__init__.py) — `HiveMindPluginTypes` (the five families: database, network, agent, binary, policy)
- [`hivemind_core/config.py`](https://github.com/JarbasHiveMind/HiveMind-core/blob/HEAD/hivemind_core/config.py) — default `server.json` wiring, including the `policy.chain` block
