# Protocol

The HiveMind Protocol enables seamless exchange of information and commands within a distributed network. It defines
message types and their handling methods, serving as a *transport* protocol. While the protocol primarily operates with
OpenVoiceOS (OVOS) messages, it is versatile enough to support other payloads.

The protocol is categorized into two main roles: **Listener Protocol** and **Client Protocol**.

## Roles and Message Types

### Listener Protocol

- **Accepts**: `BUS`, `SHARED_BUS`, `PROPAGATE`, `ESCALATE`, `INTERCOM`
- **Emits**: `BUS`, `PROPAGATE`, `BROADCAST`, `INTERCOM`

### Client Protocol

- **Accepts**: `BUS`, `PROPAGATE`, `BROADCAST`, `INTERCOM`
- **Emits**: `BUS`, `SHARED_BUS`, `PROPAGATE`, `ESCALATE`, `INTERCOM`

## Message Types Reference

Each `HiveMessage` carries a `msg_type` from the `HiveMessageType` enum (defined in `hivemind_bus_client.message.HiveMessageType`) that dictates how the hub routes and processes the message.

| Type | Purpose | Direction | Transport |
|---|---|---|---|
| **`BUS`** | Standard message for the AI | Bidirectional | Injected to OVOS bus |
| **`SHARED_BUS`** | Passive monitoring of a slave's local bus | Slave → Master | Observed, not processed |
| **`ESCALATE`** | Request from satellite to master for help | Uplink | Hop-limited escalation |
| **`BROADCAST`** | Admin command to all satellites (master only) | Master → All | Flooding |
| **`PROPAGATE`** | Bidirectional flood to all peers | Bidirectional | Full mesh propagation |
| **`INTERCOM`** | End-to-end encrypted peer-to-peer | Point-to-point | Encrypted tunnel |
| **`PING`** | Network discovery probe; always inside PROPAGATE | Bidirectional | Topology mapping |
| **`PONG`** | Discovery reply with hive path metadata | Uplink | Route information |
| **`HELLO`** | Node announcement at connection time | Downlink | Session sync |
| **`HANDSHAKE`** | Cryptographic key exchange | Bidirectional | Encrypted handshake |
| **`THIRDPRTY`** | User-defined payload type | Application-dependent | Relayed opaquely |
| **`BIN`** | Raw binary data (audio, file transfers) | Bidirectional | Binary encoding |

**Key notes:**
- **PING/PONG**: Always wrapped inside a PROPAGATE message; never sent bare. Used for topology discovery via `HiveMapper` in `hivemind_core.hive_map`.
- **HELLO/HANDSHAKE**: Automatically managed by `HiveMessageBusClient` during connection setup (via `poorman_handshake.PasswordHandShake`).
- **Context metadata**: All messages carry a `context` dict with routing keys (`peer`, `source`, `destination`, `session`, etc.) — see [Session Management & Context Keys](#session-management--context-keys) below.

### Permissions

Permissions are based on a combination of:

- Access key
- Allowed Message types
- Blacklisted Intent types
- Blacklisted Skill IDs

> 💡 Use the [hivemind-core](https://github.com/JarbasHiveMind/HiveMind-core) package to authorize message types or blacklist intents/skills.

**Example**: Allow the "speak" message type:

```bash
$ hivemind-core allow-msg "speak"
```


---

## Payload Messages

Payload messages encapsulate OpenVoiceOS `Message` objects, acting as carriers for information or commands. These are
the "cargo" the HiveMind Protocol transports across the network.

Integrations with external AI backends require middleware to process OVOS messages.
See [hivemind-persona](https://github.com/JarbasHiveMind/hivemind-persona) for an example implementation.

> ⚠️ All HiveMind servers are expected to handle natural language queries. At a minimum,
> the `recognizer_loop:utterance` OVOS message must be supported.

> 💡 Use the [hivemind-websocket-client](https://github.com/JarbasHiveMind/hivemind_websocket_client) package to send a bus message from the command line

### BUS Message

- **Purpose**: Single-hop communication between slaves and masters.
- **Behavior**:
    - A master receiving a `BUS` message checks global whitelists/blacklists and slave permissions.
    - Authorized messages are injected into the master's OVOS-core bus.
    - Direct responses from the master's OVOS-core are forwarded back to the originating slave.

**Command Line**:

```bash
$ hivemind-client send-mycroft --help
Usage: hivemind-client send-mycroft [OPTIONS]

  send a single mycroft message

Options:
  --key TEXT       HiveMind access key (default read from identity file)
  --password TEXT  HiveMind password (default read from identity file)
  --host TEXT      HiveMind host (default read from identity file)
  --port INTEGER   HiveMind port number (default: 5678)
  --siteid TEXT    location identifier for message.context  (default read from
                   identity file)
  --msg TEXT       ovos message type to inject
  --payload TEXT   ovos message.data json
  --help           Show this message and exit.
```

> 💡 Valid payloads for OVOS can be found [here](https://github.com/OpenVoiceOS/message_spec)

**Visualization**:

![BUS Message Flow](https://raw.githubusercontent.com/JarbasHiveMind/HiveMind-core/dev/resources/bus.gif)


### SHARED_BUS Message

- **Purpose**: Passive monitoring of a slave device's OVOS-core bus.
- **Direction**: Slave → Master.
- **Behavior**:
    - Requires explicit configuration on the slave device.
    - Similar to `BUS`, but for observation, not processing.

> 💡 This feature is typically enabled through the [HiveMind Skill](https://github.com/JarbasHiveMind/ovos-skill-fallback-hivemind).

**Visualization**:

![Shared Bus Message Flow](https://raw.githubusercontent.com/JarbasHiveMind/HiveMind-core/dev/resources/shared_bus.gif)

---


### INTERCOM Message

messages may also be encrypted with a node [public_key](https://jarbashivemind.github.io/HiveMind-community-docs/03_pairing/#the-identity-file), this ensures intermediate nodes are unable to read the message contents

A encrypted message is a regular hive message, but has the type `"INTERCOM"` and payload `{"ciphertext": "XXXXXXX"}`

Where `"ciphertext"` can only be decoded by the target Node, not by any intermediary

these messages are usually the payload of transport messages such as `ESCALATE` or `PROPAGATE` payloads. 

> 💡 Intermediate nodes do not know **the contents** of the message, nor **who the recipient is**

When a message needs to be sent securely, it is encrypted using the recipient node's public PGP key. This ensures that only the intended recipient, who possesses the corresponding private PGP key, can decrypt the message.

After encryption, the message is signed with the sender's private PGP key. This provides authentication and integrity, ensuring that the message has not been tampered with and confirming the sender's identity.

Upon receiving an encrypted message, the recipient node attempts to decrypt it using its private PGP key. If successful, the message payload is then processed and emitted internally.

the target node public key needs to be known beforehand if you want to send secret messages

---

## Transport Messages

Transport messages encapsulate another `HiveMessage` object as their payload. These types are particularly relevant
for [Nested Hives](https://jarbashivemind.github.io/HiveMind-community-docs/15_nested/).

### BROADCAST Message

- **Purpose**: Multi-hop communication from master → slaves.
- **Behavior**:
    - Disseminates messages to all connected slaves.
    - Supports `target_site_id` for directing messages to specific nodes.

**Example**: A master can make all slaves in `site_id: "kitchen"` speak a specific message.

> 💡 `BROADCAST` messages are typically sent by skills running in a hivemind server

**Visualization**:

![Broadcast Message Flow](https://raw.githubusercontent.com/JarbasHiveMind/HiveMind-core/dev/resources/broadcast.gif)

### ESCALATE Message

- **Purpose**: Multi-hop communication from slave → master.
- **Behavior**:
    - Elevates messages up the authority chain for higher-level processing.

**Visualization**:

![Escalate Message Flow](https://raw.githubusercontent.com/JarbasHiveMind/HiveMind-core/dev/resources/escalate.gif)

**Command Line**:

```bash
$ hivemind-client escalate --help
Usage: hivemind-client escalate [OPTIONS]

  escalate a single mycroft message

Options:
  --key TEXT       HiveMind access key (default read from identity file)
  --password TEXT  HiveMind password (default read from identity file)
  --host TEXT      HiveMind host (default read from identity file)
  --port INTEGER   HiveMind port number (default: 5678)
  --siteid TEXT    location identifier for message.context  (default read from
                   identity file)
  --msg TEXT       ovos message type to inject
  --payload TEXT   ovos message.data json
  --help           Show this message and exit.

```

### PROPAGATE Message

- **Purpose**: Multi-hop communication in both directions (master ↔ slaves).
- **Behavior**:
    - Ensures the message is delivered to all relevant nodes.

**Visualization**:

![Propagate Message Flow](https://raw.githubusercontent.com/JarbasHiveMind/HiveMind-core/dev/resources/propagate.gif)

**Command Line**:

```bash
$ hivemind-client propagate --help
Usage: hivemind-client propagate [OPTIONS]

  propagate a single mycroft message

Options:
  --key TEXT       HiveMind access key (default read from identity file)
  --password TEXT  HiveMind password (default read from identity file)
  --host TEXT      HiveMind host (default read from identity file)
  --port INTEGER   HiveMind port number (default: 5678)
  --siteid TEXT    location identifier for message.context  (default read from
                   identity file)
  --msg TEXT       ovos message type to inject
  --payload TEXT   ovos message.data json
  --help           Show this message and exit.
```

---

## Session Management & Context Keys

HiveMind uses OVOS `Message.context` keys to manage routing, session state, and permissions. When a satellite sends an utterance, the hub injects routing metadata before passing it to OVOS, then uses that same metadata to route the response back.

### Context Keys — Full Reference

| Key | Set by | Source | Value | Role |
|-----|--------|--------|-------|------|
| `context["peer"]` | `handle_inject_agent_msg` | `hivemind_core/protocol.py:949` | Satellite peer ID (e.g. `"HiveMindV0.0@127.0.0.1:8222/0"`) | Identifies which satellite sent this message; used in test assertions |
| `context["source"]` (inbound) | `handle_inject_agent_msg` | `protocol.py:949` | Same as `peer` | OVOS origin identifier; becomes `destination` after `Message.reply()` swap |
| `context["destination"]` (inbound) | `handle_inject_agent_msg` | `protocol.py:942-945` | `"skills"` (default) or `["audio"]` | Prevents message being treated as broadcast; routes responses back to originator |
| `context["session"]` | `_update_blacklist` | `protocol.py:903` | Full serialized `Session` dict | Per-satellite session state; includes `session_id`, `blacklisted_skills`, `blacklisted_intents` |
| `context["session"]["session_id"]` | `Session.__init__` | `ovos-bus-client/session.py:311` | UUID or explicit string (e.g., `"test-session"`) | Stable identifier persisting across utterances; enables conversational state |
| `context["session"]["blacklisted_skills"]` | `_update_blacklist` | `protocol.py:918` | List of skill IDs | ACL enforcement: `IntentService` skips these skills |
| `context["session"]["blacklisted_intents"]` | `_update_blacklist` | `protocol.py:921` | List of intent names | ACL enforcement: `IntentService` skips these intents |
| `context["destination"]` (outbound) | `Message.reply()` swap | `ovos-bus-client/message.py:195-198` | Peer ID (swapped from `source`) | What `HiveMindListenerProtocol.handle_internal_mycroft()` reads to route response back |
| `context["source"]` (outbound) | `handle_internal_mycroft` | `protocol.py:123` | `"hive"` | Marks message as coming from HiveMind hub |

### Session ID Lifecycle

`session_id` is the stable identifier that lets OVOS maintain conversational state (active skill, intent history, etc.) across multiple utterances from the same satellite.

1. **Created on satellite** — Explicit ID or auto-generated UUID:
   ```python
   sess = Session("test-session")  # or Session() for UUID
   message = Message(
       "recognizer_loop:utterance",
       {"utterances": ["hello"]},
       {"session": sess.serialize(), "source": "sat", "destination": "master"}
   )
   ```

2. **Extracted at hub** (`protocol.py:903`) — `_update_blacklist()` replaces `context["session"]` with the hub's per-connection session tracking (`client.sess`), ensuring DB-enforced blacklists are always applied while honouring the satellite's `session_id`.

3. **Read by OVOS IntentService** — `_validate_session()` deserializes the session from the message and maintains conversational state across skill invocations.

4. **Updated after skill match** — `IntentService` updates `context["session"]` with new `active_skills` and returns it in all replies.

5. **Returned to satellite** — All replies (`speak`, `mycroft.skill.handler.complete`, etc.) carry the updated `context["session"]`, allowing satellites to inspect `msg.context["session"]["active_skills"]` to see the current state.

### Message Reply Routing (The Core Mechanism)

Reverse routing relies on the `Message.reply()` method from `ovos-bus-client` (message.py:195-198), which swaps `source` ↔ `destination` in the context. When HiveMind injects an utterance it sets:
```python
context["source"] = client.peer          # e.g. "HiveMindV0.0@127.0.0.1:8222/0"
context["destination"] = "skills"        # default, or [peer] if already set
```

After `Message.reply()` is called by OVOS:
- `destination` becomes the peer ID (the originating satellite)
- `source` becomes `"skills"`

Every subsequent skill message (`speak`, `intent.handled`, etc.) is also a `.reply()`, so the destination keeps pointing at the originator. `HiveMindListenerProtocol.handle_internal_mycroft()` reads `context["destination"]` and routes the message to the correct peer connection.

---

## Protocol Features


| Feature              | Protocol v0 | Protocol v1 |
|----------------------|-------------|-------------|
| JSON serialization   | ✅           | ✅           |
| Binary serialization | ❌           | ✅           |
| Pre-shared AES key   | ✅           | ✅           |
| Password handshake   | ❌           | ✅           |
| PGP handshake        | ❌           | ✅           |
| Zlib compression     | ❌           | ✅           |

> ⚠️ Protocol v0 is **deprecated**! However some clients (e.g., HiveMind-Js) may not yet support Protocol Version 1.

