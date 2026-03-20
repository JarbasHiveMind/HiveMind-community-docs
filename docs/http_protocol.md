# HTTP Protocol

The HTTP transport layer is an alternative to WebSocket for scenarios where persistent connections are not feasible or desired (polling devices, REST-only clients, firewalls restricting WebSockets).

**Source**: `hivemind-http-protocol` package

## Overview

The HTTP transport is implemented as a `NetworkProtocol` plugin and uses Tornado for request handling. Unlike WebSocket's persistent bidirectional connection, HTTP operates as request-response with per-client message queues on the server.

**Primary Class**: `HiveMindHttpProtocol(NetworkProtocol)` — `hivemind_http_protocol/__init__.py`

## Authentication

Authentication is performed via a custom `authorization` request argument containing a Base64-encoded `useragent:key` string. This is validated against the HiveMind database before any message processing.

**Method**: `HiveMindHttpHandler.decode_auth()` — Extracts and validates the access key on every request.

## Client Management

Each unique access key gets a `HiveMindClientConnection` instance. The handler manages:

- **Connection State**: Tracks which client keys are active
- **Message Sinks**: Outgoing messages are placed into per-client queues (`undelivered` for OVOS messages, `undelivered_bin` for binary) instead of immediate socket delivery
- **Handshake**: Initializes `poorman_handshake.PasswordHandShake` if the client has a configured password

**Method**: `HiveMindHttpHandler.get_client(useragent, key, cache)`

## Server Lifecycle

### Initialization (`run()`)

- **SSL Setup**: Calls `create_self_signed_cert()` if `ssl: true` is configured
- **AsyncIO**: Configures `AnyThreadEventLoopPolicy` for broad compatibility
- **Routes**: Registers HTTP handlers for message endpoints

**Source**: `HiveMindHttpProtocol.run()`

## Message Queue Polling

Clients poll HTTP endpoints to retrieve queued outgoing messages. Per-client queues decouple message delivery from the HTTP request cycle.

**Endpoints**:
- **`GET /send`** — Retrieve pending OVOS messages from `client.undelivered` queue
- **`GET /send_bin`** — Retrieve pending binary messages from `client.undelivered_bin` queue

This allows clients on intermittent connections (mobile, IoT) to reconnect and retrieve buffered responses without losing messages.

## Comparison with WebSocket

| Aspect | HTTP | WebSocket |
|--------|------|-----------|
| Connection | Stateless, request-response | Persistent bidirectional |
| Handshake | Standard HTTPS | WebSocket upgrade + password/PGP handshake |
| Message Delivery | Pull-based (client polls) | Push-based (server sends) |
| Latency | Higher (polling interval) | Lower (immediate) |
| Firewall Friendly | ✅ Standard HTTP/HTTPS | ⚠️ Some firewalls block WebSocket |
| CPU / Device Power | Lower (no persistent socket) | Higher (socket overhead) |
| Use Case | IoT, mobile, REST clients | Voice satellites, real-time apps |

## Configuration

HTTP transport is enabled via `server.json`:

```json
{
  "network_protocols": {
    "http": {
      "host": "0.0.0.0",
      "port": 5679,
      "ssl": false,
      "ssl_keyfile": "",
      "ssl_certfile": ""
    }
  }
}
```

## See Also

- [Protocol: Message Types & Routing](/HiveMind-community-docs/docs/04_protocol.md)
- [WebSocket Protocol Implementation](/HiveMind-community-docs/docs/04_protocol.md#message-types-reference)
- [Client Libraries: HiveMindHTTPClient](/HiveMind-community-docs/docs/11_devs.md#hivemindhttpclient-api)
