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
| `INTERCOM` | Transport | Point-to-point | End-to-end encrypted tunnel |
| `PING` | Discovery | Bidirectional (via PROPAGATE) | Topology probe |
| `PONG` | Discovery | Satellite → Hub | Topology probe reply |
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

## INTERCOM

End-to-end encrypted point-to-point message.

- **Payload**: `{"ciphertext": "<PGP-encrypted-data>"}` — only the target node can decrypt
- **Signature**: sender's private PGP key signs the ciphertext
- **Routing**: typically wrapped in ESCALATE or PROPAGATE to reach the target
- **Intermediate nodes**: cannot read the content or determine the recipient

## PING / PONG

Topology discovery.

- `PING` is always wrapped inside `PROPAGATE`
- Every node that receives PING replies with `PONG` carrying the path it travelled
- `HiveMapper` in `hivemind_core.hive_map` collects PONGs to build a live topology map

## HELLO / HANDSHAKE

Connection management. Handled automatically by `HiveMessageBusClient` and `hivemind-core`.

- `HELLO` announces the node (node_id, public key) — sent unencrypted
- `HANDSHAKE` performs PBKDF2 key exchange — sent unencrypted, establishes the session key
- After the handshake, a second `HELLO` (encrypted) carries session data, site_id, and client public key

## THIRDPRTY

User-defined payload. Relayed opaquely by the hub without any special processing. Use it for application-level protocols layered on top of HiveMind.
