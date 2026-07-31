# Protocol

The HiveMind Protocol exchanges information and commands within a distributed network. It defines message types and how to handle them, acting as a transport protocol. The protocol works mainly with OpenVoiceOS (OVOS) messages, but it can carry other payloads too.

The protocol has two main roles: the Listener Protocol and the Client Protocol.

## Roles and message types

### Listener Protocol

- **Accepts**: `BUS`, `SHARED_BUS`, `PROPAGATE`, `ESCALATE`, `INTERCOM`
- **Emits**: `BUS`, `PROPAGATE`, `BROADCAST`, `INTERCOM`

### Client Protocol

- **Accepts**: `BUS`, `PROPAGATE`, `BROADCAST`, `INTERCOM`
- **Emits**: `BUS`, `SHARED_BUS`, `PROPAGATE`, `ESCALATE`, `INTERCOM`

### Permissions

Permissions combine:

- the access key.
- allowed message types.
- blacklisted intent types.
- blacklisted skill IDs.

> Use the [hivemind-core](https://github.com/JarbasHiveMind/HiveMind-core) package to authorize message types or blacklist intents and skills.

Example: allow the "speak" message type:

```bash
hivemind-core allow-msg "speak"
```

---

## Payload messages

Payload messages carry OpenVoiceOS `Message` objects. They are the cargo the HiveMind Protocol transports across the network.

Integrations with external AI backends need middleware to process OVOS messages. See [hivemind-persona](https://github.com/JarbasHiveMind/hivemind-persona) for an example implementation.

> Every HiveMind server must handle natural language queries. At minimum, it must support the `recognizer_loop:utterance` OVOS message.

> Use the [hivemind-websocket-client](https://github.com/JarbasHiveMind/hivemind_websocket_client) package to send a bus message from the command line.

### BUS message

- **Purpose**: single-hop communication between slaves and masters.
- **Behavior**:
    - A master receiving a `BUS` message checks global whitelists and blacklists, and the slave's permissions.
    - Authorized messages get injected into the master's OVOS-core bus.
    - Direct responses from the master's OVOS-core forward back to the originating slave.

**Command line**:

```bash
hivemind-client send-mycroft --help
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

> Find valid OVOS payloads [here](https://github.com/OpenVoiceOS/message_spec).

**Visualization**:

![BUS Message Flow](https://raw.githubusercontent.com/JarbasHiveMind/HiveMind-core/dev/resources/bus.gif)

### SHARED_BUS message

- **Purpose**: passive monitoring of a slave device's OVOS-core bus.
- **Direction**: slave to master.
- **Behavior**:
    - Requires explicit configuration on the slave device.
    - Works like `BUS`, but for observation, not processing.

> The [HiveMind Skill](https://github.com/JarbasHiveMind/ovos-skill-fallback-hivemind) typically enables this feature.

**Visualization**:

![Shared Bus Message Flow](https://raw.githubusercontent.com/JarbasHiveMind/HiveMind-core/dev/resources/shared_bus.gif)

---

### INTERCOM message

Messages can also be encrypted with a node's [public key](03_pairing.md#the-identity-file). This stops intermediate nodes from reading the message contents.

An encrypted message is a regular hive message, but has the type `"INTERCOM"` and payload `{"ciphertext": "XXXXXXX"}`.

Only the target node can decode `"ciphertext"`, not any intermediary.

These messages usually appear as the payload of transport messages such as `ESCALATE` or `PROPAGATE`.

> Intermediate nodes do not know the contents of the message, or who the recipient is.

To send a message securely, HiveMind encrypts it with the recipient node's public PGP key. Only the intended recipient, holding the matching private PGP key, can decrypt it.

After encryption, HiveMind signs the message with the sender's private PGP key. This authenticates the sender and confirms the message was not tampered with.

When a node receives an encrypted message, it tries to decrypt it with its private PGP key. If decryption succeeds, the node processes and emits the payload internally.

You must know the target node's public key beforehand to send it a secret message.

---

## Transport messages

Transport messages carry another `HiveMessage` object as their payload. These types matter most for [Nested Hives](15_nested.md).

### BROADCAST message

- **Purpose**: multi-hop communication from master to slaves.
- **Behavior**:
    - Sends messages to all connected slaves.
    - Supports `target_site_id` to direct messages to specific nodes.

**Example**: a master can make all slaves in `site_id: "kitchen"` speak a specific message.

> Skills running on a hivemind server typically send `BROADCAST` messages.

**Visualization**:

![Broadcast Message Flow](https://raw.githubusercontent.com/JarbasHiveMind/HiveMind-core/dev/resources/broadcast.gif)

### ESCALATE message

- **Purpose**: multi-hop communication from slave to master.
- **Behavior**:
    - Elevates messages up the authority chain for higher-level processing.

**Visualization**:

![Escalate Message Flow](https://raw.githubusercontent.com/JarbasHiveMind/HiveMind-core/dev/resources/escalate.gif)

**Command line**:

```bash
hivemind-client escalate --help
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

### PROPAGATE message

- **Purpose**: multi-hop communication in both directions, master to slaves and back.
- **Behavior**:
    - Delivers the message to all relevant nodes.

**Visualization**:

![Propagate Message Flow](https://raw.githubusercontent.com/JarbasHiveMind/HiveMind-core/dev/resources/propagate.gif)

**Command line**:

```bash
hivemind-client propagate --help
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

## Protocol features

| Feature              | Protocol v0 | Protocol v1 |
|----------------------|-------------|-------------|
| JSON serialization   | Yes         | Yes         |
| Binary serialization | No          | Yes         |
| Pre-shared AES key   | Yes         | Yes         |
| Password handshake   | No          | Yes         |
| PGP handshake        | No          | Yes         |
| Zlib compression     | No          | Yes         |

> Protocol v0 is deprecated. Some clients, such as HiveMind-Js, may not yet support Protocol Version 1.

---
[← Home Assistant](07_homeassistant.md) · [Home](index.md) · [Binarization →](18_binarization.md)
