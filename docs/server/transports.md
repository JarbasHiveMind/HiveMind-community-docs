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
    - MQTT is broker-mediated (alpha) and Usenet is store-and-forward (experimental, git-only) — in both the carrier only ever sees ciphertext.

!!! tip "You rarely need to change this"
    If you don't know which transport you want, you want WebSocket — it's the
    reference implementation and the default in every example. The rest of this page is
    for people with a specific need (broker-mediated IoT, censorship-resistant relay).

---

## Status at a glance

Transports register under the `hivemind.network.protocol`
[plugin](../concepts/plugins.md) entry-point group. They differ a lot in maturity —
read the **PyPI status** column before you build on one:

| Transport | Package | Entry point (`hivemind.network.protocol`) | PyPI status | Notes |
|---|---|---|---|---|
| WebSocket | `hivemind-websocket-protocol` | `hivemind-websocket-plugin` | published | Reference / default. Port **5678**. TLS via `ssl` / `cert_dir` / `cert_name`. |
| HTTP | `hivemind-http-protocol` | `hivemind-http-plugin` | published (early) | Request/response. Source default port **5678** — the manual's `5679` is an explicit override to avoid colliding with WebSocket. |
| MQTT | `hivemind-mqtt-protocol` | `hivemind-mqtt-plugin` | alpha only | Broker-mediated (good for IoT). The broker only ever sees ciphertext. |
| Usenet | `hivemind-usenet` | `hivemind-usenet-wormhole` (class `UsenetWormhole`) | **not on PyPI — git install only** | Experimental, store-and-forward, censorship-resistant. |

!!! warning "Maturity matters"
    **MQTT is alpha** and **Usenet is experimental and git-only.** Don't assume
    `pip install hivemind-usenet` works — it is not on PyPI, and it depends on
    git-only carrier libraries. Treat both as for-the-curious until they ship to PyPI.

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

!!! warning "Set an explicit port"
    The HTTP plugin's **source default port is `5678`** — the same as WebSocket. The
    default `server.json` ships it on `5679` as a deliberate override. If you enable
    the HTTP plugin and forget to set `port`, it will collide with the WebSocket
    listener. Always give the HTTP block its own port.

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
    | `tls` / `cert` | — | TLS to the broker |
    | `topic_prefix` | `hivemind` | prefix for the topics used |
    | `hash_topics` | — | hash topic names so the broker can't infer routing from them |

    This is **alpha** — pin and test before relying on it.

---

## Usenet (experimental)

??? note "Advanced: censorship-resistant store-and-forward"
    The Usenet transport (`hivemind-usenet`, class `UsenetWormhole`, entry point
    `hivemind-usenet-wormhole`) relays HiveMind traffic over Usenet — a
    store-and-forward, censorship-resistant carrier. It is **experimental** and
    **not published on PyPI**; you install it from git, and it depends on git-only
    carrier libraries. Do not write deployment docs that assume
    `pip install hivemind-usenet` works.

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
- [`hivemind_http_protocol/__init__.py`](https://github.com/JarbasHiveMind/hivemind-http-protocol/blob/HEAD/hivemind_http_protocol/__init__.py) — the HTTP transport and its source-default port `5678`
- [`hivemind-mqtt-protocol`](https://github.com/JarbasHiveMind/hivemind-mqtt-protocol) — the alpha MQTT transport and its broker config keys
- [`hivemind-usenet`](https://github.com/JarbasHiveMind/hivemind-usenet) — the experimental, git-only Usenet transport
