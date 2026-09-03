# Transports

Think of a transport as the road the message drives on, not the message itself. A
satellite and hivemind-core have to agree on *a* road — a WebSocket, an HTTP request, an
MQTT broker — but the cargo riding on it is the same encrypted HiveMind protocol no
matter which they pick. Swap the road and the [encryption](../concepts/security.md) and
message format don't change a bit; the carrier only ever sees sealed envelopes. Which is
why, for almost everyone, this page ends at one word: WebSocket.

!!! abstract "In a nutshell"
    - Transports register under the `hivemind.network.protocol` plugin entry-point group; the carrier is swappable without touching encryption or the message format.
    - WebSocket (port `5678`) is the reference and default; HTTP is request/response.
    - MQTT is broker-mediated (alpha) and Usenet is store-and-forward (experimental, source-install only for now) — in both the carrier only ever sees ciphertext.

!!! tip "You rarely need to change this"
    If you don't know which transport you want, you want WebSocket — it's the
    reference implementation and the default in every example. The rest of this page is
    for people with a specific need (broker-mediated IoT, censorship-resistant relay).

---

## Status at a glance

Four roads exist, but they are not equally paved. Before you fall for a clever one, read
the **PyPI status** column — it's the difference between "ship it" and "fun to read
about." All four register under the same `hivemind.network.protocol`
[plugin](../concepts/plugins.md) entry-point group:

| Transport | Package | Entry point (`hivemind.network.protocol`) | PyPI status | Notes |
|---|---|---|---|---|
| WebSocket | `hivemind-websocket-protocol` | `hivemind-websocket-plugin` | published | Reference / default. Port **5678**. TLS via `ssl` / `cert_dir` / `cert_name`. |
| HTTP | `hivemind-http-protocol` | `hivemind-http-plugin` | published (early) | Request/response. Defaults to port **5679**, distinct from WebSocket's `5678`. |
| MQTT | `hivemind-mqtt-protocol` | `hivemind-mqtt-plugin` | alpha only | Broker-mediated (good for IoT). The broker only ever sees ciphertext. |
| Usenet | `hivemind-usenet` | `hivemind-usenet-wormhole` (class `UsenetWormhole`) | **source-install only** | Experimental, store-and-forward, censorship-resistant. |

!!! warning "Maturity matters"
    **MQTT is alpha** and **Usenet is experimental, source-install only.** The repo is
    public, but `pip install hivemind-usenet` isn't live yet — install from source until
    it publishes. Treat both as for-the-curious.

---

## WebSocket (default)

The reference transport. A persistent, bidirectional socket on port `5678`. TLS is
enabled per protocol in `server.json` by setting `ssl: true` and pointing
`cert_dir` / `cert_name` at the certificate to serve (see
[Operations → TLS](operations.md#tls)). This is what every quickstart and example uses.

---

## HTTP

A request/response transport for environments where a long-lived socket is awkward
(some proxies, serverless front-ends). Each utterance is a request; the answer comes
back in the response.

The HTTP plugin defaults to port `5679` on its own, distinct from the WebSocket
listener's `5678` default. Enabling both with no `port` set works out of the box.

---

## MQTT (alpha)

??? note "Advanced: broker-mediated transport for IoT"
    MQTT routes HiveMind traffic through an MQTT broker instead of a direct socket —
    handy in IoT fleets that already run a broker. Because HiveMind encrypts at the
    application layer, **the broker only ever sees ciphertext**: it relays messages
    but cannot read them.

    Configuration keys (under the plugin's block in `server.json`):

    | Key | Default | Meaning |
    |---|---|---|
    | `broker_host` | `localhost` | MQTT broker address |
    | `broker_port` | `1883` | MQTT broker port |
    | `broker_username` / `broker_password` | — | broker credentials, if required |
    | `tls` | `false` | Enable TLS to the broker |
    | `tls_ca_certs` / `tls_certfile` / `tls_keyfile` | — | CA bundle path / client cert path / client key path (mTLS) |
    | `topic_prefix` | `hivemind` | prefix for the topics used |

    This is **alpha** — pin and test before relying on it.

---

## Usenet (experimental)

??? note "Advanced: censorship-resistant store-and-forward"
    The Usenet transport (`hivemind-usenet`, class `UsenetWormhole`, entry point
    `hivemind-usenet-wormhole`) relays HiveMind traffic over Usenet — a
    store-and-forward, censorship-resistant carrier. It is **experimental**. The repo
    is public, but `pip install hivemind-usenet` isn't live yet — install from source
    (`git clone` + `pip install .`) until it publishes.

---

## Not a transport: the audio binary protocol

The [Audio Binary Protocol](audio-binary-protocol.md) is a *different* kind of plugin.
It is **not** a network transport — it's a binary **payload handler** (entry-point group
`hivemind.binary.protocol`) that runs *on top of* whichever transport you chose, so the
server can receive raw audio, run wake-word/STT/TTS, and stream audio back. You can mix it
with any transport above. See [Audio Binary Protocol](audio-binary-protocol.md).

---

## How this all plugs together

Every transport is a [plugin](../concepts/plugins.md) discovered by entry point, which
is why swapping the carrier doesn't touch the encryption or the message format. See
[Plugin Architecture](../concepts/plugins.md) for how `hivemind-core` resolves and loads them.

---

## Source

Validated against the HiveMind source:

- [`hivemind_websocket_protocol/__init__.py`](https://github.com/JarbasHiveMind/hivemind-websocket-protocol/blob/HEAD/hivemind_websocket_protocol/__init__.py) — the default WebSocket transport, port `5678`, and TLS keys
- [`hivemind_http_protocol/__init__.py`](https://github.com/JarbasHiveMind/hivemind-http-protocol/blob/HEAD/hivemind_http_protocol/__init__.py) — the HTTP transport and its default port `5679`
- [`hivemind-mqtt-protocol`](https://github.com/JarbasHiveMind/hivemind-mqtt-protocol) — the alpha MQTT transport and its broker config keys
- [`hivemind-usenet`](https://github.com/JarbasHiveMind/hivemind-usenet) — the experimental Usenet transport
