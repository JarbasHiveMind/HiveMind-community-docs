# Protocol

A kitchen Pi, a server in the closet, a phone on the couch, maybe a second server
upstairs — for any of them to talk, they have to agree on two things: what a message
*looks like*, and where it goes *next*. That agreement is the HiveMind protocol. Every
message that crosses the wire is a single, uniform object — a `HiveMessage` — a JSON
envelope (or its compact binary twin) stamped with a `msg_type` that says what the
message is for and which way it should travel. The carrier underneath can be anything;
the envelope is always the same shape, so a satellite and hivemind-core never have to
guess.

!!! abstract "In a nutshell"
    - Every message is a `HiveMessage` tagged with a `msg_type`. There are only three flavours: **payloads** that carry content (`BUS`, `SHARED_BUS`, `THIRDPRTY`, `BINARY`), **routing verbs** that steer a message up, down, or across the mesh (`ESCALATE`, `BROADCAST`, `PROPAGATE`, `QUERY`, `CASCADE`, `INTERCOM`, `PING`), and **connection** frames for setup (`HELLO`, `HANDSHAKE`).
    - `BUS` is the one you'll meet most: a single hop from a satellite to the brain and back.
    - hivemind-core tags each message with who asked and which session it belongs to, then uses `Message.reply()` to make sure the answer finds its way home to the right satellite.

## HiveMessage types

Each message carries a `msg_type` from the `HiveMessageType` enum, defined in `hivemind_bus_client.message`.

### Payload messages

These carry OVOS `Message` objects (or other application payloads) as their content.

| Type | Purpose | Direction |
|---|---|---|
| **`BUS`** | Single-hop message to/from the AI back-end | Bidirectional |
| **`SHARED_BUS`** | Passive monitoring of a satellite's local OVOS bus | Satellite → hivemind-core |
| **`THIRDPRTY`** | User-defined payload, relayed opaquely | Application-defined |
| **`BINARY`** | Raw binary data (audio, images, files) | Bidirectional |

### Transport / routing messages

These wrap another `HiveMessage` as their payload and control how it is routed through the mesh.

| Type | Purpose | Direction |
|---|---|---|
| **`ESCALATE`** | Multi-hop upward — satellite asks parent for help | Satellite → hivemind-core (up the chain) |
| **`BROADCAST`** | Multi-hop downward — hivemind-core commands all satellites | hivemind-core → All satellites (down the chain) |
| **`PROPAGATE`** | Flood in all directions — reaches all reachable nodes | Bidirectional |
| **`QUERY`** | Routed request with a single expected response | Bidirectional |
| **`CASCADE`** | Scatter/gather — every node may answer; originator picks best | Bidirectional |
| **`INTERCOM`** | End-to-end encrypted point-to-point tunnel | Between any two nodes |
| **`PING`** | Topology probe — always wrapped inside PROPAGATE | Bidirectional (via PROPAGATE) |
| **`RENDEZVOUS`** | Reserved for rendezvous-nodes | Bidirectional |

### Connection management messages

These are handled automatically by `HiveMessageBusClient` and `hivemind-core`. You do not emit them manually.

| Type | Purpose |
|---|---|
| **`HELLO`** | Node announcement on connection (carries node ID and public key) |
| **`HANDSHAKE`** | Cryptographic key exchange. Protocol **v3** runs a **Noise** handshake (always-encrypted session); the legacy **v1/v2** path uses a password (salted-hash + PBKDF2) or RSA envelope. See [Security](security.md#handshake-and-encryption). |

---

## Roles

### hivemind-core (listener protocol)

- **Accepts**: `BUS`, `SHARED_BUS`, `PROPAGATE`, `ESCALATE`, `QUERY`, `CASCADE`, `INTERCOM`
- **Emits**: `BUS`, `PROPAGATE`, `BROADCAST`, `QUERY`, `CASCADE`, `INTERCOM`

### Satellite (client protocol)

- **Accepts**: `BUS`, `PROPAGATE`, `BROADCAST`, `QUERY`, `CASCADE`, `INTERCOM`
- **Emits**: `BUS`, `SHARED_BUS`, `PROPAGATE`, `ESCALATE`, `QUERY`, `CASCADE`, `INTERCOM`

---

## BUS message — the workhorse

`BUS` is the standard single-hop request/response. A satellite wraps an OVOS `Message` in a `HiveMessage(HiveMessageType.BUS, ovos_message)` and sends it to hivemind-core, which:

1. Checks the client's `allowed_types` whitelist — if the OVOS message type is not allowed, the message is dropped
2. Runs the configured policy chain (by default `OVOSAgentPolicy`) which injects per-client session data including skill/intent blacklists
3. Injects the OVOS message into the local bus with routing context (`source`, `destination`, `peer`, `session`)
4. Routes all OVOS replies back to the originating satellite by reading `context["destination"]`

The `Message.reply()` mechanism (from `ovos-bus-client`) swaps `source` ↔ `destination` on every reply, so all downstream skill messages (`speak`, `intent.handled`, etc.) automatically carry the originating satellite's peer ID in `destination`. `hivemind-core` reads that field and routes each reply to the correct connection.

![A BUS message and its reply, one hop each way](https://raw.githubusercontent.com/JarbasHiveMind/HiveMind-core/dev/resources/bus.gif)

---

## SHARED_BUS — passive monitoring

`SHARED_BUS` lets a satellite share its own local OVOS bus with hivemind-core passively. hivemind-core observes but does not inject the messages into its own bus. Typically used by the [ovos-skill-fallback-hivemind](https://github.com/JarbasHiveMind/ovos-skill-fallback-hivemind) skill.

![SHARED_BUS mirrors a satellite's local bus up to hivemind-core](https://raw.githubusercontent.com/JarbasHiveMind/HiveMind-core/dev/resources/shared_bus.gif)

---

## INTERCOM — end-to-end encrypted peer-to-peer

`INTERCOM` messages are sealed in a **hybrid envelope**: a random AES-256-GCM session key is wrapped with the **recipient node's RSA public key** (PKCS#1 OAEP) and the payload is encrypted with AES-256-GCM. The envelope carries base64 fields `encrypted_key`, `ciphertext`, `tag`, `nonce`, and `signature`. Intermediate nodes — including hivemind-core — cannot read the payload. Only the target node, which holds the corresponding RSA private key, can unwrap the session key and decrypt it. After decryption, the node verifies the sender's RSA signature (PSS over SHA-256) using the sender's public key (stored in the trust store).

`INTERCOM` is usually the payload of a transport message (`ESCALATE` or `PROPAGATE`) so it reaches its destination through the mesh. Intermediate nodes forward it without being able to read it.

---

## Request/response — QUERY

`QUERY` is like `ESCALATE`, but the node that handles the request sends a response back to the originator. It is the right tool when you need a definite answer from *one* node in the chain.

**Request flow:**

1. A satellite wraps an OVOS `Message` in `HiveMessage(HiveMessageType.QUERY, bus_hive_message)` and sends it to hivemind-core, including a `query_id` in the message metadata.
2. Each instance in the chain attempts to answer from its local agent (within a timeout). If the local agent produces a response, that instance wraps it as a QUERY response and sends it back downstream.
3. If the local agent does not answer, the instance forwards the QUERY upstream via `query_to_master`. At the top-level master with no upstream, a `hive.query.timeout` error response is returned instead.

**Response envelope:**

A QUERY response is itself a `HiveMessage(HiveMessageType.QUERY, ...)` with `metadata["is_response"] = True`. The payload wraps an inner `BUS` `HiveMessage`, which in turn carries the OVOS reply `Message`. Fields in `metadata`:

| Field | Value |
|---|---|
| `is_response` | `True` |
| `query_id` | UUID correlating request and response |
| `originator_peer` | Peer that issued the original request |
| `responder_peer` | Peer that produced the answer |

**Relay nodes** receiving a QUERY response with `is_response = True` route it toward `originator_peer` without re-processing it as a new request.

---

## Scatter/gather — CASCADE

`CASCADE` is like `PROPAGATE`, but every reachable node may answer. Use it when you want to collect responses from across the hive and pick the best one — for example, asking all nodes for their current status or broadcasting a question to all connected agents.

**Request flow:**

1. A satellite sends `HiveMessage(HiveMessageType.CASCADE, bus_hive_message)` with a `query_id`.
2. Each instance that receives the CASCADE: (a) tries its local agent and, if it gets a response, sends that back as a CASCADE response; (b) forwards the CASCADE onward to all other connected peers and upstream.
3. Multiple responses can arrive at the originator from different nodes.

**Response collection:**

On the client side, `CascadeAggregator` buffers responses for a configurable `cascade_timeout` (default 5 s). Once the timeout expires — or the expected number of responses has arrived (derived from `HiveMapper.nodes`) — a `cascade_select_callback` picks the winning response and delivers it. The default select callback chooses randomly; applications should supply a domain-appropriate selector.

**Trust model:**

Unlike QUERY (where responses come from a known upstream chain), CASCADE responses can originate from any node in the hive. The select callback is responsible for evaluating and choosing among responses that may come from untrusted nodes.

---

## PING and topology mapping

`PING` is always wrapped inside a `PROPAGATE` — it floods to all reachable nodes. There is no separate reply message type: each node that receives a `PING` re-emits *its own* `PING` (carrying the same `flood_id`), flooded onward via `PROPAGATE`. Receivers deduplicate by `flood_id` so each probe is processed once. The PING payload is `{flood_id, peer, site_id, timestamp}`. `HiveMapper` in `hivemind_core.hive_map` observes these re-emitted PINGs to build a topology map of the live hive.

---

## Session and context keys

When a satellite sends a BUS message, hivemind-core injects routing metadata into `Message.context` before passing it to OVOS:

| Key | Value | Purpose |
|---|---|---|
| `context["peer"]` | Satellite peer ID, e.g. `"HiveMindV0.0@127.0.0.1:8222/0"` | Identifies the originating satellite |
| `context["source"]` | Same as `peer` | OVOS origin identifier |
| `context["destination"]` | `"skills"` by default | Prevents message being treated as a broadcast |
| `context["session"]` | Serialized `Session` dict | Per-satellite session state |
| `context["session"]["session_id"]` | Stable UUID per satellite connection | Maintains conversational state across utterances |
| `context["session"]["blacklisted_skills"]` | List of skill IDs | Enforced by `IntentService` |
| `context["session"]["blacklisted_intents"]` | List of intent names | Enforced by `IntentService` |

After `Message.reply()` is called by OVOS, `source` and `destination` are swapped, so `destination` becomes the satellite's peer ID. `HiveMindListenerProtocol.handle_internal_mycroft()` reads this field to route the response to the correct satellite connection.

---

## Protocol versions

The hivemind-core `ProtocolVersion` enum bumps capabilities one step at a time:

| Feature | ZERO (v0) | ONE (v1) | TWO (v2) | THREE (v3) |
|---|:---:|:---:|:---:|:---:|
| JSON serialization | ✅ | ✅ | ✅ | ✅ |
| Handshake (password / RSA) | ❌ | ✅ | ✅ | — |
| Binary serialization | ❌ | ❌ | ✅ | ✅ |
| Noise handshake, always-encrypted | ❌ | ❌ | ❌ | ✅ |

- **v0** — JSON only, no handshake, no binary framing; uses only a pre-shared encryption key (the legacy, deprecated `Encryption Key` in `add-client` output).
- **v1** — adds the handshake (password salted-hash + PBKDF2, or RSA), the RSA identity, and zlib compression.
- **v2** — additionally enables binary framing.
- **v3** — replaces the v1/v2 handshake with a **Noise** handshake (`Noise_XXpsk2_25519_ChaChaPoly_SHA256` by default), giving an always-encrypted, forward-secret session. This is the current default for capable clients. See [Security → Handshake and encryption](security.md#handshake-and-encryption).

The server admits a connection only if it can negotiate at least `min_protocol_version` (config default **2**, so the oldest JSON-only / no-binary v0/v1 clients are refused by default), and advertises the highest version both sides support (`THREE` whenever the Noise primitive is available and a password is configured for the client). This `ProtocolVersion` enum is distinct from the binary-serialization `PROTOCOL_VERSION` constant in `serialization.py`.

!!! note "You don't negotiate v2 to send binary"
    Whether a v1/v2 connection uses binary framing is gated by the `binarize` boolean exchanged in the handshake, **not** by negotiating `ProtocolVersion.TWO` on its own. Under v3 the session is always encrypted and binary-capable. See [Protocol Spec](../developers/protocol-spec.md) for the framing and Noise details.

---

## Source

Validated against the HiveMind source:

- [`hivemind_bus_client/message.py`](https://github.com/JarbasHiveMind/hivemind-websocket-client/blob/HEAD/hivemind_bus_client/message.py) — `HiveMessage` and the `HiveMessageType` enum
- [`hivemind_bus_client/serialization.py`](https://github.com/JarbasHiveMind/hivemind-websocket-client/blob/HEAD/hivemind_bus_client/serialization.py) — JSON/binary serialization and the `PROTOCOL_VERSION` constant
- [`hivemind_core/protocol.py`](https://github.com/JarbasHiveMind/HiveMind-core/blob/HEAD/hivemind_core/protocol.py) — listener/client roles, `binarize` gating, routing, and the `ProtocolVersion` enum
