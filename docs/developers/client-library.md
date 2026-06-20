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

## See also

- [Protocol Reference](protocol-spec.md) — wire format, message types, routing
- [Testing Guide](testing.md) — writing tests with the in-process harness
- [CLI Reference](../reference/cli.md) — `hivemind-client` commands
