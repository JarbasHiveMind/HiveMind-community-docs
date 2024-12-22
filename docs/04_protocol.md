# Protocol

The HiveMind Protocol enables seamless exchange of information and commands within a distributed network. It defines
message types and their handling methods, serving as a *transport* protocol. While the protocol primarily operates with
OpenVoiceOS (OVOS) messages, it is versatile enough to support other payloads.

The protocol is categorized into two main roles: **Listener Protocol** and **Client Protocol**.

## Roles and Message Types

### Listener Protocol

- **Accepts**: `BUS`, `SHARED_BUS`, `PROPAGATE`, `ESCALATE`
- **Emits**: `BUS`, `PROPAGATE`, `BROADCAST`

### Client Protocol

- **Accepts**: `BUS`, `PROPAGATE`, `BROADCAST`
- **Emits**: `BUS`, `SHARED_BUS`, `PROPAGATE`, `ESCALATE`

---

## Payload Messages

Payload messages encapsulate OpenVoiceOS `Message` objects, acting as carriers for information or commands. These are
the "cargo" the HiveMind Protocol transports across the network.

Integrations with external AI backends require middleware to process OVOS messages.
See [hivemind-persona](https://github.com/JarbasHiveMind/hivemind-persona) for an example implementation.

> ⚠️ All HiveMind servers are expected to handle natural language queries. At a minimum,
> the `recognizer_loop:utterance` OVOS message must be supported.

### BUS Message

- **Purpose**: Single-hop communication between slaves and masters.
- **Behavior**:
    - A master receiving a `BUS` message checks global whitelists/blacklists and slave permissions.
    - Authorized messages are injected into the master's OVOS-core bus.
    - Direct responses from the master's OVOS-core are forwarded back to the originating slave.

> 💡 Use the [hivemind-websocket-client](https://github.com/JarbasHiveMind/hivemind_websocket_client) package to send a bus message from the command line

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

#### Permissions

Permissions can be based on:

- Message type
- Intent type
- Skill ID
- Access key
- IP address rules

> 💡 Use the [hivemind-core](https://github.com/JarbasHiveMind/HiveMind-core) package to authorize message types or blacklist intents/skills.

**Example**: Allow the "speak" message type:

```bash
$ hivemind-core allow-msg "speak"
```

**Visualization**:

![BUS Message Flow](https://raw.githubusercontent.com/JarbasHiveMind/HiveMind-core/dev/resources/bus.gif)

**Use Case**: Terminal applications (e.g., voice-sat) inject natural language queries. Other integrations (e.g., Home
Assistant) may inject specific messages based on their configuration.

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


**Example**: Escalate a OVOS message:

```bash
$ hivemind-client escalate --msg "intent_failure" --payload "{}"
```

**Visualization**:

![Escalate Message Flow](https://raw.githubusercontent.com/JarbasHiveMind/HiveMind-core/dev/resources/escalate.gif)

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

TODO - explain each of the features above in a subsection below