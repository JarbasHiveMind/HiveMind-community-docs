# Protocol Specification

This is the page you reach for when you're writing a HiveMind client from scratch — in
Rust, in Go, on a microcontroller, in whatever the Python and JavaScript libraries don't
already cover — and you need to get every byte exactly right. It is the ground truth: the
shape of the envelope, the step-by-step handshake, how the session key is derived, and
the compact binary framing that kicks in when both sides agree to it. Nothing is
hand-waved. Follow it literally and an independent client will reach a fully encrypted,
session-established connection that hivemind-core accepts.

!!! abstract "In a nutshell"
    - Every message is a `HiveMessage` (`msg_type`, `payload`, `context`); `BUS` messages carry OVOS `Message` objects.
    - The session `ProtocolVersion` is negotiated at connect time: v1/v2 use a password or RSA handshake, v3 uses a Noise handshake and is always encrypted.
    - The [handshake state machine](#handshake-state-machine) and [negotiation defaults](#negotiation-defaults) are sufficient to bring an independent client to a fully-encrypted, session-established state.
    - `binarize` is a per-connection capability, not a version bump: it packs messages into a compact binary frame instead of JSON.

!!! note "Using the Python or JS client?"
    The library already implements everything on this page. Read it only to implement a HiveMind client from scratch in another language, or to debug the wire format; otherwise start with the [Client Library](client-library.md).

---

## Message envelope

Start with the shape of a single message, because everything else is a variation on it.
No matter its type, a `HiveMessage` is only ever three fields:

- `msg_type` — a `HiveMessageType` enum value (see [Protocol Concepts](../concepts/protocol.md))
- `payload` — the message content (an OVOS `Message` object, a nested `HiveMessage`, or raw bytes)
- `context` — optional routing metadata dict

Get those three right and you can represent any message on the wire. The rest of this
page is about the two hard parts: agreeing on encryption before you send them, and packing
them tightly once you do.

---

## Protocol version

Every connection speaks a single protocol version: **v3**, the Noise handshake. The
`ProtocolVersion` enum (`ZERO`/`ONE`/`TWO`/`THREE`, in `hivemind_core/protocol.py`) still
enumerates the historical rungs on the wire, but only `THREE` is ever accepted — there is no
negotiable floor, and no v0/v1/v2 fallback path. A connection that cannot complete the v3
Noise handshake (no Noise primitive available, or no password presented) is refused: the
server closes it with WebSocket code `1008` and the reason `this node requires protocol v3
(the Noise handshake)`.

The legacy serialization-layer `PROTOCOL_VERSION` constant in `serialization.py` is `1`; that
binary-framing version is a distinct thing from this session `ProtocolVersion` and is not
bumped for v3.

| Version | Transport encoding | Key exchange |
|---|---|---|
| **v3** (the only accepted version) | Binary framing, always encrypted | **Noise** (`Noise_XXpsk2_25519_ChaChaPoly_SHA256` default — negotiated by browser/JS clients too via `@noble`; `AESGCM` suite as a fallback for minimal Web-Crypto-only peers; `KKpsk0` once the client's static key is pinned). PSK = `argon2id(password, SHA-256(node_id))` |

---

## Handshake state machine

This is the heart of the page — the exact choreography that takes a fresh socket to an
encrypted session. Field names below are lifted straight from the reference implementation
(`hivemind_core/protocol.py` server side; `hivemind_bus_client` client side).

### Framing rules that hold for the whole handshake

- **`HELLO` and `HANDSHAKE` messages travel in plaintext JSON only until the Noise session
  is established.** Once the handshake completes and `noise_transport` is set, every later
  message — including the client's second `HELLO` — is Noise-encrypted like everything
  else. Any cleartext frame outside that pre-handshake window is rejected and the connection
  closed with code `1008`.
- The transport-layer JSON for any `HiveMessage` is the object returned by `HiveMessage.as_dict`: `{"msg_type", "payload", "metadata", "route", "node", "target_site_id", "target_pubkey", "source_peer"}`. For `HELLO`/`HANDSHAKE` only `msg_type` and `payload` matter.
- `msg_type` values are the **string** enum values, not the binary integers: `HELLO = "hello"`, `HANDSHAKE = "shake"`. Two of the strings are not the lowercased enum name (`HANDSHAKE = "shake"`, `BINARY = "bin"`), so copy those two from the [Quick reference](../reference/message-types.md#quick-reference) table instead of deriving them.
- The Noise handshake itself rides inside `HANDSHAKE` messages carrying a `noise` object: `{"pattern": ..., "suite": ..., "msg": "<hex>"}`. The client is always the Noise initiator; the server is the responder.

### Connection setup sequence

Authentication to the WebSocket itself happens first, before any HiveMessage — that is outside this state machine. The client puts the access key in the connect URL as an `authorization` query parameter, base64 of `useragent:access_key`, so the URL reads `ws://host:port?authorization=<b64>`. The client sends its `site_id` later, in the second `HELLO`. Once the socket is open:

1. **Server → Client `HELLO`** (plaintext). Announces the server identity, bound into the Noise handshake prologue.
2. **Server → Client `HANDSHAKE`** (plaintext, **no `noise.msg`**). Advertises `max_protocol_version` (always `THREE`), `binarize`, allowed `encodings`/`ciphers`, and the Noise `patterns`/`suites` on offer.
3. **Client → Server `HANDSHAKE`** (plaintext, **Noise message 1**). Names the selected pattern/suite and starts the Noise handshake, embedding the client's `binarize`/`encodings` choice inside the encrypted Noise payload.
4. **Server → Client `HANDSHAKE`** (plaintext, **Noise message 2**). Continues the handshake, embedding the server's selected `encoding`. For `KKpsk0` (both static keys already known) the handshake is complete here.
5. **Client → Server `HANDSHAKE`** (plaintext, **Noise message 3**, `XXpsk2` only). Authenticates the client's static key and completes the handshake.
6. **Client → Server `HELLO`** (Noise-encrypted, sent *after* the transport is established). Carries the client's session, site_id, and public key.
7. From here on, all other message types are Noise-encrypted.

```
Server                                  Client
  |  HELLO {pubkey, peer, node_id}        |   (1) plaintext
  | ------------------------------------> |
  |  HANDSHAKE {max_protocol_version,     |   (2) plaintext, advertises capabilities
  |   binarize, encodings, ciphers,       |
  |   noise: {patterns, suites}}          |
  | ------------------------------------> |
  |  HANDSHAKE {noise: {pattern, suite,   |   (3) plaintext, Noise message 1
  |   msg: <hex>}}                        |
  | <------------------------------------ |
  |  HANDSHAKE {noise: {msg: <hex>}}      |   (4) plaintext, Noise message 2
  | ------------------------------------> |
  |  HANDSHAKE {noise: {msg: <hex>}}      |   (5) plaintext, Noise message 3 (XXpsk2 only)
  | <------------------------------------ |
  |  HELLO {pubkey, session, site_id}     |   (6) Noise-encrypted
  | <------------------------------------ |
  |  <encrypted BUS / QUERY / ... >       |   (7) Noise-encrypted
  |<====================================>|
```

The wire-level Noise handshake tokens, the prologue binding of the `HELLO`/`HANDSHAKE`
payloads, and the PSK derivation (including the provisioned-PSK path for constrained
devices) live in `hivemind_bus_client/noise.py` and `poorman_handshake/noise/`; the
[Security](../concepts/security.md) page covers the model at a higher level.

#### Server → Client `HELLO` — `payload` fields

| Field | Type | Meaning |
|---|---|---|
| `pubkey` | str (PEM) | Server's public key, bound into the Noise prologue. |
| `peer` | str | Identifies this client in OVOS `message.context` (server's view of the connection). |
| `node_id` | str | The server's peer id; becomes the client's `node_id` (how the local bus refers to the master). |

#### Server → Client `HANDSHAKE` (capability advertisement) — `payload` fields

| Field | Type | Meaning |
|---|---|---|
| `max_protocol_version` | int | Always `THREE`. What a v3 client reads to select the Noise handshake. |
| `noise` | object | `{"patterns": [...], "suites": [...]}` in preference order (`XXpsk2`/`KKpsk0`, `25519_ChaChaPoly_SHA256`/`25519_AESGCM_SHA256`). `KKpsk0` is offered only once this client's static key has been pinned by a prior `XXpsk2` handshake. |
| `binarize` | bool | Server supports the binary framing scheme. From `cfg["binarize"]`, default `False`. |
| `encodings` | list[str] | Server-allowed `SupportedEncodings`, server preference order. Defaults to **all** encodings. |
| `ciphers` | list[str] | Server-allowed `SupportedCiphers`, server preference order. The shipped default is `["CHACHA20-POLY1305", "AES-GCM"]`. The server falls back to `["AES-GCM"]` only when `allowed_ciphers` is empty. |

#### Client → Server `HELLO` — `payload` fields (sent Noise-encrypted, after the transport is established)

| Field | Type | Meaning |
|---|---|---|
| `pubkey` | str (PEM) | The client's own public key. |
| `session` | str (JSON) | Serialized OVOS `Session` for `session_id`. The server deserializes this as the client's session. |
| `site_id` | str | The client's site id (used for `BROADCAST`/`PROPAGATE` `target_site_id` filtering). |

> Note: a non-admin client's declared `session_id == "default"` is silently reassigned to a
> fresh per-connection session id at connect time, not disconnected. If a non-admin BUS
> message still somehow declares `"default"`, `DefaultSessionPolicy` denies that message
> (`session_id_default_forbidden`, notified via `hive.policy.denied`) rather than closing
> the connection. Admins keep the literal `"default"` session id.

---

## Negotiation & defaults

The handshake left a few things "negotiated" — the encoding, the cipher, the key math.
This section pins down exactly what those resolve to, including the two defaults that bite
newcomers most often. Start with the sneakiest one.

### Default encoding is `JSON_HEX`, not `JSON_B64`

Several helper functions (`encrypt_as_json`, `decrypt_from_json`) carry a Python default argument of `JSON_B64`, but those defaults are **never the protocol default**. The protocol default everywhere it matters is **`JSON_HEX`**:

- The client's handshake handler falls back to `SupportedEncodings.JSON_HEX` when the server response omits `encoding`.
- The server's handshake handler falls back to `[SupportedEncodings.JSON_HEX]` when the client omits `encodings`.

So an independent client that omits `encodings` from its Noise-handshake payload must encode encrypted JSON using **hex** (`JSON-HEX`), and the cipher default is **`AES-GCM`**.

### Encrypted-JSON envelope shape

After the handshake, encrypted messages are JSON objects (then text-encoded per the negotiated encoding) of the form produced by `encrypt_as_json`:

```json
{ "ciphertext": "<encoded>", "tag": "<encoded>", "nonce": "<encoded>" }
```

- For `AES-GCM`: `nonce` is 16 bytes, `tag` is 16 bytes, key is 16/24/32 bytes (poorman secrets are 32 → AES-256).
- For `CHACHA20-POLY1305`: `nonce` is 12 bytes (RFC 7539), `tag` is 16 bytes, key is 32 bytes.
- Each of `ciphertext`/`tag`/`nonce` is independently text-encoded with the negotiated `SupportedEncodings` codec (default hexlify/unhexlify for `JSON-HEX`).
- Web-Crypto compatibility: if `tag` is absent, the last 16 bytes of `ciphertext` are treated as the tag.

### `SupportedEncodings` (from `encryption.py`)

The string value on the wire is the right column.

| Enum | Wire value | Codec |
|---|---|---|
| `JSON_B91` | `JSON-B91` | Base91 |
| `JSON_Z85B` | `JSON-Z85B` | Z85B |
| `JSON_Z85P` | `JSON-Z85P` | Z85P |
| `JSON_B64` | `JSON-B64` | Base64 |
| `JSON_URLSAFE_B64` | `JSON-URLSAFE-B64` | URL-safe Base64 |
| `JSON_B32` | `JSON-B32` | Base32 |
| `JSON_HEX` | `JSON-HEX` | Base16/hex (**protocol default**) |

### `SupportedCiphers` (from `encryption.py`)

| Enum | Wire value | Notes |
|---|---|---|
| `AES_GCM` | `AES-GCM` | Default; 16/24/32-byte key, 16-byte nonce, 16-byte tag |
| `CHACHA20_POLY1305` | `CHACHA20-POLY1305` | RFC 7539; 32-byte key, 12-byte nonce, 16-byte tag |

`optimal_ciphers()` orders these by CPU AES-NI support: `[AES-GCM, CHACHA20-POLY1305]` when AES-NI is present, `[CHACHA20-POLY1305, AES-GCM]` otherwise.

### Key derivation (delegated to `poorman_handshake`)

The session key math lives in the external **`poorman_handshake`** package, not in `hivemind-bus-client`. A non-Python client must reimplement it.

The Noise transport key comes from the handshake itself (`XXpsk2`/`KKpsk0` over X25519). The
pre-shared key mixed into that handshake is `PSK = argon2id(password, salt = SHA-256(node_id))`
— salting with the server's `node_id` makes the PSK server-specific. A pre-provisioned
32-byte `psk` remains an option for constrained devices that cannot run argon2id on-device
(`hivemind-core derive-psk`); PBKDF2 remains an explicit fallback when a server advertises
it. See [Security → Handshake and encryption](../concepts/security.md#handshake-and-encryption)
for the full model, including forward secrecy and static-key pinning.

---

## Binary framing

Everything so far assumed JSON on the wire — readable, but chatty. When both sides agree
to `binarize`, the same messages get packed into a tight bitstream instead, which is how
raw audio rides the protocol without drowning it. (Remember: this is a per-connection flag,
not a version bump — the wire `ProtocolVersion` stays `ONE`.) The format is a small header
followed by the payload, and the header is bit-packed, so read the widths carefully:

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

Followed by: metadata bytes, then payload bytes. To pad to a byte boundary, zero bits are **prepended** to the left of the start marker; the decoder skips these leading zeros until it reads the first `1`. The metadata length is a `uint:8` (max 255 bytes).

The metadata block must always hold a valid JSON object. The encoder writes `{}` when there is no metadata, and the decoder always parses the block, so the minimum uncompressed length is 2 bytes. A frame with a metadata length of 0 makes the reference decoder raise a JSON error.

### Message type encoding

| Value | Type |
|---|---|
| 0 | HANDSHAKE |
| 1 | BUS |
| 2 | SHARED_BUS |
| 3 | BROADCAST |
| 4 | PROPAGATE |
| 5 | ESCALATE |
| 6 | HELLO |
| 7 | QUERY |
| 8 | CASCADE |
| 9 | PING |
| 10 | RENDEZVOUS |
| 12 | BINARY |

The type field is a 5-bit unsigned integer. Codes `0`-`10` and `12` are assigned. Code `11` was `THIRDPRTY`, a type that has been removed; the code stays reserved and must not be given to another type. Code `11` and codes `13`-`31` are unassigned: a sender must not emit them, and a receiver rejects such a frame as malformed instead of mapping it to a type.

`INTERCOM` has no code of its own, so it cannot be binary-framed. The encoder raises rather than relabelling it. Send `INTERCOM` as a text frame.

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
| 6 | TTS_AUDIO | Synthesized speech audio (hivemind-core → satellite) |

### Versioned vs unversioned framing

The `versioned` bit is **0** by default in the reference encoder (`get_bitstring(..., versioned=False)`). When `0`, the 8-bit protocol-version field is **omitted** and the decoder assumes `PROTOCOL_VERSION` (= 1). Only set the versioned bit to `1` if you also emit the `uint:8` version byte. The two examples below show the versioned form for clarity.

### Example: BUS message (uncompressed, versioned)

```
1 | 1 | 00000001 | 00001 | 0 | 00000010 | <metadata> | <payload>
```

- `1` — start marker
- `1` — versioned flag
- `00000001` — protocol version 1
- `00001` — BUS (type 1)
- `0` — not compressed
- `00000010` — metadata length 2
- `<metadata>` — `{}`, the empty JSON object
- `<payload>` — UTF-8 JSON string

### Example: BINARY message (raw audio)

```
1 | 1 | 00000001 | 01100 | 0 | 00000010 | <metadata> | 0001 | <audio_bytes>
```

- `01100` — BINARY (type 12)
- `00000010` — metadata length 2, followed by `{}`
- `0001` — RAW_AUDIO binary payload type
- `<audio_bytes>` — PCM audio data

---

## Compression

When the compression flag is set, zlib compresses the metadata block, and also the payload of every type except `BINARY` (each typically ~49–50% smaller). Compression is most effective on large payloads; it adds overhead for small messages.

`BINARY` payload bytes are an exception. The encoder appends them raw and the decoder returns them raw, whatever the compression flag says. If you zlib-compress raw audio yourself and set the flag, the receiver passes the compressed blob to the audio plugin as PCM. The satellite then plays noise.

---

## Session context

See [Protocol Concepts — Session and context keys](../concepts/protocol.md#session-and-context-keys) for the full reference of keys injected into `Message.context` by hivemind-core.

---

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

---

## Transports

The protocol runs over any transport that can carry byte streams:

| Transport | Plugin | Default port |
|---|---|---|
| WebSocket | `hivemind-websocket-plugin` | 5678 |
| HTTP (polling) | `hivemind-http-plugin` | 5679 |
| MQTT (broker) | `hivemind-mqtt-plugin` | 1883 |
| Usenet wormhole | `hivemind-usenet-wormhole` | — |

WebSocket and HTTP are the stable defaults. MQTT (package `hivemind-mqtt-protocol`)
is a published alpha providing a complete hivemind-core transport without a satellite client.
The Usenet wormhole (package `hivemind-usenet`) is experimental — a
high-latency covert/fallback control-plane, not a real-time transport.

---

## Source

Validated against the HiveMind source:

- [`hivemind_bus_client/serialization.py`](https://github.com/JarbasHiveMind/hivemind-websocket-client/blob/HEAD/hivemind_bus_client/serialization.py) — `PROTOCOL_VERSION`, the binary framing encoder/decoder, the `_INT2TYPE` message-type table
- [`hivemind_bus_client/message.py`](https://github.com/JarbasHiveMind/hivemind-websocket-client/blob/HEAD/hivemind_bus_client/message.py) — `HiveMessageType`, `HiveMessage.as_dict`
- [`hivemind_core/protocol.py`](https://github.com/JarbasHiveMind/HiveMind-core/blob/HEAD/hivemind_core/protocol.py) — `ProtocolVersion`, `max_protocol_version`, the server-side handshake state machine and capability defaults
