# Protocol Specification

This page describes the wire format for HiveMind messages, including the binary framing introduced in protocol version 1.

## Message envelope

Every HiveMind message is a `HiveMessage` with:

- `msg_type` — a `HiveMessageType` enum value (see [Protocol Concepts](../concepts/protocol.md))
- `payload` — the message content (an OVOS `Message` object, a nested `HiveMessage`, or raw bytes)
- `context` — optional routing metadata dict

## Protocol versions

| Version | Transport encoding | Key exchange | Compression |
|---|---|---|---|
| **v0** | JSON | Pre-shared AES key only | None |
| **v1** | JSON or binary | PBKDF2 password handshake + PGP | Optional zlib |

## Binary framing (v1)

When both sides negotiate `binarize: true` in the handshake, messages are framed in a compact binary format instead of JSON.

### Header layout

```
<uint:1=start_marker> <uint:1=versioned_bit> [<uint:8=protocol_version>] <uint:5=msg_type> <uint:1=compression_bit> <uint:8=metadata_len>
```

| Field | Bits | Description |
|---|---|---|
| Start marker | 1 | Always `1`; used for alignment |
| Versioned flag | 1 | `1` if protocol version follows |
| Protocol version | 8 | Present only if versioned flag is `1` |
| Message type | 5 | `HiveMessageType` encoded as 5-bit uint (up to 32 types) |
| Compression flag | 1 | `1` if payload is zlib-compressed |
| Metadata length | 8 | Length of metadata block in bytes |

Followed by: metadata bytes, then payload bytes. Padding zeros are added to align to a byte boundary.

### Message type encoding

| Value | Type |
|---|---|
| 0 | HANDSHAKE |
| 1 | BUS |
| 2 | SHARED_BUS |
| 3 | BROADCAST |
| 4 | PROPAGATE |
| 5 | ESCALATE |
| 6 | INTERCOM |
| 7 | PING |
| 8 | PONG |
| 9 | HELLO |
| 10 | THIRDPRTY |
| 12 | BINARY |

### Binary payload type

For `BINARY` (`msg_type = 12`) messages, a 4-bit unsigned integer immediately after the metadata block indicates the binary content type:

| Value | Name | Description |
|---|---|---|
| 0 | UNDEFINED | Opaque binary content |
| 1 | RAW_AUDIO | Continuous microphone stream |
| 2 | NUMPY_IMAGE | Numpy array image (e.g., webcam frame) |
| 3 | FILE | File transfer; see context for filename |
| 4 | STT_AUDIO_TRANSCRIBE | Full audio utterance — return transcript only |
| 5 | STT_AUDIO_HANDLE | Full audio utterance — transcribe and handle intent |
| 6 | TTS_AUDIO | Synthesized speech audio (hub → satellite) |

### Example: BUS message (uncompressed, versioned)

```
1 | 1 | 00000001 | 00001 | 0 | 00000000 | <metadata> | <payload>
```

- `1` — start marker
- `1` — versioned flag
- `00000001` — protocol version 1
- `00001` — BUS (type 1)
- `0` — not compressed
- `00000000` — no metadata
- `<payload>` — UTF-8 JSON string

### Example: BINARY message (raw audio)

```
1 | 1 | 00000001 | 01100 | 0 | 00000000 | <metadata> | 0001 | <audio_bytes>
```

- `01100` — BINARY (type 12)
- `0001` — RAW_AUDIO binary payload type
- `<audio_bytes>` — PCM audio data

## Compression

When the compression flag is set, both the metadata and payload are compressed independently with zlib. Compression is most effective on large payloads (reduces size by up to ~50%); it adds overhead for small messages.

## Session context

See [Protocol Concepts — Session and context keys](../concepts/protocol.md#session-and-context-keys) for the full reference of keys injected into `Message.context` by the hub.

## OVOS messages (payload format)

OVOS `Message` objects are the standard payload for `BUS` messages. The structure:

```json
{
  "type": "recognizer_loop:utterance",
  "data": {
    "utterances": ["what time is it?"],
    "lang": "en-us"
  },
  "context": {
    "session": {...},
    "source": "hive",
    "destination": "skills"
  }
}
```

The full OVOS message specification is maintained at [OpenVoiceOS/message_spec](https://github.com/OpenVoiceOS/message_spec).

## Transports

The protocol runs over any transport that can carry byte streams:

| Transport | Plugin | Default port |
|---|---|---|
| WebSocket | `hivemind-websocket-plugin` | 5678 |
| HTTP (polling) | `hivemind-http-plugin` | 5679 |
| MQTT | `hivemind-mqtt-protocol` | — |
| Usenet (NNTP) | `hivemind-usenet` | — |
