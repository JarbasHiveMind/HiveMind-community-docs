# HiveMessage Types Reference

All message types are defined in `HiveMessageType` in `hivemind_bus_client.message`.

## Quick reference

| Type | Category | Direction | Purpose |
|---|---|---|---|
| `BUS` | Payload | Bidirectional | Single-hop message to/from AI back-end |
| `SHARED_BUS` | Payload | Satellite → Hub | Passive monitoring of satellite's local OVOS bus |
| `THIRDPRTY` | Payload | Application-defined | User-defined payload, relayed opaquely |
| `BINARY` | Payload | Bidirectional | Raw binary data (audio, images, files) |
| `ESCALATE` | Transport | Satellite → Hub (up) | Multi-hop upward routing |
| `BROADCAST` | Transport | Hub → All (down) | Multi-hop downward routing |
| `PROPAGATE` | Transport | Bidirectional | All-directions flood |
| `QUERY` | Transport | Bidirectional | Routed request with a single expected response |
| `CASCADE` | Transport | Bidirectional | Scatter/gather — all nodes may respond |
| `INTERCOM` | Transport | Point-to-point | End-to-end encrypted tunnel |
| `PING` | Discovery | Bidirectional (via PROPAGATE) | Topology probe |
| `RENDEZVOUS` | Transport | Bidirectional | Reserved for rendezvous-nodes |
| `HELLO` | Connection | Bidirectional | Node announcement on connect |
| `HANDSHAKE` | Connection | Bidirectional | Cryptographic key exchange |

## BUS

Standard single-hop message between satellite and hub.

- **Payload**: OVOS `Message` object
- **Hub behaviour**: checks `allowed_types`, runs policy chain, injects into OVOS bus, routes replies back
- **Satellite behaviour**: send utterances and other OVOS messages; listen for responses

## SHARED_BUS

Passive monitoring of the satellite's local OVOS bus.

- **Direction**: satellite → hub
- **Hub behaviour**: observes without injecting into the local bus
- **Use**: typically via [ovos-skill-fallback-hivemind](https://github.com/JarbasHiveMind/ovos-skill-fallback-hivemind)

## BINARY

Raw binary payload.

- **Payload**: bytes + a 4-bit binary type indicator (see binary payload types below)
- **Processed by**: the configured binary data handler plugin (e.g. `hivemind-audio-binary-protocol`)

### Binary payload types

| Value | Name | Purpose |
|---|---|---|
| 0 | UNDEFINED | Opaque binary |
| 1 | RAW_AUDIO | Continuous microphone stream |
| 2 | NUMPY_IMAGE | Numpy array image frame |
| 3 | FILE | File transfer |
| 4 | STT_AUDIO_TRANSCRIBE | Full utterance → return transcript |
| 5 | STT_AUDIO_HANDLE | Full utterance → transcribe and handle intent |
| 6 | TTS_AUDIO | Synthesized speech (hub → satellite) |

## ESCALATE

Wraps another `HiveMessage` and routes it upward through the hub chain.

- **Payload**: a nested `HiveMessage`
- **Hop limit**: finite; loops are prevented
- **Use**: satellite cannot handle a request locally; asks the parent hub

## BROADCAST

Wraps another `HiveMessage` and routes it downward to all satellites.

- **Payload**: a nested `HiveMessage`
- **Target filtering**: supports `target_site_id` to reach only satellites in a specific location
- **Sent by**: hub (typically triggered by a skill)

## PROPAGATE

Wraps another `HiveMessage` and floods it in all directions.

- **Payload**: a nested `HiveMessage`
- **Use**: mesh-wide announcements, topology discovery (PING is always wrapped in PROPAGATE)

## QUERY

A routed request that expects exactly one response, correlated by a `query_id`. Like `ESCALATE`, but the response travels back to the originator.

- **Payload**: a nested `HiveMessage` (typically wrapping a `BUS` message)
- **Permission**: the sending client must have `can_escalate = True`
- **Hub behaviour (request)**: the hub attempts to answer from its local agent (within a timeout). If the local agent answers, a QUERY response is sent back immediately. If not, the request is forwarded upstream via `query_to_master`. At the top-level master with no upstream, a `hive.query.timeout` error response is returned.
- **Hub behaviour (response)**: messages with `metadata["is_response"] = True` are routed downstream toward the originator identified by `metadata["originator_peer"]`.
- **Response metadata fields**:

| Field | Description |
|---|---|
| `query_id` | UUID correlating request and response |
| `originator_peer` | Peer ID of the node that issued the original request |
| `responder_peer` | Peer ID of the node that answered |
| `is_response` | `true` — distinguishes response from request |

The response payload wraps an inner `BUS` HiveMessage; the inner BUS message carries the OVOS agent's reply.

## CASCADE

A scatter/gather request: like `PROPAGATE`, but every reachable node may answer. Responses are collected and a **select callback** disambiguates to pick the best answer.

- **Payload**: a nested `HiveMessage` (wrapping a `BUS` message)
- **Permission**: the sending client must have `can_propagate = True`
- **Hub behaviour (request)**: the hub tries its local agent, then forwards the CASCADE to all other connected peers and upstream. Every node that can answer sends a response.
- **Hub behaviour (response)**: response messages (`is_response = True`) are routed back toward the originator. At the originator's hub, a `cascade_select_callback` (if configured) collects all responses and picks a winner. Without a select callback, each response is forwarded individually.
- **Client behaviour**: `CascadeAggregator` on the client side buffers responses for a `cascade_timeout` window (default 5 s). Once the timer expires — or the expected number of responses has arrived — the `cascade_select_callback` picks the best response and delivers it.
- **Trust**: unlike QUERY responses (which come from a known upstream node), CASCADE responses can originate from any node in the hive. The select callback is responsible for choosing among potentially untrusted responses.
- **Response metadata fields**: same as QUERY (`query_id`, `originator_peer`, `responder_peer`, `is_response`).

## INTERCOM

End-to-end encrypted point-to-point message.

- **Payload**: a hybrid envelope — a random AES-256-GCM session key wrapped with the target node's RSA public key (PKCS#1 OAEP), with the body encrypted under AES-256-GCM. Base64 fields: `encrypted_key`, `ciphertext`, `tag`, `nonce`, `signature`. Only the target node (holding the RSA private key) can unwrap the session key and decrypt.
- **Signature**: sender's RSA private key signs (PSS over SHA-256)
- **Routing**: typically wrapped in ESCALATE or PROPAGATE to reach the target
- **Intermediate nodes**: cannot read the content or determine the recipient

## PING

Topology discovery.

- `PING` is always wrapped inside `PROPAGATE`
- There is no `PONG` reply type. Every node that receives a `PING` re-emits its own `PING` with the same `flood_id`, flooded onward via `PROPAGATE`; receivers deduplicate by `flood_id`
- **Payload**: `{flood_id, peer, site_id, timestamp}`
- `HiveMapper` in `hivemind_core.hive_map` observes the re-emitted PINGs to build a live topology map

## HELLO / HANDSHAKE

Connection management. Handled automatically by `HiveMessageBusClient` and `hivemind-core`.

- `HELLO` announces the node (node_id, RSA public key in PEM) — sent unencrypted
- `HANDSHAKE` performs the key exchange — sent unencrypted, establishes the session key. Two modes:
  - **Password mode** (`PasswordHandShake`): the session key is derived with PBKDF2-HMAC-SHA256, 100000 iterations
  - **RSA mode**: a random 32-byte secret is wrapped with the peer's RSA public key (no PBKDF2)
- After the handshake, a second `HELLO` carries session data, site_id, and client public key

> **Plaintext, even after crypto is established.** Both `HELLO` and `HANDSHAKE` always travel as plaintext JSON — including the client's *second* `HELLO`, which is sent after the session key exists but is still unencrypted. Only `BUS`/`QUERY`/other payload traffic is encrypted.
>
> **Request vs response disambiguation.** A `HANDSHAKE` is a *request* (capability advertisement, "please start") versus a *response* (carrying key material) **solely by the presence of the `envelope` field** — there is no type discriminator. No `envelope` ⇒ request; `envelope` present ⇒ response/material.
>
> For the full connection-setup sequence with exact field tables in both directions, see the [Handshake state machine](../developers/protocol-spec.md#handshake-state-machine) in the protocol spec.

## RENDEZVOUS

Reserved for rendezvous-nodes. Defined in the `HiveMessageType` enum but not yet wired into general routing.

## THIRDPRTY

User-defined payload. Relayed opaquely by the hub without any special processing. Use it for application-level protocols layered on top of HiveMind.
