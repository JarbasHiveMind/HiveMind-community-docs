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

That `msg_type` stamp is the first thing every node reads, and it always comes from one
list: the `HiveMessageType` enum in `hivemind_bus_client.message`. The list is long, but
it sorts cleanly into three jobs — carry content, route content, or set up the
connection. Meet them a group at a time.

### Payload messages

These are the ones with something *inside* — an OVOS `Message`, or your own application
data. If a message is doing actual work, it's almost certainly one of these:

| Type | Purpose | Direction |
|---|---|---|
| **`BUS`** | Single-hop message to/from the AI back-end | Bidirectional |
| **`SHARED_BUS`** | Passive monitoring of a satellite's local OVOS bus | Satellite → hivemind-core |
| **`THIRDPRTY`** | User-defined payload, relayed opaquely | Application-defined |
| **`BINARY`** | Raw binary data (audio, images, files) | Bidirectional |

Ninety-nine percent of the time you'll be sending `BUS`. The other three are for
special jobs — mirroring a bus, smuggling your own payload, or shipping audio.

### Transport / routing messages

The next group carries nothing of its own. Each one *wraps another `HiveMessage`* and
exists only to steer it — up the tree, down the tree, or out in every direction. Read
the Direction column as the shape of the journey:

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

These only earn their keep once you have more than one hivemind-core — a plain
one-server hive never needs them. The [Mesh Topology](mesh.md) page shows them in
motion; the deep dives on `QUERY`, `CASCADE`, `INTERCOM`, and `PING` are further down
this page.

### Connection management messages

The last group you'll never send by hand — the library and hivemind-core exchange them
for you before your first real message even leaves. They're listed here so you recognise
them in a packet capture, not because you have to do anything with them:

| Type | Purpose |
|---|---|
| **`HELLO`** | Node announcement on connection (carries node ID and public key) |
| **`HANDSHAKE`** | Cryptographic key exchange. Protocol **v3** runs a **Noise** handshake (always-encrypted session); the legacy **v1/v2** path uses a password (salted-hash + PBKDF2) or RSA envelope. See [Security](security.md#handshake-and-encryption). |

---

## Roles

A message type isn't a free-for-all: not every node may *send* every type, and not every
node will *accept* one. hivemind-core and a satellite play different parts, so they
subscribe to different halves of the list. Here's who does what:

### hivemind-core (listener protocol)

- **Accepts**: `BUS`, `SHARED_BUS`, `PROPAGATE`, `ESCALATE`, `QUERY`, `CASCADE`, `INTERCOM`
- **Emits**: `BUS`, `PROPAGATE`, `BROADCAST`, `QUERY`, `CASCADE`, `INTERCOM`

### Satellite (client protocol)

- **Accepts**: `BUS`, `PROPAGATE`, `BROADCAST`, `QUERY`, `CASCADE`, `INTERCOM`
- **Emits**: `BUS`, `SHARED_BUS`, `PROPAGATE`, `ESCALATE`, `QUERY`, `CASCADE`, `INTERCOM`

Notice the asymmetry: only a satellite emits `ESCALATE` (it asks upward) and `SHARED_BUS`
(it mirrors its own bus), and only hivemind-core emits `BROADCAST` (it commands
downward). The verb you're allowed to send depends on which end of the connection you
are.

---

## BUS message — the workhorse

Everything above is scaffolding for this one message. When you ask "what's the weather?",
a `BUS` message is what carries the question and a `BUS` message is what carries the
answer home. It's a plain single hop: a satellite wraps an OVOS `Message` in a
`HiveMessage(HiveMessageType.BUS, ovos_message)` and sends it to hivemind-core, which
then walks four steps:

1. Checks the client's `allowed_types` whitelist — if the OVOS message type is not allowed, the message is dropped
2. Runs the configured policy chain (by default `OVOSAgentPolicy`) which injects per-client session data including skill/intent blacklists
3. Injects the OVOS message into the local bus with routing context (`source`, `destination`, `peer`, `session`)
4. Routes all OVOS replies back to the originating satellite by reading `context["destination"]`

Step 4 is the clever bit — how does a `speak` message that a skill emits *seconds later*
know which satellite to go back to? The trick is `Message.reply()` (from
`ovos-bus-client`). It swaps `source` ↔ `destination` on every reply, so every downstream
skill message — `speak`, `intent.handled`, and the rest — automatically carries the
originating satellite's peer ID in `destination`. hivemind-core reads that field and
routes each reply to the right connection.

The upshot for you: you send an utterance and the answer comes back to *your* device, not
someone else's, without your ever addressing it. Two satellites can ask at the same
moment and neither hears the other's reply. Here it is, one hop each way:

![A BUS message and its reply, one hop each way](https://raw.githubusercontent.com/JarbasHiveMind/HiveMind-core/dev/resources/bus.gif)

---

## SHARED_BUS — passive monitoring

`BUS` is a satellite *asking* hivemind-core to do something. `SHARED_BUS` is the
opposite posture: a satellite letting hivemind-core *watch* its own local OVOS bus,
read-only. hivemind-core sees the traffic but never injects it into its own bus — think
one-way mirror, not a shared room. The [ovos-skill-fallback-hivemind](https://github.com/JarbasHiveMind/ovos-skill-fallback-hivemind)
skill is the usual reason you'd reach for it.

![SHARED_BUS mirrors a satellite's local bus up to hivemind-core](https://raw.githubusercontent.com/JarbasHiveMind/HiveMind-core/dev/resources/shared_bus.gif)

---

## INTERCOM — end-to-end encrypted peer-to-peer

Every message so far passes *through* hivemind-core, which means hivemind-core can read
it. `INTERCOM` is for when two nodes want to say something that even the servers relaying
it can't overhear — a sealed letter passed hand to hand, where every courier can carry it
but only the addressee can open it.

The sealing is a **hybrid envelope**: a random AES-256-GCM session key is wrapped with the **recipient node's RSA public key** (PKCS#1 OAEP) and the payload is encrypted with AES-256-GCM. The envelope carries base64 fields `encrypted_key`, `ciphertext`, `tag`, `nonce`, and `signature`. Intermediate nodes — including hivemind-core — cannot read the payload. Only the target node, which holds the corresponding RSA private key, can unwrap the session key and decrypt it. After decryption, the node verifies the sender's RSA signature (PSS over SHA-256) using the sender's public key (stored in the trust store).

`INTERCOM` is usually the payload of a transport message (`ESCALATE` or `PROPAGATE`) so it reaches its destination through the mesh. Intermediate nodes forward it without being able to read it.

---

## Request/response — QUERY

The next three types only matter once your hive is more than one server deep — when a
question might have to travel past the first hivemind-core to find an answer. `QUERY` is
the first of them.

Picture a guest assistant that can't answer "what's on my shared calendar?" on its own,
so it passes the question up to the household server. `QUERY` is `ESCALATE` with a promise
of a reply: it climbs the chain like an escalation, but the node that finally answers
sends the response all the way back down to whoever asked. Reach for it when you need one
definite answer from *one* node somewhere above you.

**Request flow:**

1. A satellite wraps an OVOS `Message` in `HiveMessage(HiveMessageType.QUERY, bus_hive_message)` and sends it to hivemind-core, including a `query_id` in the message metadata.
2. Each instance in the chain attempts to answer from its local agent (within a timeout). If the local agent produces a response, that instance wraps it as a QUERY response and sends it back downstream.
3. If the local agent does not answer, the instance forwards the QUERY upstream via `query_to_master`. At the top-level master with no upstream, a `hive.query.timeout` error response is returned instead.

**Response envelope:**

The answer needs to find its way back down to the exact node that asked, so the response
carries a little bookkeeping. A QUERY response is itself a `HiveMessage(HiveMessageType.QUERY, ...)` with `metadata["is_response"] = True`. The payload wraps an inner `BUS` `HiveMessage`, which in turn carries the OVOS reply `Message`. The `metadata` fields are what the return trip routes on:

| Field | Value |
|---|---|
| `is_response` | `True` |
| `query_id` | UUID correlating request and response |
| `originator_peer` | Peer that issued the original request |
| `responder_peer` | Peer that produced the answer |

**Relay nodes** receiving a QUERY response with `is_response = True` route it toward `originator_peer` without re-processing it as a new request.

---

## Scatter/gather — CASCADE

`QUERY` wants *the* answer from one node. `CASCADE` wants *every* answer from everyone.
It floods the hive like `PROPAGATE`, but every reachable node that can respond does — and
the originator gathers the pile and picks a winner. Use it to ask the whole hive a
question at once: "who's currently playing music?", "what's each node's status?", "which
of you can handle this?"

**Request flow:**

1. A satellite sends `HiveMessage(HiveMessageType.CASCADE, bus_hive_message)` with a `query_id`.
2. Each instance that receives the CASCADE: (a) tries its local agent and, if it gets a response, sends that back as a CASCADE response; (b) forwards the CASCADE onward to all other connected peers and upstream.
3. Multiple responses can arrive at the originator from different nodes.

**Response collection:**

On the client side, `CascadeAggregator` buffers responses for a configurable `cascade_timeout` (default 5 s). Once the timeout expires — or the expected number of responses has arrived (derived from `HiveMapper.nodes`) — a `cascade_select_callback` picks the winning response and delivers it. The default select callback chooses randomly; applications should supply a domain-appropriate selector.

**Trust model:**

Here's the catch worth remembering. A `QUERY` answer comes from a known node up your own
chain; a `CASCADE` answer can come from *anywhere* in the hive, including nodes you have
no particular reason to trust. So the select callback isn't just picking the prettiest
reply — it's your one chance to vet who you're listening to before you act on it.

---

## PING and topology mapping

Before any of the routing above can work, a node has to know what the hive even *looks
like* — who's out there, and how many hops away. `PING` is how it finds out, and it
works by echo rather than reply.

A `PING` is always wrapped inside a `PROPAGATE`, so it floods to every reachable node.
There's no `PONG`: each node that hears a `PING` simply re-emits *its own* `PING` carrying
the same `flood_id`, which floods onward in turn. Receivers dedupe on `flood_id` so a
probe is only ever processed once, and the payload each one carries is small —
`{flood_id, peer, site_id, timestamp}`. `HiveMapper` in `hivemind_core.hive_map` sits and
watches those echoes come back, and from the pattern it draws a live map of the whole
hive.

---

## Session and context keys

Back down at the level of a single `BUS` message: how does hivemind-core keep two
satellites' conversations from bleeding into each other, and enforce that a guest device
can't invoke the skills you blacklisted for it? The answer is a bundle of metadata it
quietly staples onto every message before handing it to OVOS. You rarely set these
yourself, but knowing they're there explains a lot of "how did it know that?" moments:

| Key | Value | Purpose |
|---|---|---|
| `context["peer"]` | Satellite peer ID, e.g. `"HiveMindV0.0@127.0.0.1:8222/0"` | Identifies the originating satellite |
| `context["source"]` | Same as `peer` | OVOS origin identifier |
| `context["destination"]` | `"skills"` by default | Prevents message being treated as a broadcast |
| `context["session"]` | Serialized `Session` dict | Per-satellite session state |
| `context["session"]["session_id"]` | Stable UUID per satellite connection | Maintains conversational state across utterances |
| `context["session"]["blacklisted_skills"]` | List of skill IDs | Enforced by `IntentService` |
| `context["session"]["blacklisted_intents"]` | List of intent names | Enforced by `IntentService` |

And this is where the reply trick from the BUS section pays off: once OVOS calls
`Message.reply()`, `source` and `destination` swap, so `destination` now holds the
satellite's peer ID. `HiveMindListenerProtocol.handle_internal_mycroft()` reads that one
field and the answer knows its way home.

---

## Protocol versions

One last thing shapes every connection: not all clients are equally capable. A modern
browser can run the newest encryption; a five-dollar chip cannot. So HiveMind doesn't
force a single protocol on everyone — it negotiates, and the `ProtocolVersion` enum is
the ladder it climbs, one rung of capability at a time:

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

Two sides don't argue about this — they meet in the middle. The server admits a
connection only if it can negotiate at least `min_protocol_version` (config default **2**,
so the oldest JSON-only / no-binary v0/v1 clients are turned away by default), then
advertises the highest version both sides can manage (`THREE` whenever the Noise
primitive is available and the client has a password). The practical takeaway: a capable
client quietly gets the strong, always-encrypted v3 session, and an ancient one is
refused rather than silently downgraded. (One footnote to avoid confusion: this
`ProtocolVersion` enum is a different thing from the binary-serialization
`PROTOCOL_VERSION` constant in `serialization.py`.)

!!! note "You don't negotiate v2 to send binary"
    Whether a v1/v2 connection uses binary framing is gated by the `binarize` boolean exchanged in the handshake, **not** by negotiating `ProtocolVersion.TWO` on its own. Under v3 the session is always encrypted and binary-capable. See [Protocol Spec](../developers/protocol-spec.md) for the framing and Noise details.

---

## Source

Validated against the HiveMind source:

- [`hivemind_bus_client/message.py`](https://github.com/JarbasHiveMind/hivemind-websocket-client/blob/HEAD/hivemind_bus_client/message.py) — `HiveMessage` and the `HiveMessageType` enum
- [`hivemind_bus_client/serialization.py`](https://github.com/JarbasHiveMind/hivemind-websocket-client/blob/HEAD/hivemind_bus_client/serialization.py) — JSON/binary serialization and the `PROTOCOL_VERSION` constant
- [`hivemind_core/protocol.py`](https://github.com/JarbasHiveMind/HiveMind-core/blob/HEAD/hivemind_core/protocol.py) — listener/client roles, `binarize` gating, routing, and the `ProtocolVersion` enum
