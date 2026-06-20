# Developer Guide — Client Library

`hivemind-bus-client` (repo: `hivemind-websocket-client`) is the primary Python library for building HiveMind clients. It handles connection management, the PBKDF2 handshake, AES-256-GCM encryption, and serialization transparently.

## Install

```bash
pip install hivemind-bus-client
```

## Libraries

| Library | Language | Notes |
|---|---|---|
| [hivemind-bus-client](https://github.com/JarbasHiveMind/hivemind_websocket_client) | Python | Primary client; WebSocket + HTTP + async + MQTT |
| [HiveMind-js](https://github.com/JarbasHiveMind/HiveMind-js) | JavaScript | Browser and Node.js |
| [ovos-solver-hivemind-plugin](https://github.com/JarbasHiveMind/ovos-solver-hivemind-plugin) | Python | OVOS solver plugin — chat with a HiveMind hub |

## Basic connection

```python
from hivemind_bus_client.client import HiveMessageBusClient

# Reads host, port, key, password from ~/.config/hivemind/_identity.json
client = HiveMessageBusClient()
client.connect()
```

Or with explicit parameters:

```python
client = HiveMessageBusClient(
    host="192.168.1.10",
    port=5678,
    key="my-access-key",
    password="my-password"
)
client.connect()
```

## Sending utterances

```python
from ovos_bus_client.message import Message
from hivemind_bus_client.message import HiveMessage, HiveMessageType

utt = "What time is it?"
msg = HiveMessage(
    HiveMessageType.BUS,
    Message("recognizer_loop:utterance", {"utterances": [utt]})
)
client.emit(msg)
```

## Handling responses

```python
def handle_speak(message):
    print(f"AI says: {message.data['utterance']}")

client.on_mycroft("speak", handle_speak)
```

## Conversational loop

```python
from hivemind_bus_client.client import HiveMessageBusClient
from ovos_bus_client.message import Message
from hivemind_bus_client.message import HiveMessage, HiveMessageType

client = HiveMessageBusClient()
client.connect()

def on_speak(msg):
    print(f"Response: {msg.data['utterance']}")

client.on_mycroft("speak", on_speak)

while True:
    try:
        utterance = input("You: ").strip()
        if utterance:
            client.emit(HiveMessage(
                HiveMessageType.BUS,
                Message("recognizer_loop:utterance", {"utterances": [utterance]})
            ))
    except KeyboardInterrupt:
        break
```

The synchronous `HiveMessageBusClient` runs its WebSocket on a background thread; there is no `close()` method — interrupting the loop is enough to stop sending. (An explicit `close()` exists only on the async client in `hivemind_bus_client.async_client`.)

## Sending binary data (audio)

```python
audio_bytes = b"..."  # Raw PCM data
client.emit(HiveMessage(HiveMessageType.BINARY, audio_bytes))
```

## HTTP client

For environments where a persistent WebSocket is not feasible:

```python
from hivemind_bus_client.http_client import HiveMindHTTPClient

client = HiveMindHTTPClient(host="http://192.168.1.10", port=5679)
client.connect()
```

## Identity management

```bash
# Write the identity file
hivemind-client set-identity \
  --host 192.168.1.10 \
  --port 5678 \
  --key "my-key" \
  --password "my-password" \
  --siteid living-room

# Test the stored identity
hivemind-client test-identity
```

The identity file is at `~/.config/hivemind/_identity.json`.

## Receiving binary data

Inbound binary payloads (TTS audio, files) are dispatched through a
`BinaryDataCallbacks` instance. Subclass it, override the handlers you care
about, and pass it as `bin_callbacks=` to the constructor:

```python
from hivemind_bus_client.client import HiveMessageBusClient, BinaryDataCallbacks

class MyBinaryHandler(BinaryDataCallbacks):
    def handle_receive_tts(self, bin_data: bytes,
                           utterance: str, lang: str, file_name: str):
        # synthesized TTS audio to play back
        with open(file_name, "wb") as f:
            f.write(bin_data)

    def handle_receive_file(self, bin_data: bytes, file_name: str):
        # arbitrary file pushed from the hub
        with open(file_name, "wb") as f:
            f.write(bin_data)

client = HiveMessageBusClient(bin_callbacks=MyBinaryHandler())
client.connect()
```

## Serialization and encryption

`client.emit()` and `client.on()` handle all serialization and encryption automatically:

1. The `HiveMessage` is serialized (JSON or binary framing for protocol v1)
2. The payload is compressed with zlib if enabled
3. The result is encrypted with AES-256-GCM using the session key
4. The encrypted payload is encoded (Z85 + Base91) for text transport

## Decorator helpers

`hivemind_bus_client.decorators` provides decorators that register a function as
a handler for a given message type. Each takes a `bus=` argument (the connected
client). The OVOS-bus variant, `on_mycroft_message`, filters on the inner OVOS
message type:

```python
from hivemind_bus_client.client import HiveMessageBusClient
from hivemind_bus_client.decorators import on_mycroft_message

client = HiveMessageBusClient()
client.connect()

@on_mycroft_message(payload_type="speak", bus=client)
def on_speak(message):
    print(f"AI says: {message.data['utterance']}")
```

Other decorators target specific `HiveMessageType` envelopes:
`on_hive_message`, `on_payload`, `on_ping`, `on_broadcast`, `on_propagate`,
`on_escalate`, `on_handshake`, `on_hello`, `on_query`, `on_cascade`,
`on_rendezvous`, `on_third_party`, and `on_shared_bus`.

## Topology mapping (PING flood)

HiveMind discovers its topology with a PING flood: each node answers a `PING` by
re-emitting its own `PING` carrying the same `flood_id`, propagated across the
hive. There is no separate `PONG` message type — replies are just more `PING`s.

```python
import uuid, time
from hivemind_bus_client.message import HiveMessage, HiveMessageType

flood_id = str(uuid.uuid4())
ping_inner = HiveMessage(
    HiveMessageType.PING,
    payload={
        "flood_id": flood_id,
        "timestamp": time.time(),
        "peer": f"{client.site_id}::{client.session_id}",
        "site_id": client.site_id,
    }
)
# PING must always be wrapped in PROPAGATE so it floods the hive
ping_outer = HiveMessage(HiveMessageType.PROPAGATE, payload=ping_inner)
client.emit(ping_outer)

def on_ping_reply(message):
    payload = message.payload
    route = message.route  # list of {source, targets} hop records
    print(f"PING from {payload['peer']} via {len(route)} hops")

# Responses arrive as PING messages (re-emitted with the same flood_id)
client.on(HiveMessageType.PING, on_ping_reply)
```

For automated topology collection use `HiveMapper` from
`hivemind_bus_client.hive_map`. Call `mapper.start_ping(flood_id)`, feed each
received inner `PING` to `mapper.on_ping(ping_msg)`, and read back the discovered
`mapper.nodes` / `mapper.edges`.

## Request / response

`HiveMessageBusClient` ships blocking helpers that emit a message and wait for a
matching reply on the background thread. All take a `timeout` (seconds, default
`3.0`) and return the received message or `None` on timeout.

```python
def wait_for_message(self, message_type, timeout=3.0)
def wait_for_payload(self, payload_type,
                     message_type=HiveMessageType.THIRDPRTY, timeout=3.0)
def wait_for_mycroft(self, mycroft_msg_type, timeout=3.0)
def wait_for_response(self, message, reply_type=None, timeout=3.0)
def wait_for_payload_response(self, message, payload_type,
                             reply_type=None, timeout=3.0)
```

- `wait_for_message` — wait for the next `HiveMessage` of a given `HiveMessageType`.
- `wait_for_payload` — wait for a `HiveMessage` of `message_type` whose **inner**
  payload type matches `payload_type` (defaults the envelope to `THIRDPRTY`).
- `wait_for_mycroft` — convenience wrapper: `wait_for_payload(mycroft_msg_type,
  message_type=HiveMessageType.BUS)`. Use it to wait for an inner OVOS message
  type (e.g. `"speak"`).
- `wait_for_response` — emit `message`, then wait for a reply. `reply_type`
  defaults to `message.msg_type`. A Mycroft `Message` waits on the inner payload
  type; a `HiveMessage` waits on the envelope type.
- `wait_for_payload_response` — emit `message`, then wait for a reply envelope of
  `reply_type` whose inner payload matches `payload_type`.

The request/response idiom (ask once, block for the answer):

```python
from ovos_bus_client.message import Message
from hivemind_bus_client.message import HiveMessageType

reply = client.wait_for_payload_response(
    message=Message("recognizer_loop:utterance",
                    {"utterances": ["what time is it?"]}),
    payload_type="speak",
    reply_type=HiveMessageType.BUS,
    timeout=5,
)
if reply is not None:
    print(reply.payload.data["utterance"])
```

The same five helpers exist on the async client as coroutines — `await
client.wait_for_response(...)`.

## INTERCOM — end-to-end satellite-to-satellite

`emit_intercom` sends a message addressed to one specific peer, encrypted so that
relays along the path cannot read it. The recipient's RSA **public key is
required**.

```python
def emit_intercom(self, message, pubkey):  # pubkey: str | bytes | RSA.RsaKey
```

On the WebSocket/async client the payload is hybrid-encrypted via
`hybrid_encrypt` (encryption.py): a random AES-256 key encrypts the serialized
message with AES-GCM, that AES key is RSA-OAEP-wrapped with `pubkey`, and the
ciphertext is signed with this node's private key. The envelope is a dict with
base64 fields `encrypted_key`, `ciphertext`, `tag`, `nonce`, and `signature`,
wrapped in a `HiveMessageType.INTERCOM` message.

```python
recipient_pubkey = client.identity.trusted_keys["living-room-hub"]
client.emit_intercom(
    HiveMessage(HiveMessageType.BUS,
                Message("speak", {"utterance": "psst"})),
    pubkey=recipient_pubkey,
)
```

For the inbound side to accept and inject an untargeted INTERCOM message, the
sender's key must be in the receiver's `trusted_keys` (see *Identity & trusted
keys* below); a message whose `target_public_key` matches the receiver is always
accepted.

> **HTTP wire format differs.** `HiveMindHTTPClient.emit_intercom` does **not**
> use the hybrid envelope. It RSA-encrypts the whole serialized message with
> `encrypt_RSA(pubkey, ...)`, signs it, and sends a payload of just
> `{"ciphertext": ..., "signature": ...}` (both base64). Don't mix the two
> formats across transports.

## Async client

`AsyncHiveMessageBusClient` (`hivemind_bus_client.async_client`) mirrors the sync
client on `asyncio` + the `websockets` library. Install with the extra:

```bash
pip install hivemind-bus-client[async]
```

The connect lifecycle is explicit and all I/O is awaited:

```python
import asyncio
from hivemind_bus_client.async_client import AsyncHiveMessageBusClient
from hivemind_bus_client.message import HiveMessage, HiveMessageType
from ovos_bus_client.message import Message

async def main():
    bus = AsyncHiveMessageBusClient(key="my-key", password="my-password",
                                    host="192.168.1.10", port=5678)
    await bus.connect()                 # opens WS, binds protocol, awaits handshake
    try:
        await bus.emit(HiveMessage(
            HiveMessageType.BUS,
            Message("recognizer_loop:utterance", {"utterances": ["hi"]})))
        reply = await bus.wait_for_mycroft("speak", timeout=5)
        if reply is not None:
            print(reply.payload.data["utterance"])
    finally:
        await bus.close()               # closes WS, cancels the receive task

asyncio.run(main())
```

Key coroutines: `connect(bus=None, protocol=None, site_id=None)`, `close()`,
`emit(...)`, `emit_mycroft(...)`, `emit_intercom(...)`, and the five `wait_for_*`
waiters. Handler registration is **synchronous** (so existing protocol handlers
work unchanged): `on(event, func)`, `once(event, func)` (fire-once), and
`remove(event, func)`.

Use the async client when your app already runs an event loop (aiohttp/FastAPI,
discord.py, etc.). Use the sync `HiveMessageBusClient` for scripts and
thread-based apps — it runs its WebSocket on a background thread and needs no loop.

## HTTP client specifics

`HiveMindHTTPClient` (`hivemind_bus_client.http_client`) is a
`threading.Thread`. Its `__init__` calls `self.start()`, so the polling thread
begins as soon as you construct it — you still must call `connect()` before
emitting (it raises `ConnectionAbortedError` otherwise).

```python
from hivemind_bus_client.http_client import HiveMindHTTPClient

client = HiveMindHTTPClient(host="http://192.168.1.10", port=5679)
client.connect()                         # POST /connect, then awaits handshake
resp = client.emit(some_hive_message)    # returns a requests.Response
```

- `emit(message, binary_type=...)` returns the `requests.Response` from
  `POST {base_url}/send_message` (form field `message`), not `None`.
- The background `run()` loop polls `GET {base_url}/get_messages` and
  `GET {base_url}/get_binary_messages` once per second and feeds each result to
  `on_message`. There is no server push; latency is bounded by that poll interval.
- Register handlers with `on(hive_type, func)` for envelopes and
  `on_mycroft(ovos_type, func)` for inner OVOS messages; `remove` /
  `remove_mycroft` unregister them.
- `disconnect()` (POST `/disconnect`) and `shutdown()` stop the loop.

## CASCADE aggregation

A CASCADE query is answered by many nodes across the hive. `CascadeAggregator`
(`hivemind_bus_client.protocol`) collects those responses over a window and picks
one winner.

```python
class CascadeAggregator:
    def __init__(self, timeout, select_callback, emit_callback,
                 expected_responses=None): ...
    def add_response(self, message): ...
    def cancel(self): ...
```

When the **first** response arrives a `timeout`-second timer starts; subsequent
responses are buffered. It resolves when the timer fires **or** once
`expected_responses` responses have arrived (whichever is first), at which point
`select_callback(List[HiveMessage]) -> Optional[HiveMessage]` chooses the winner
and `emit_callback(HiveMessage)` delivers it. `cancel()` drops the timer and
buffered responses without emitting.

Inside `HiveMindSlaveProtocol.handle_cascade`, a node builds one per query:
`select_callback` defaults to `random.choice` (or the protocol's
`cascade_select_callback`), `expected_responses` defaults to the number of nodes
known to the `HiveMapper`, and `emit_callback` injects the winner's inner BUS
payload onto the internal bus.

## Identity & trusted keys

`NodeIdentity` (`hivemind_bus_client.identity`) wraps the identity file
(`~/.config/hivemind/_identity.json` via `JsonConfigXDG`, or pass
`identity_file=`). Beyond the connection fields (`name`, `access_key`,
`password`, `default_master`, `default_port`, `site_id`, `public_key`,
`private_key`):

```python
identity = NodeIdentity()
identity.create_keys()                       # generate + store an RSA keypair
identity.add_trusted_key("living-room-hub",  # alias -> public key
                         peer_public_key)     # True if added, False if alias exists
identity.save()                              # persist to the identity file
identity.reload()                            # re-read from disk
print(identity.trusted_keys)                 # {alias: pubkey, ...}
```

`trusted_keys` is the alias→public-key mapping used to verify peers in PROPAGATE,
CASCADE, and INTERCOM handling — only messages from a trusted key (or explicitly
targeted at this node) are injected onto the bus. Related helpers:
`remove_trusted_key(alias)`, `is_trusted_key(pubkey)`,
`get_trusted_alias(pubkey)`.

## Reconnection caveat

**None of the clients reconnect automatically.** The sync, async, and HTTP
clients all clear their crypto/handshake state on close and stop; there is no
built-in retry/backoff loop. If your deployment needs resilience against dropped
connections, build your own supervisor: detect the drop (e.g. emit failures, the
async receive loop emitting `"close"`, or your own heartbeat) and re-run
`connect()` on a fresh client with backoff.

## Encodings & ciphers

The JSON transport encoding and the symmetric cipher are configurable. The enums
live in `hivemind_bus_client.encryption`:

`SupportedEncodings` (string values): `JSON_B91` (`"JSON-B91"`), `JSON_Z85B`
(`"JSON-Z85B"`), `JSON_Z85P` (`"JSON-Z85P"`), `JSON_B64` (`"JSON-B64"`),
`JSON_URLSAFE_B64` (`"JSON-URLSAFE-B64"`), `JSON_B32` (`"JSON-B32"`), `JSON_HEX`
(`"JSON-HEX"`).

`SupportedCiphers`: `AES_GCM` (`"AES-GCM"`), `CHACHA20_POLY1305`
(`"CHACHA20-POLY1305"`).

Clients default to `JSON_HEX` + `AES_GCM` (the server defaults before they became
configurable; the actual values are negotiated during the handshake). To force
them, set the attributes after construction:

```python
from hivemind_bus_client.encryption import SupportedEncodings, SupportedCiphers

client = HiveMessageBusClient()
client.json_encoding = SupportedEncodings.JSON_B64
client.cipher = SupportedCiphers.CHACHA20_POLY1305
client.connect()
```

## Next

- [Protocol Specification](protocol-spec.md) — wire format, message types, routing
- [Testing Guide](testing.md) — writing tests with the in-process harness

## See also

- [Protocol Reference](protocol-spec.md) — wire format, message types, routing
- [Testing Guide](testing.md) — writing tests with the in-process harness
- [CLI Reference](../reference/cli.md) — `hivemind-client` commands
