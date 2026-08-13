# Protocol

The HiveMind Protocol exchanges information and commands within a distributed network. It defines message types and how to handle them, acting as a transport protocol. The protocol works mainly with OpenVoiceOS (OVOS) messages, but it can carry other payloads too.

The protocol has two main roles: the Listener Protocol and the Client Protocol.

## Roles and message types

### Listener Protocol

- **Accepts**: `BUS`, `SHARED_BUS`, `PROPAGATE`, `ESCALATE`, `INTERCOM`, `QUERY`, `CASCADE`, `RENDEZVOUS`
- **Emits**: `BUS`, `PROPAGATE`, `BROADCAST`, `INTERCOM`, `QUERY`, `CASCADE`, `RENDEZVOUS`

### Client Protocol

- **Accepts**: `BUS`, `PROPAGATE`, `BROADCAST`, `INTERCOM`, `QUERY`, `CASCADE`, `RENDEZVOUS`
- **Emits**: `BUS`, `SHARED_BUS`, `PROPAGATE`, `ESCALATE`, `INTERCOM`, `QUERY`, `CASCADE`, `RENDEZVOUS`

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

INTERCOM is the end-to-end encrypted peer-to-peer channel. Messages are encrypted with a node's [public key](03_pairing.md#the-identity-file), so intermediate nodes cannot read the contents, and only the intended recipient can decrypt them.

An encrypted message is a regular hive message, but has the type `"INTERCOM"` and payload `{"ciphertext": "XXXXXXX"}`.

Only the target node can decode `"ciphertext"`, not any intermediary.

These messages usually appear as the payload of transport messages such as `ESCALATE` or `PROPAGATE`.

> Intermediate nodes do not know the contents of the message, or who the recipient is.

To send a message securely, HiveMind encrypts it with the recipient node's public PGP key. Only the intended recipient, holding the matching private PGP key, can decrypt it.

After encryption, HiveMind signs the message with the sender's private PGP key.

#### Origin authentication

A receiving node verifies that signature against a key in its own [trusted keys](03_pairing.md#contents-of-the-identity-file), with its master's public key acting as a trust anchor. A frame with no pinned or trusted key for the sender, no signature, or a signature that fails verification is dropped: it is never decrypted, dispatched, relayed to peers, or escalated upstream.

This stops an outsider from injecting a message onto a node's bus by knowing its public key — the key is public by design, published in every `PING` answer, so knowing it was never meant to be enough on its own. The guarantee has a limit worth stating plainly: it stops an outsider, but it does not stop one trusted peer claiming to be another trusted peer.

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

### QUERY message

- **Purpose**: ask a specific peer a question and get a response routed back, without flooding the whole mesh.
- **Permission**: sending a `QUERY` requires `can_escalate`.
- **Behavior**:
    - The response is a frame carrying `{"is_response": true, "originator_peer": <peer>}` in its metadata, so a relaying node can route it straight back instead of flooding it.
    - The permission check runs before a frame claiming to be a response is routed, so a client without `can_escalate` cannot forge `is_response`/`originator_peer` to deliver arbitrary content to another peer.
    - Routing trusts the sender's own `originator_peer`. A sender that does have `can_escalate` can still address a response to a peer for a query it never took part in — the check stops an unprivileged client from forging a response, it does not verify the response actually answers the query it claims to.

### CASCADE message

- **Purpose**: like `QUERY`, but collects responses from multiple peers down a fan-out instead of a single target.
- **Permission**: sending a `CASCADE` requires `can_propagate`.
- **Behavior**: same routing and permission-ordering guarantees, and the same limit, as `QUERY` above.

### RENDEZVOUS message

- **Purpose**: store-and-forward mailbox for peers that are not connected to the same node at the same time.
- **Behavior**:
    - The node serving the mailbox answers with `mailbox_node`, naming itself, on every path — including refusals.
    - Mail is held per node: two peers attached to different nodes still exchange nothing through `RENDEZVOUS`. With `mailbox_node` in the answer, a client can at least tell that is what happened, instead of a silent empty mailbox.

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
