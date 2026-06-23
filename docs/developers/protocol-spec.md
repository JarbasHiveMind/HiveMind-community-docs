# Protocol Specification

> **Who needs this page?** Only if you are implementing a HiveMind client from scratch in a language without an official library, or debugging the wire format. If you are using the Python or JS client, the library already does everything described here — start with the [Client Library](client-library.md) instead.

This page describes the wire format for HiveMind messages, including the binary framing that is enabled by the `binarize` capability negotiated during the handshake.

If you are implementing an independent (non-Python) client, read the [Handshake state machine](#handshake-state-machine) and [Negotiation & defaults](#negotiation-defaults) sections below first — they are written to be sufficient to bring a client to a fully-encrypted, session-established state without reading the reference implementation.

## Message envelope

Every HiveMind message is a `HiveMessage` with:

- `msg_type` — a `HiveMessageType` enum value (see [Protocol Concepts](../concepts/protocol.md))
- `payload` — the message content (an OVOS `Message` object, a nested `HiveMessage`, or raw bytes)
- `context` — optional routing metadata dict

## Protocol versions

The hivemind-core `ProtocolVersion` enum (`ZERO`/`ONE`/`TWO`, in `hivemind_core/protocol.py`) is declared with three members, but on the wire it tops out at **ONE** in practice:

- The server advertises `max_protocol_version = ProtocolVersion.ONE` (hardcoded). `min_protocol_version` is `ONE` when the client has no pre-shared crypto key and the server requires crypto, otherwise `ZERO`.
- The serialization-layer `PROTOCOL_VERSION` constant in `serialization.py` is also `1`. The binary framing decoder/encoder only accept `proto_version <= 1` and raise `UnsupportedProtocolVersion` otherwise.

Because of this, **`ProtocolVersion.TWO` is effectively unreachable**. Binary framing is **not** gated behind a v2 version bump — it is enabled purely by the `binarize` capability boolean negotiated during the handshake (see [Negotiation & defaults](#negotiation-defaults)). A connection can therefore be `ProtocolVersion.ONE` and still exchange binary-framed messages.

| Version | Transport encoding | Key exchange | Compression |
|---|---|---|---|
| **v0** | JSON | Pre-shared AES key only (legacy, deprecated) | None |
| **v1** | JSON (and binary framing when `binarize` is negotiated) | Handshake: PBKDF2 password **or** RSA | Optional zlib |

## Handshake state machine

This section is the authoritative connection-setup sequence for an independent client. Field names are taken verbatim from the reference implementation (`hivemind_core/protocol.py` server side; `hivemind_bus_client/protocol.py` `HiveMindSlaveProtocol` client side). Implement it exactly.

### Framing rules that hold for the whole handshake

- **`HELLO` and `HANDSHAKE` messages always travel in PLAINTEXT JSON**, even after the session key has been established. This includes the client's *second* `HELLO` (the one carrying session data), which is emitted **after** `crypto_key` is set but is still sent unencrypted. Only the post-handshake `BUS`/`QUERY`/etc. traffic is encrypted. A client must therefore be able to send/receive `HELLO` and `HANDSHAKE` as plain `HiveMessage` JSON regardless of crypto state.
- The transport-layer JSON for any `HiveMessage` is the object returned by `HiveMessage.as_dict`: `{"msg_type", "payload", "metadata", "route", "node", "target_site_id", "target_pubkey", "source_peer"}`. For `HELLO`/`HANDSHAKE` only `msg_type` and `payload` matter.
- `msg_type` values are the **string** enum values, not the binary integers: `HELLO = "hello"`, `HANDSHAKE = "shake"`.
- **A `HANDSHAKE` is a REQUEST vs a RESPONSE distinguished ONLY by the presence of the `envelope` field.** There is no separate type or flag. The client decides which branch of its `HANDSHAKE` handler to run purely by `if "envelope" in payload`. A server→client `HANDSHAKE` *without* `envelope` is the "please start the handshake" request advertising capabilities; a message *with* `envelope` is the response that completes the exchange.

### Connection setup sequence

Authentication to the WebSocket itself happens first via HTTP headers (`authorization: <access_key>`, `x-hm-site-id`), before any HiveMessage — that is outside this state machine. Once the socket is open:

1. **Server → Client `HELLO`** (plaintext). Announces the server identity.
2. **Server → Client `HANDSHAKE`** (plaintext, **no `envelope`**). Advertises server capabilities and asks the client to start the handshake.
3. **Client → Server `HANDSHAKE`** (plaintext, **with `envelope`**). Carries the client's handshake material plus its cipher/encoding/binarize selections.
4. **Server → Client `HANDSHAKE`** (plaintext, **with `envelope`**). Carries the server's handshake material and the final selected `encoding`/`cipher`. After this, **both sides can derive the same `crypto_key`.**
5. **Client → Server `HELLO`** (plaintext, but sent *after* crypto is established). Carries the client's session, site_id, and public key.
6. From here on, all other message types are encrypted with the negotiated `crypto_key`/`cipher`/`encoding`.

```
Server                                  Client
  |  HELLO {pubkey, peer, node_id}        |   (1) plaintext
  | ------------------------------------> |
  |  HANDSHAKE {handshake, min/max_proto, |   (2) plaintext, NO envelope = REQUEST
  |   binarize, preshared_key, password,  |
  |   crypto_required, encodings, ciphers}|
  | ------------------------------------> |
  |                                       |   client picks branch on `password`
  |  HANDSHAKE {envelope, encodings,      |   (3) plaintext, HAS envelope
  |   ciphers, binarize [, pubkey]}       |
  | <------------------------------------ |
  |  HANDSHAKE {envelope, encoding,       |   (4) plaintext, HAS envelope = RESPONSE
  |   cipher}                             |       both sides now hold crypto_key
  | ------------------------------------> |
  |  HELLO {pubkey, session, site_id}     |   (5) plaintext, sent AFTER crypto set
  | <------------------------------------ |
  |  <encrypted BUS / QUERY / ... >       |   (6) encrypted
  |<====================================>|
```

#### (1) Server → Client `HELLO` — `payload` fields

| Field | Type | Meaning |
|---|---|---|
| `pubkey` | str (PEM) | Server's RSA public key. The client stores this as `mpubkey` and uses it to authenticate the server in RSA mode. Only honored on the first HELLO (before `node_id` is set). |
| `peer` | str | Identifies this client in OVOS `message.context` (server's view of the connection). |
| `node_id` | str | The server's peer id; becomes the client's `node_id` (how the local bus refers to the master). |

#### (2) Server → Client `HANDSHAKE` (request) — `payload` capability fields

| Field | Type | Meaning |
|---|---|---|
| `handshake` | bool | `True` ⇒ client MUST complete a handshake or the connection is dropped. (`needs_handshake = not client.crypto_key and self.handshake_enabled`.) |
| `min_protocol_version` | int | Minimum acceptable `ProtocolVersion` (`1` if crypto required and no preshared key, else `0`). |
| `max_protocol_version` | int | Maximum acceptable `ProtocolVersion`. **Hardcoded to `1`.** |
| `binarize` | bool | Server supports the binary framing scheme. From `cfg["binarize"]`, default `False`. |
| `preshared_key` | bool | Server already holds a pre-shared crypto key for this client (legacy V0 path). |
| `password` | bool | Server has a password configured for this client ⇒ password handshake is available (V1). If `True` and the client also has a password, the client takes the **password branch**. |
| `crypto_required` | bool | Server rejects unencrypted payloads. |
| `encodings` | list[str] | Server-allowed `SupportedEncodings`, server preference order. Defaults to **all** encodings. |
| `ciphers` | list[str] | Server-allowed `SupportedCiphers`, server preference order. Defaults to `["AES-GCM"]`. |

#### (3) Client → Server `HANDSHAKE` (with `envelope`) — `payload` fields

The client always sends:

| Field | Type | Meaning |
|---|---|---|
| `binarize` | bool | Client's choice (it echoes the server's advertised value in the reference client). |
| `encodings` | list[str] | Client's **preference-ordered** acceptable encodings (reference client sends `list(SupportedEncodings)`). |
| `ciphers` | list[str] | Client's **preference-ordered** acceptable ciphers (reference client sends `optimal_ciphers()`, i.e. AES first if the CPU has AES-NI, else ChaCha20 first). |

Plus exactly one of:

| Field | Type | When |
|---|---|---|
| `envelope` | str (hex) | **Password branch.** Present when the client uses a `PasswordHandShake`. This is the field that marks the message as carrying handshake material. |
| `pubkey` | str (PEM) | **RSA branch.** Present when no password is used; the client sends its RSA public key instead and there is **no** `envelope` in *this* client message. |

> Selection ownership: the client *proposes* `encodings`/`ciphers` in preference order. The **server** intersects them with its own allowed sets and then selects the client's element `[0]` of each filtered list (`client.cipher = ciphers[0]`, `client.encoding = encodings[0]`). If the intersection is empty for either, the server disconnects the client. **This negotiation runs ONLY on the password branch.** On the RSA-pubkey branch the server never reads the client's `encodings`/`ciphers`, so the encoding/cipher silently stay at the server defaults (see [Negotiation & defaults](#negotiation-defaults)).

#### (4) Server → Client `HANDSHAKE` (response, with `envelope`) — `payload` fields

| Field | Type | Meaning |
|---|---|---|
| `envelope` | str (hex) | Server handshake material. In password mode this is `PasswordHandShake.generate_handshake()` (an hSub). In RSA mode it is `HandShake.generate_handshake(client_pubkey)` = `hexlify(signature + ciphertext)`. |
| `encoding` | str | **Final** selected encoding (the value the client must use for all encrypted JSON from now on). The client reads `payload.get("encoding") or JSON_HEX`. |
| `cipher` | str | **Final** selected cipher. The client reads `payload.get("cipher") or AES_GCM`. |

On receipt the client derives `crypto_key`:

- **Password mode:** `pswd_handshake.receive_and_verify(envelope)` validates the server proved the same password, then `crypto_key = pswd_handshake.secret` (see PBKDF2 math below).
- **RSA mode:** if the client knows the server pubkey (`mpubkey`, from HELLO) it calls `handshake.receive_and_verify(envelope, mpubkey)` (verifies the PSS signature over the ciphertext, then decrypts); otherwise `handshake.receive_handshake(envelope)` (trust-on-first-use). Then `crypto_key = handshake.secret`.

#### (5) Client → Server `HELLO` — `payload` fields (sent plaintext, after crypto_key is set)

| Field | Type | Meaning |
|---|---|---|
| `pubkey` | str (PEM) | The client's own RSA public key (`identity.public_key`). |
| `session` | str (JSON) | Serialized OVOS `Session` for `session_id`. The server deserializes this as the client's session. |
| `site_id` | str | The client's site id (used for `BROADCAST`/`PROPAGATE` `target_site_id` filtering). |

> Note: a client requesting `session_id == "default"` is disconnected unless it is an administrator.

## Negotiation & defaults

### Default encoding is `JSON_HEX`, not `JSON_B64`

Several helper functions (`encrypt_as_json`, `decrypt_from_json`) carry a Python default argument of `JSON_B64`, but those defaults are **never the protocol default**. The protocol default everywhere it matters is **`JSON_HEX`**:

- The client's handshake handler falls back to `SupportedEncodings.JSON_HEX` when the server response omits `encoding`.
- The server's handshake handler falls back to `[SupportedEncodings.JSON_HEX]` when the client omits `encodings`.

So an independent client that never negotiates an encoding (e.g. RSA branch) must encode encrypted JSON using **hex** (`JSON-HEX`), and the cipher default is **`AES-GCM`**.

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

**Password mode (`PasswordHandShake`):**

- **Handshake envelope = an hSub** (hashed subject), NOT PBKDF2. `generate_handshake()` produces `hsub = iv + SHA256(iv + password)`, hex-encoded and **truncated to 48 hex chars** (`create_hsub(..., hsublen=48)`). The `iv` is 8 random bytes (16 hex chars).
- **Verification:** the receiver re-derives the hSub using the `iv` parsed from the first 16 hex chars of the peer's hSub and checks for collision (`match_hsub`) — this proves both ends share the password without transmitting it.
- **Shared salt:** `salt = iv_client XOR iv_server` (byte-wise XOR of the two 8-byte IVs; `receive_handshake` does `bytes(a ^ b for a, b in zip(self.iv, iv_from_hsub(peer_shake)))`).
- **Session key:** `secret = PBKDF2-HMAC-SHA256(password, salt, iterations=100000)` → 32 bytes. This is the AES/ChaCha key (`crypto_key`).
- Note the HSUB uses a **plain salted SHA-256**, while the session key uses **PBKDF2 (100000 iters)** — they are different primitives; do not conflate them.

**RSA mode (`HandShake`):**

- The party generating the handshake (the **server**, which calls `generate_handshake(peer_pubkey)`) picks a random 32-byte `secret`, RSA-encrypts it for the peer with **PKCS#1 OAEP**, and prepends a **PSS-over-SHA-256** signature of the ciphertext. The wire envelope is `hexlify(signature + ciphertext)`; the signature length equals the signer's RSA key size in bytes.
- The receiver strips the signature (first `key_size_in_bytes` bytes), RSA-decrypts the remainder with its private key to recover the 32-byte `secret`, and (in `receive_and_verify`) first verifies the PSS signature against the known peer public key.
- That 32-byte `secret` becomes `crypto_key`. No PBKDF2 is involved in RSA mode.

## Binary framing

When both sides negotiate `binarize: true` in the handshake, messages are framed in a compact binary format instead of JSON. This is a per-connection capability flag, not a protocol-version increment — the wire `ProtocolVersion` remains `ONE`.

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
| 11 | THIRDPRTY |
| 12 | BINARY |

`INTERCOM` has no dedicated binary integer slot — it defaults to `11` when encoded. The type field is a 5-bit unsigned integer.

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

### Versioned vs unversioned framing

The `versioned` bit is **0** by default in the reference encoder (`get_bitstring(..., versioned=False)`). When `0`, the 8-bit protocol-version field is **omitted** and the decoder assumes `PROTOCOL_VERSION` (= 1). Only set the versioned bit to `1` if you also emit the `uint:8` version byte. The two examples below show the versioned form for clarity.

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

When the compression flag is set, both the metadata and payload are compressed independently with zlib (each typically ~49–50% smaller). Compression is most effective on large payloads; it adds overhead for small messages.

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
| MQTT (broker) | `hivemind-mqtt-plugin` | 1883 |
| Usenet wormhole | `hivemind-usenet-wormhole` | — |

WebSocket/HTTP are stable defaults. MQTT (package `hivemind-mqtt-protocol`) is a
published alpha with a complete hub listener; its satellite client is still
planned. The Usenet wormhole (package `hivemind-usenet`) is experimental and
unpublished — a high-latency covert/fallback control-plane, not a real-time
transport.

## Source

Validated against the HiveMind source:

- [`hivemind_bus_client/serialization.py`](https://github.com/JarbasHiveMind/hivemind-websocket-client/blob/HEAD/hivemind_bus_client/serialization.py) — `PROTOCOL_VERSION`, the binary framing encoder/decoder, the `_INT2TYPE` message-type table
- [`hivemind_bus_client/message.py`](https://github.com/JarbasHiveMind/hivemind-websocket-client/blob/HEAD/hivemind_bus_client/message.py) — `HiveMessageType`, `HiveMessage.as_dict`, the `INTERCOM`-defaults-to-`11` encoding
- [`hivemind_core/protocol.py`](https://github.com/JarbasHiveMind/HiveMind-core/blob/HEAD/hivemind_core/protocol.py) — `ProtocolVersion`, `max_protocol_version`, the server-side handshake state machine and capability defaults
