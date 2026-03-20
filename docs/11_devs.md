# Developer Guide

HiveMind provides multiple client libraries for building satellites, bridges, and integrations. This guide covers the primary Python client.

## Client Libraries

- **[HiveMind-websocket-client](https://github.com/JarbasHiveMind/hivemind_websocket_client)** (Python) — Full-featured WebSocket client with encryption, CLI, and async support.
- **[HiveMindJs](https://github.com/JarbasHiveMind/HiveMind-js)** (JavaScript) — Browser and Node.js client.
- **[ovos-solver-hivemind-plugin](https://github.com/JarbasHiveMind/ovos-solver-hivemind-plugin)** (Python) — OVOS solver plugin to chat with HiveMind.

## HiveMessageBusClient API

The primary class for real-time, bidirectional communication over WebSockets.

**Source**: `hivemind_bus_client.client.HiveMessageBusClient` ([`hivemind-websocket-client` docs/client_api.md](https://github.com/JarbasHiveMind/hivemind_websocket_client/blob/dev/docs/client_api.md))

### Basic Connection

The client automatically loads identity from `~/.config/hivemind-core/identity.json` and executes the `poorman_handshake.PasswordHandShake` for encrypted transport (AES-256-GCM).

```python
from hivemind_bus_client.client import HiveMessageBusClient

# Reads host, port, key, password from identity file
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

### Handling Events

Listen for AI events using `on_mycroft` (inherited from `ovos_bus_client.client.MessageBusClient`):

```python
def handle_speak(message):
    print(f"AI says: {message.data['utterance']}")

client.on_mycroft("speak", handle_speak)
```

### Sending Utterances

Wrap an OVOS `Message` in a `HiveMessage` and use `emit`:

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

### Sending Binary Data

Raw PCM audio or other binary payloads use the `BIN` message type:

```python
audio_bytes = b"..."  # Raw PCM data
hive_msg = HiveMessage(HiveMessageType.BIN, audio_bytes)
client.emit(hive_msg)
```

### Network Discovery (PING/PONG)

Map all reachable nodes in the hive using PING messages (always wrapped in PROPAGATE):

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
ping_outer = HiveMessage(HiveMessageType.PROPAGATE, payload=ping_inner)
client.emit(ping_outer)

def on_pong(message):
    payload = message.payload  # inner PONG dict
    route = message.route      # List[{source, targets}] — hive path
    print(f"PONG from {payload['peer']} via {len(route)} hops")

client.on(HiveMessageType.PONG, on_pong)
```

For automated topology collection, use `HiveMapper` from `hivemind_core.hive_map`.

## HiveMindHTTPClient API

For scenarios where a persistent WebSocket is not feasible (polling, REST-only environments).

**Source**: `hivemind_bus_client.http_client.HiveMindHTTPClient`

```python
from hivemind_bus_client.http_client import HiveMindHTTPClient

client = HiveMindHTTPClient(host="http://192.168.1.10", port=5679)
client.connect()  # Performs encrypted handshake via HTTP POST
```

## Identity Management

Each HiveMind client is identified by a unique identity file: `~/.config/hivemind-core/identity.json`.

**Source**: `hivemind-websocket-client/docs/identity.md`

This file stores:
- **`access_key`**: Authorization credential for the hub
- **`password`**: Handshake password (used by `poorman_handshake`)
- **`host`**: Default hub address
- **`port`**: Default hub port (WebSocket: 5678, HTTP: 5679)
- **`site_id`**: Location identifier for context routing
- **`peer_id`**: Unique satellite identifier

Create or update via CLI:

```bash
$ hivemind-client set-identity --host 192.168.1.10 --port 5678 \
    --key "my-key" --password "my-password"
```

## Binary Data Handlers

For protocols that transport non-JSON binary data (e.g., audio streams), `BIN` messages carry opaque byte payloads.

**Source**: `hivemind-websocket-client/docs/binary_handlers.md`

Register custom binary handlers:

```python
from hivemind_bus_client.message import HiveMessageType

def handle_audio_bin(data: bytes, context: dict) -> None:
    """Process raw audio bytes from a satellite."""
    # Write to file, stream to TTS, etc.
    pass

client.on_bin(HiveMessageType.BIN, handle_audio_bin)
```

## Message Types

All HiveMessage types are defined in `HiveMessageType` enum:

- **`BUS`**, **`SHARED_BUS`**, **`ESCALATE`**, **`BROADCAST`**, **`PROPAGATE`**, **`INTERCOM`** — Standard message routing modes
- **`PING`**, **`PONG`** — Network discovery (always wrapped in PROPAGATE)
- **`HELLO`**, **`HANDSHAKE`** — Connection management (automatic)
- **`BIN`** — Raw binary payload
- **`THIRDPRTY`** — User-defined application payload

For full details, see [Protocol: Message Types Reference](/HiveMind-community-docs/docs/04_protocol.md#message-types-reference).

## Installation

Install via pip:

```bash
pip install hivemind-websocket-client
```

Or with `uv` (recommended):

```bash
uv pip install hivemind-websocket-client
```

Development installation:

```bash
uv pip install -e git+https://github.com/JarbasHiveMind/hivemind_websocket_client.git#egg=hivemind-websocket-client
```

## CLI Reference

The installed `hivemind-client` command provides tools for interacting with a HiveMind hub.

**Source**: `hivemind-websocket-client/docs/cli_guide.md`, `cli.md`

Common commands:

```bash
# Send an utterance
$ hivemind-client send-utterance "what time is it"

# Send a mycroft message
$ hivemind-client send-mycroft --msg "speak" --payload '{"utterance": "hello"}'

# Set identity credentials
$ hivemind-client set-identity --host 192.168.1.10 --key my-key --password my-pass

# Discover nodes in the hive
$ hivemind-client propagate --msg "hive-mapper:ping"
```

Run `hivemind-client --help` for full command list.

## Examples

**Conversational Loop** — Send utterances and listen for responses:

```python
from hivemind_bus_client.client import HiveMessageBusClient
from ovos_bus_client.message import Message
from hivemind_bus_client.message import HiveMessage, HiveMessageType
import sys

client = HiveMessageBusClient()
client.connect()

def on_speak(msg):
    print(f"Response: {msg.data['utterance']}")

client.on_mycroft("speak", on_speak)

while True:
    try:
        utterance = input("You: ").strip()
        if utterance:
            hive_msg = HiveMessage(
                HiveMessageType.BUS,
                Message("recognizer_loop:utterance", {"utterances": [utterance]})
            )
            client.emit(hive_msg)
    except KeyboardInterrupt:
        break

client.close()
```

**Audio Streaming** — Send raw audio for STT:

```python
import pyaudio
from hivemind_bus_client.message import HiveMessage, HiveMessageType

CHUNK = 2048
FORMAT = pyaudio.paFloat32
CHANNELS = 1
RATE = 16000

p = pyaudio.PyAudio()
stream = p.open(format=FORMAT, channels=CHANNELS, rate=RATE, input=True, frames_per_buffer=CHUNK)

client = HiveMessageBusClient()
client.connect()

try:
    while True:
        data = stream.read(CHUNK)
        msg = HiveMessage(HiveMessageType.BIN, data)
        client.emit(msg)
finally:
    stream.stop_stream()
    stream.close()
    p.terminate()
    client.close()
```

## Serialization & Encryption

Before being sent over the network, `HiveMessage` objects are:

1. **Serialized** using `hivemind_bus_client.serialization.HiveMessageSerializer`
2. **Encrypted** using AES-256-GCM in `hivemind_bus_client.encryption.HiveMessageEncryptor`
3. **Encoded** using Z85+Base91 for safe text transport in `hivemind_bus_client.encodings`

These steps are transparent to the user — `client.emit()` and `client.on()` handle it automatically.

## Decorators

The library includes decorators in `hivemind_bus_client.decorators` for wrapping functions as HiveMind services:

- **`@with_hivemind`** — Automatically manages client connection and cleanup for decorated functions.

```python
from hivemind_bus_client.decorators import with_hivemind

@with_hivemind
def my_service(client: HiveMessageBusClient):
    client.emit(...)  # client is already connected
    # cleanup happens automatically
```

## See Also

- [Protocol: Message Types & Routing](/HiveMind-community-docs/docs/04_protocol.md)
- [Session Management & Context Keys](/HiveMind-community-docs/docs/04_protocol.md#session-management--context-keys)
- [Satellites: Voice Satellite](/HiveMind-community-docs/docs/07_voicesat.md)
- [Satellites: Mic Satellite](/HiveMind-community-docs/docs/07_micsat.md)
