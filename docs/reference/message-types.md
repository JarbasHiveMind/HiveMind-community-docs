# HiveMessage Types Reference

Every message crossing a hive is one of a small, fixed set of kinds — a `BUS` request, a
`BROADCAST` rolling downhill, an `INTERCOM` sealed for a single recipient. This is the
catalogue of all of them: what each one is for, which way it flows, and what it carries.
When you're reading a packet capture or wiring up routing and need to know precisely what
a `CASCADE` does, look it up here.

!!! abstract "In a nutshell"
    - Message types fall into **payload** (`BUS`, `SHARED_BUS`, `BINARY`), **transport** (`ESCALATE`, `BROADCAST`, `PROPAGATE`, `QUERY`, `CASCADE`, `INTERCOM`, `RENDEZVOUS`), **discovery** (`PING`), and **connection** (`HELLO`, `HANDSHAKE`) categories.
    - Transport verbs wrap another `HiveMessage` to route it up (`ESCALATE`), down (`BROADCAST`), or everywhere (`PROPAGATE`).
    - `QUERY` expects one correlated response; `CASCADE` is scatter/gather with a select callback.
    - Skim the [Quick reference](#quick-reference) table first; reach for the per-type sections for details.

All message types are defined in `HiveMessageType` in `hivemind_bus_client.message`.

---

## Quick reference

If you only need to jog your memory, this table is the whole page — every type, its
category, which way it flows, and what it's for, on one screen. Skim the Category column
first: it groups the thirteen types into the four jobs they do, and the per-type sections
below expand whichever one you landed on. The Wire value column is the exact string to put
in the JSON `msg_type` field. Two of them are not the lowercased enum name — `HANDSHAKE` is
`shake` and `BINARY` is `bin` — so copy those two rather than derive them.

| Type | Wire value | Category | Direction | Purpose |
|---|---|---|---|---|
| `BUS` | `bus` | Payload | Bidirectional | Single-hop message to/from AI back-end |
| `SHARED_BUS` | `shared_bus` | Payload | Satellite → hivemind-core | Passive monitoring of satellite's local OVOS bus |
| `BINARY` | `bin` | Payload | Bidirectional | Raw binary data (audio, images, files) |
| `ESCALATE` | `escalate` | Transport | Satellite → hivemind-core (up) | Multi-hop upward routing |
| `BROADCAST` | `broadcast` | Transport | hivemind-core → All (down) | Multi-hop downward routing |
| `PROPAGATE` | `propagate` | Transport | Bidirectional | All-directions flood |
| `QUERY` | `query` | Transport | Bidirectional | Routed request with a single expected response |
| `CASCADE` | `cascade` | Transport | Bidirectional | Scatter/gather — all nodes may respond |
| `INTERCOM` | `intercom` | Transport | Point-to-point | End-to-end encrypted tunnel |
| `PING` | `ping` | Discovery | Bidirectional (via PROPAGATE) | Topology probe |
| `RENDEZVOUS` | `rendezvous` | Transport | Bidirectional | Deposit and collect mail for an offline peer |
| `HELLO` | `hello` | Connection | Bidirectional | Node announcement on connect |
| `HANDSHAKE` | `shake` | Connection | Bidirectional | Cryptographic key exchange |

Twelve of the thirteen have a 5-bit code for [binary framing](../developers/protocol-spec.md#message-type-encoding). `INTERCOM` never had one; it sends as JSON only. Code `11` belonged to a fourteenth type, `THIRDPRTY`, which has since been removed from the protocol; the code stays permanently reserved and unassigned rather than reused.

The sections that follow take the types one at a time, roughly in order of how often
you'll touch them — the everyday `BUS` first, the mesh-routing verbs next, and the
connection frames you never send by hand last.

---

## BUS

Standard single-hop message between satellite and hivemind-core.

- **Payload**: OVOS `Message` object
- **hivemind-core behaviour**: checks `allowed_types`, runs policy chain, injects into OVOS bus, routes replies back
- **Satellite behaviour**: send utterances and other OVOS messages; listen for responses

---

## SHARED_BUS

Passive monitoring of the satellite's local OVOS bus.

- **Direction**: satellite → hivemind-core
- **hivemind-core behaviour**: observes without injecting into the local bus
- **Use**: typically via [ovos-hivemind-pipeline-plugin](https://github.com/JarbasHiveMind/ovos-hivemind-pipeline-plugin)'s `slave_mode` setting

---

## BINARY

Raw binary payload.

- **Payload**: bytes + a 4-bit binary type indicator (see binary payload types below)
- **Permission**: the client `allowed_types` whitelist must not be empty. Binary payloads cross the same ACL gate as bus messages, and an empty whitelist denies every one of them with `acl_disallowed_type`. See [How the policy chain works](../concepts/security.md#how-the-policy-chain-works).
- **Processed by**: the configured binary data handler plugin (e.g. `hivemind-audio-binary-protocol`), and only after the policy chain allows the payload

### Binary payload types

| Value | Name | Purpose |
|---|---|---|
| 0 | UNDEFINED | Opaque binary |
| 1 | RAW_AUDIO | Continuous microphone stream |
| 2 | NUMPY_IMAGE | Numpy array image frame |
| 3 | FILE | File transfer |
| 4 | STT_AUDIO_TRANSCRIBE | Full utterance → return transcript |
| 5 | STT_AUDIO_HANDLE | Full utterance → transcribe and handle intent |
| 6 | TTS_AUDIO | Synthesized speech (hivemind-core → satellite) |

---

## THIRDPRTY (removed)

`THIRDPRTY` was a user-defined message type with no built-in HiveMind handling, meant as
an escape hatch for custom, application-specific payloads. It has been removed from the
protocol; `HiveMessageType` no longer has a `THIRDPRTY` member. Its old binary framing
code, `11`, stays permanently reserved and is never reused for a new type.

---

## ESCALATE

Wraps another `HiveMessage` and routes it upward through the hivemind-core chain.

- **Payload**: a nested `HiveMessage`
- **Loop control**: there is no hop counter and no TTL. Every relaying node appends a `route` hop whose `source` is its own public key, and a node drops any message whose `route` already lists that key. A client that does not append this hop makes its relayed frames impossible for peers to suppress.
- **Use**: satellite cannot handle a request locally; asks the parent hivemind-core

---

## BROADCAST

Wraps another `HiveMessage` and routes it downward to all satellites.

- **Payload**: a nested `HiveMessage`
- **Target filtering**: supports `target_site_id` to reach only satellites in a specific location
- **Sent by**: hivemind-core (typically triggered by a skill)

---

## PROPAGATE

Wraps another `HiveMessage` and floods it in all directions.

- **Payload**: a nested `HiveMessage`
- **Use**: mesh-wide announcements, topology discovery (PING is always wrapped in PROPAGATE)

---

## QUERY

A routed request that expects exactly one response, correlated by a `query_id`. Like `ESCALATE`, but the response travels back to the originator.

- **Payload**: a nested `HiveMessage` (typically wrapping a `BUS` message)
- **Permission**: the sending client must have `can_escalate = True`
- **hivemind-core behaviour (request)**: hivemind-core attempts to answer from its local agent (within a timeout). If the local agent answers, a QUERY response is sent back immediately. If not, the request is forwarded upstream via `query_to_master`. At the top-level master with no upstream, a `hive.query.timeout` error response is returned.
- **hivemind-core behaviour (response)**: messages with `metadata["is_response"] = True` are routed downstream toward the originator identified by `metadata["originator_peer"]`.
- **Response metadata fields**:

| Field | Description |
|---|---|
| `query_id` | UUID correlating request and response |
| `originator_peer` | Peer ID of the node that issued the original request |
| `responder_peer` | Peer ID of the node that answered |
| `is_response` | `true` — distinguishes response from request |

The response payload wraps an inner `BUS` HiveMessage; the inner BUS message carries the OVOS agent's reply.

---

## CASCADE

A scatter/gather request: like `PROPAGATE`, but every reachable node may answer. Responses are collected and a **select callback** disambiguates to pick the best answer.

- **Payload**: a nested `HiveMessage` (wrapping a `BUS` message)
- **Permission**: the sending client must have `can_propagate = True`
- **hivemind-core behaviour (request)**: hivemind-core tries its local agent, then forwards the CASCADE to all other connected peers and upstream. Every node that can answer sends a response.
- **hivemind-core behaviour (response)**: response messages (`is_response = True`) are routed back toward the originator. At the originator's hivemind-core, a `cascade_select_callback` (if configured) collects all responses and picks a winner. Without a select callback, each response is forwarded individually.
- **Client behaviour**: `CascadeAggregator` on the client side buffers responses for a `cascade_timeout` window (default 5 s). Once the timer expires — or the expected number of responses has arrived — the `cascade_select_callback` picks the best response and delivers it.
- **Trust**: unlike QUERY responses (which come from a known upstream node), CASCADE responses can originate from any node in the hive. The select callback is responsible for choosing among potentially untrusted responses.
- **Response metadata fields**: same as QUERY (`query_id`, `originator_peer`, `responder_peer`, `is_response`).

---

## INTERCOM

End-to-end encrypted point-to-point message.

- **Payload**: a hybrid envelope — a random AES-256-GCM session key wrapped with the target node's RSA public key (PKCS#1 OAEP), with the body encrypted under AES-256-GCM. Base64 fields: `encrypted_key`, `ciphertext`, `tag`, `nonce`, `signature`. Only the target node (holding the RSA private key) can unwrap the session key and decrypt.
- **hivemind-core accepts both envelope shapes**: when `encrypted_key` is present it unwraps the hybrid envelope; otherwise it falls back to decrypting `ciphertext` directly with RSA. The signature always covers `ciphertext`, so verification is unchanged either way. The plain-RSA shape stays accepted for compatibility, but a raw RSA block caps the payload at roughly 214 bytes with a 2048-bit key, too small for a serialized bus message — only the hybrid shape can carry a real message. All three clients (sync, async, HTTP) send the hybrid shape.
- **Signature**: REQUIRED. The sender signs the raw ciphertext bytes, meaning the base64-decoded `ciphertext`, with PSS over SHA-256. The receiver checks it *before* decryption.
- **Routing**: typically wrapped in ESCALATE or PROPAGATE to reach the target
- **Intermediate nodes**: cannot read the content, but they do see the recipient. The outer `target_pubkey` field is cleartext, and a relay reads it to decide whether to consume the frame or forward it.
- **Origin check**: the target verifies the signature against the sender's pinned public key. A bad signature, a missing signature, or an unknown originator means the message is dropped
- **A dropped message is not relayed**: it goes no further to peers and is not escalated upstream
- **Plaintext payloads**: a node with `require_crypto` (the default) drops an `INTERCOM` that is not a signed envelope, and logs `dropping unauthenticated message`. `require_crypto` is a listener attribute set in code, not a `server.json` key, and clearing it gives up all proof of who sent the message
- **Binary framing**: `INTERCOM` has no 5-bit type code, so it cannot be binary-framed at all. Send it as a text frame.

---

## PING

Topology discovery.

- `PING` is always wrapped inside `PROPAGATE`
- There is no `PONG` reply type. Every node that receives a `PING` re-emits its own `PING` with the same `flood_id`, flooded onward via `PROPAGATE`, and receivers deduplicate by `flood_id`
- `flood_id` dedup is specific to PING. It sits on top of the [route-hop loop suppression](#escalate) that covers PROPAGATE, ESCALATE, CASCADE and PING alike
- **Payload**: `{flood_id, peer, site_id, timestamp, public_key}`. A satellite/client-originated PING also carries `lang`; hivemind-core's own self-originated PING omits it
- `HiveMapper` in `hivemind_bus_client.hive_map` observes the re-emitted PINGs to build a live topology map

---

## HELLO / HANDSHAKE

Connection management. Handled automatically by `HiveMessageBusClient` and `hivemind-core`.

- `HELLO` announces the node (node_id, RSA public key in PEM) — sent unencrypted
- `HANDSHAKE` performs the key exchange — sent unencrypted, establishes the session key. Two modes:
  - **Password mode** (`PasswordHandShake`): the session key is derived with PBKDF2-HMAC-SHA256, 100000 iterations
  - **RSA mode**: a random 32-byte secret is wrapped with the peer's RSA public key (no PBKDF2)
- After the handshake, a second `HELLO` carries session data, site_id, and client public key — this one IS encrypted, since it's sent once the session key already exists

> **`HANDSHAKE` and the first `HELLO` are the only plaintext frames.** `HANDSHAKE` always
> travels as plaintext JSON, since it carries the crypto negotiation before any key
> exists. A node's first `HELLO`, announcing itself, is plaintext too. Every frame after
> that — including the client's second `HELLO` — is encrypted like any other
> post-handshake traffic.
>
> **Request vs response disambiguation.** A `HANDSHAKE` is a *request* (capability advertisement, "please start") versus a *response* (carrying key material) **solely by the presence of the `envelope` field** — there is no type discriminator. No `envelope` ⇒ request; `envelope` present ⇒ response/material.
>
> For the full connection-setup sequence with exact field tables in both directions, see the [Handshake state machine](../developers/protocol-spec.md#handshake-state-machine) in the protocol spec.

---

## RENDEZVOUS

A store-and-forward dead drop, for two nodes that are never online at the same time. One deposits a message addressed to the other's **access key** on the rendezvous node; the recipient collects it whenever it next connects.

A rendezvous node is an ordinary hivemind-core node with the optional [hivemind-rendezvous](https://github.com/JarbasHiveMind/hivemind-rendezvous) package installed and `rendezvous.enabled` set in its config. It serves this type on the listener that already accepts clients, so being a rendezvous point costs no extra port or credentials.

Three commands, carried in the payload as `cmd`:

| `cmd` | Fields | Reply |
|---|---|---|
| `deposit` | `target_key`, `payload` (a serialised `INTERCOM` message), optional `ttl` | `deposit_id` |
| `collect` | none | `messages`: a list of `{deposit_id, payload}` |
| `ack` | `deposit_ids` | `removed` |

Only `INTERCOM` may be deposited: it is the one type the relay has no reason to look inside. That is a routing rule, not a confidentiality guarantee — nothing about the message *type* makes a payload encrypted, and the relay does not inspect one to check. Encrypt to the recipient's public key before depositing, which is what `hivemind_rendezvous.client.make_deposit_envelope` does.

**You never name a mailbox.** `collect` and `ack` act on the access key your connection authenticated with, so asking for another node's mail is not something the protocol can express.

The address is deliberately an access key and **not** a public key. A public key is announced in `HELLO` with no proof of possession, and it is public by design — it is the `INTERCOM` addressing key. Owning a mailbox by naming one would let any admitted client claim any other's mail, which is exactly the hole fixed in hivemind-core 4.13.2a1.

**Delivery is at-least-once.** `collect` leaves messages in place; they are deleted only when you `ack` their deposit ids. Expect an occasional duplicate, and ack only once the messages are safely in hand — anything unacked is handed out again next time.

**Limits.** A deposit is refused with `invalid_ttl` for a non-integer or non-positive `ttl`, `payload_too_large` above 256 KiB, `mailbox_full` at the per-mailbox cap, and `too_many_mailboxes` at the store cap. Messages expire after seven days.

A node that is not a rendezvous point replies `{"status": "error", "reason": "not_a_rendezvous_node"}`, which is deliberately distinct from a successful collect that returns nothing.

---

## Source

Validated against the HiveMind source:

- [`hivemind_bus_client/message.py`](https://github.com/JarbasHiveMind/hivemind-websocket-client/blob/HEAD/hivemind_bus_client/message.py) — `HiveMessageType` and the binary payload-type enum
- [`hivemind_bus_client/serialization.py`](https://github.com/JarbasHiveMind/hivemind-websocket-client/blob/HEAD/hivemind_bus_client/serialization.py) — the on-the-wire type encoding and HELLO/HANDSHAKE framing
