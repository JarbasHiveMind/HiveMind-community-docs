# Protocol

In plain terms: this is the shared language HiveMind nodes speak. Every message is wrapped in a small envelope that says what kind of message it is and where it should go, so hubs and satellites can route it correctly.

HiveMind defines a message protocol that runs over pluggable transports. Every message on the wire is a `HiveMessage` — a JSON envelope (or binary-encoded equivalent in protocol v2) carrying a `msg_type` and a payload.

## HiveMessage types

Each message carries a `msg_type` from the `HiveMessageType` enum, defined in `hivemind_bus_client.message`.

### Payload messages

These carry OVOS `Message` objects (or other application payloads) as their content.

| Type | Purpose | Direction |
|---|---|---|
| **`BUS`** | Single-hop message to/from the AI back-end | Bidirectional |
| **`SHARED_BUS`** | Passive monitoring of a satellite's local OVOS bus | Satellite → Hub |
| **`THIRDPRTY`** | User-defined payload, relayed opaquely | Application-defined |
| **`BINARY`** | Raw binary data (audio, images, files) | Bidirectional |

### Transport / routing messages

These wrap another `HiveMessage` as their payload and control how it is routed through the mesh.

| Type | Purpose | Direction |
|---|---|---|
| **`ESCALATE`** | Multi-hop upward — satellite asks parent for help | Satellite → Hub (up the chain) |
| **`BROADCAST`** | Multi-hop downward — hub commands all satellites | Hub → All satellites (down the chain) |
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
| **`HELLO`** | Node announcement on connection (carries node ID and RSA public key) |
| **`HANDSHAKE`** | Cryptographic key exchange (password mode → PBKDF2 envelope, or RSA mode → RSA-wrapped secret) |

## Roles

### Hub (listener protocol)

- **Accepts**: `BUS`, `SHARED_BUS`, `PROPAGATE`, `ESCALATE`, `QUERY`, `CASCADE`, `INTERCOM`
- **Emits**: `BUS`, `PROPAGATE`, `BROADCAST`, `QUERY`, `CASCADE`, `INTERCOM`

### Satellite (client protocol)

- **Accepts**: `BUS`, `PROPAGATE`, `BROADCAST`, `QUERY`, `CASCADE`, `INTERCOM`
- **Emits**: `BUS`, `SHARED_BUS`, `PROPAGATE`, `ESCALATE`, `QUERY`, `CASCADE`, `INTERCOM`

## BUS message — the workhorse

`BUS` is the standard single-hop request/response. A satellite wraps an OVOS `Message` in a `HiveMessage(HiveMessageType.BUS, ovos_message)` and sends it to the hub. The hub:

1. Checks the client's `allowed_types` whitelist — if the OVOS message type is not allowed, the message is dropped
2. Runs the configured policy chain (by default `OVOSAgentPolicy`) which injects per-client session data including skill/intent blacklists
3. Injects the OVOS message into the local bus with routing context (`source`, `destination`, `peer`, `session`)
4. Routes all OVOS replies back to the originating satellite by reading `context["destination"]`

The `Message.reply()` mechanism (from `ovos-bus-client`) swaps `source` ↔ `destination` on every reply, so all downstream skill messages (`speak`, `intent.handled`, etc.) automatically carry the originating satellite's peer ID in `destination`. `hivemind-core` reads that field and routes each reply to the correct connection.

## SHARED_BUS — passive monitoring

`SHARED_BUS` lets a satellite share its own local OVOS bus with the hub passively. The hub observes but does not inject the messages into its own bus. Typically used by the [ovos-skill-fallback-hivemind](https://github.com/JarbasHiveMind/ovos-skill-fallback-hivemind) skill.

## INTERCOM — end-to-end encrypted peer-to-peer

`INTERCOM` messages are sealed in a **hybrid envelope**: a random AES-256-GCM session key is wrapped with the **recipient node's RSA public key** (PKCS#1 OAEP) and the payload is encrypted with AES-256-GCM. The envelope carries base64 fields `encrypted_key`, `ciphertext`, `tag`, `nonce`, and `signature`. Intermediate nodes — including the hub — cannot read the payload. Only the target node, which holds the corresponding RSA private key, can unwrap the session key and decrypt it. After decryption, the node verifies the sender's RSA signature (PSS over SHA-256) using the sender's public key (stored in the trust store).

`INTERCOM` is usually the payload of a transport message (`ESCALATE` or `PROPAGATE`) so it reaches its destination through the mesh. Intermediate nodes forward it without being able to read it.

## Request/response — QUERY

`QUERY` is like `ESCALATE`, but the node that handles the request sends a response back to the originator. It is the right tool when you need a definite answer from *one* node in the chain.

**Request flow:**

1. A satellite wraps an OVOS `Message` in `HiveMessage(HiveMessageType.QUERY, bus_hive_message)` and sends it to the hub, including a `query_id` in the message metadata.
2. Each hub in the chain attempts to answer from its local agent (within a timeout). If the local agent produces a response, the hub wraps it as a QUERY response and sends it back downstream.
3. If the local agent does not answer, the hub forwards the QUERY upstream via `query_to_master`. At the top-level master with no upstream, a `hive.query.timeout` error response is returned instead.

**Response envelope:**

A QUERY response is itself a `HiveMessage(HiveMessageType.QUERY, ...)` with `metadata["is_response"] = True`. The payload wraps an inner `BUS` `HiveMessage`, which in turn carries the OVOS reply `Message`. Fields in `metadata`:

| Field | Value |
|---|---|
| `is_response` | `True` |
| `query_id` | UUID correlating request and response |
| `originator_peer` | Peer that issued the original request |
| `responder_peer` | Peer that produced the answer |

**Relay nodes** receiving a QUERY response with `is_response = True` route it toward `originator_peer` without re-processing it as a new request.

## Scatter/gather — CASCADE

`CASCADE` is like `PROPAGATE`, but every reachable node may answer. Use it when you want to collect responses from across the hive and pick the best one — for example, asking all nodes for their current status or broadcasting a question to all connected agents.

**Request flow:**

1. A satellite sends `HiveMessage(HiveMessageType.CASCADE, bus_hive_message)` with a `query_id`.
2. Each hub that receives the CASCADE: (a) tries its local agent and, if it gets a response, sends that back as a CASCADE response; (b) forwards the CASCADE onward to all other connected peers and upstream.
3. Multiple responses can arrive at the originator from different nodes.

**Response collection:**

On the client side, `CascadeAggregator` buffers responses for a configurable `cascade_timeout` (default 5 s). Once the timeout expires — or the expected number of responses has arrived (derived from `HiveMapper.nodes`) — a `cascade_select_callback` picks the winning response and delivers it. The default select callback chooses randomly; applications should supply a domain-appropriate selector.

**Trust model:**

Unlike QUERY (where responses come from a known upstream chain), CASCADE responses can originate from any node in the hive. The select callback is responsible for evaluating and choosing among responses that may come from untrusted nodes.

## PING and topology mapping

`PING` is always wrapped inside a `PROPAGATE` — it floods to all reachable nodes. There is no separate reply message type: each node that receives a `PING` re-emits *its own* `PING` (carrying the same `flood_id`), flooded onward via `PROPAGATE`. Receivers deduplicate by `flood_id` so each probe is processed once. The PING payload is `{flood_id, peer, site_id, timestamp}`. `HiveMapper` in `hivemind_core.hive_map` observes these re-emitted PINGs to build a topology map of the live hive.

## Session and context keys

When a satellite sends a BUS message, the hub injects routing metadata into `Message.context` before passing it to OVOS:

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

## Protocol versions

The hivemind-core `ProtocolVersion` enum bumps capabilities one step at a time:

| Feature | ZERO (v0) | ONE (v1) | TWO (v2) |
|---|:---:|:---:|:---:|
| JSON serialization | ✅ | ✅ | ✅ |
| Handshake (password or RSA) | ❌ | ✅ | ✅ |
| Binary serialization | ❌ | ❌ | ✅ |

Protocol v0 is JSON only, with no handshake and no binary framing — it uses only a pre-shared encryption key (the legacy, deprecated `Encryption Key` in `add-client` output). Protocol v1 adds the handshake (password/PBKDF2 or RSA), the RSA identity, and zlib compression. Protocol v2 additionally enables binary framing. (This `ProtocolVersion` enum is distinct from the binary-serialization `PROTOCOL_VERSION` constant in `serialization.py`.) New clients should use the highest version both sides support.

!!! note "You don't negotiate v2 to send binary"
    In practice v2 is effectively unreachable as a *negotiated* version: whether a connection uses binary framing is gated by the `binarize` boolean exchanged in the handshake, **not** by negotiating `ProtocolVersion.TWO`. You enable binary by setting `binarize`, not by bumping the protocol version. See [Protocol Spec](../developers/protocol-spec.md) for the framing details.

## Source

Validated against the HiveMind source:

- [`hivemind_bus_client/message.py`](https://github.com/JarbasHiveMind/hivemind-websocket-client/blob/HEAD/hivemind_bus_client/message.py) — `HiveMessage` and the `HiveMessageType` enum
- [`hivemind_bus_client/serialization.py`](https://github.com/JarbasHiveMind/hivemind-websocket-client/blob/HEAD/hivemind_bus_client/serialization.py) — JSON/binary serialization and the `PROTOCOL_VERSION` constant
- [`hivemind_core/protocol.py`](https://github.com/JarbasHiveMind/HiveMind-core/blob/HEAD/hivemind_core/protocol.py) — listener/client roles, `binarize` gating, routing, and the `ProtocolVersion` enum
