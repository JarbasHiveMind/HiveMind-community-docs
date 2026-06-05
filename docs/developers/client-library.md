# Developer Guide — Client Library

`hivemind-websocket-client` is the primary Python library for building HiveMind clients. It handles connection management, the PBKDF2 handshake, AES-256-GCM encryption, and serialization transparently.

## Install

```bash
pip install hivemind-websocket-client
```

## Libraries

| Library | Language | Notes |
|---|---|---|
| [hivemind-websocket-client](https://github.com/JarbasHiveMind/hivemind_websocket_client) | Python | Primary client; WebSocket + HTTP + async + MQTT |
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

client.close()
```

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

## Custom binary handlers

```python
from hivemind_bus_client.message import HiveMessageType

def handle_audio_bin(data: bytes, context: dict) -> None:
    # Process raw audio bytes
    pass

client.on_bin(HiveMessageType.BINARY, handle_audio_bin)
```

## Serialization and encryption

`client.emit()` and `client.on()` handle all serialization and encryption automatically:

1. The `HiveMessage` is serialized (JSON or binary framing for protocol v1)
2. The payload is compressed with zlib if enabled
3. The result is encrypted with AES-256-GCM using the session key
4. The encrypted payload is encoded (Z85 + Base91) for text transport

## Decorator helper

```python
from hivemind_bus_client.decorators import with_hivemind

@with_hivemind
def my_service(client: HiveMessageBusClient):
    client.emit(...)
    # connection is cleaned up automatically when the function returns
```

## Topology mapping (PING/PONG)

```python
import uuid, time
from hivemind_bus_client.message import HiveMessage, HiveMessageType

ping_id = str(uuid.uuid4())
ping_inner = HiveMessage(
    HiveMessageType.PING,
    payload={
        "ping_id": ping_id,
        "timestamp": time.time(),
        "peer": client.peer,
        "site_id": client.site_id,
    }
)
# PING must always be wrapped in PROPAGATE
ping_outer = HiveMessage(HiveMessageType.PROPAGATE, payload=ping_inner)
client.emit(ping_outer)

def on_pong(message):
    payload = message.payload
    route = message.route  # List[{source, targets}]
    print(f"PONG from {payload['peer']} via {len(route)} hops")

client.on(HiveMessageType.PONG, on_pong)
```

For automated topology collection use `HiveMapper` from `hivemind_core.hive_map`.

## See also

- [Protocol Reference](protocol-spec.md) — wire format, message types, routing
- [Testing Guide](testing.md) — writing tests with the in-process harness
- [CLI Reference](../reference/cli.md) — `hivemind-client` commands
