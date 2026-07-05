# Operations

**This page covers running `hivemind-core` in production** — TLS, reverse-proxy,
service-management, observability, and scaling concerns for an operator. Everything here
is driven by `~/.config/hivemind-core/server.json` and standard system tooling;
`hivemind-core listen` takes no command-line flags.

!!! abstract "In a nutshell"
    - TLS is configured per network protocol in `server.json`, or terminated at a reverse proxy in front of `hivemind-core`.
    - `hivemind-core` emits `hive.client.connect` / `disconnect` / `connection.error` events on the OVOS messagebus, and logs rejected connections at `ERROR`.
    - SQLite and JSON backends are single-node; Redis is the shared backend for running multiple `hivemind-core` instances against one store.

---

## TLS

TLS is configured **per network protocol** in `server.json`, not on the command line. The
relevant keys live under `network_protocol.hivemind-websocket-plugin` (and the HTTP
plugin, if used):

```json
{
  "network_protocol": {
    "hivemind-websocket-plugin": {
      "host": "0.0.0.0",
      "port": 5678,
      "ssl": true,
      "cert_dir": "~/.local/share/hivemind",
      "cert_name": "hivemind"
    }
  }
}
```

Set `ssl` to `true` and point `cert_dir`/`cert_name` at the certificate to serve
(`<cert_dir>/<cert_name>.crt` and `.key`). TLS is off by default.

**Self-signed vs real certificates.** On a trusted local network a self-signed
certificate is sufficient; satellites must then be told to accept it by passing
`--selfsigned` on their connect commands (`hivemind-voice-sat`, `hivemind-voice-relay`,
HiveMind-cli, …). For anything exposed beyond the LAN, serve a CA-issued certificate so
satellites validate it normally — or terminate TLS at a reverse proxy (below) and leave
`hivemind-core` itself plaintext on the loopback interface. See
[Security](../concepts/security.md) for the certificate guidance.

---

## Reverse proxy

For production it is common to terminate TLS at nginx or Caddy in front of the WebSocket
listener rather than configuring `ssl` on `hivemind-core` itself. The proxy holds the real (CA-issued)
certificate on `443` and forwards to `hivemind-core` on `127.0.0.1:5678`, upgrading the
WebSocket connection. When you do this, do **not** also expose `5678`/`5679` directly to
the internet — only the proxy should be reachable.

The Docker deployment guide covers a concrete proxy setup; see
[Docker Deployment](docker.md#with-ssl-via-reverse-proxy).

---

## systemd

Run `hivemind-core` as a managed service. Reuse the unit from the
[OVOS Skills Server](ovos-server.md#systemd-service) guide:

```ini
# /etc/systemd/system/hivemind-core.service
[Unit]
Description=HiveMind Core
After=network.target ovos-messagebus.service
Requires=ovos-messagebus.service

[Service]
Type=simple
User=ovos
ExecStart=/home/ovos/.venvs/ovos/bin/hivemind-core listen
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable hivemind-core
sudo systemctl start hivemind-core
```

The `After=`/`Requires=ovos-messagebus.service` lines matter for an **OVOS skills server**:
its agent protocol talks to `ovos-messagebus`, which must be up first. An
[A2A server](a2a-server.md) or [persona server](persona-server.md) does not require `ovos-messagebus`
— drop those two lines if you are not running OVOS.

---

## Connection errors and observability

Server-side connection failures mostly surface to the client as a silent socket disconnect —
the satellite simply fails to complete the handshake. Server-side, however, `hivemind-core` emits
internal-bus events on the OVOS messagebus that its agent protocol is wired to (in OVOS
mode this is `ovos-messagebus`). Rejected connections raise an error event:

| Event                          | Emitted when                                                                 |
|--------------------------------|------------------------------------------------------------------------------|
| `hive.client.connect`          | A client connects (payload carries `key`, `session_id`).                     |
| `hive.client.disconnect`       | A client disconnects (payload carries `key`).                                |
| `hive.client.connection.error` | A connection is rejected — invalid access key **or** protocol-version mismatch. |

Both rejection paths emit the same `hive.client.connection.error` event but with
distinct `error` payload values, so you can tell them apart:

- `handle_invalid_key_connected` (a client presented an unknown / disallowed API key)
  emits `{"error": "invalid access key", "peer": <peer>}` and logs
  `Client provided an invalid api key`.
- `handle_invalid_protocol_version` (a client failed the protocol-version requirements)
  emits `{"error": "protocol error", "peer": <peer>}` and logs
  `Client does not satisfy protocol requirements`.

An operator can observe invalid-key and bad-protocol-version attempts in two ways:

1. **Watch the messagebus.** Listen for `hive.client.connection.error` on the OVOS
   messagebus `hivemind-core` publishes to (for an OVOS skills server, the `ovos-messagebus` the agent
   protocol connects to) and inspect the `error`/`peer` fields. This is the structured,
   real-time signal — wire it into your monitoring to alert on repeated rejected
   connections from a single peer (a sign of a misconfigured satellite or a brute-force
   attempt).
2. **Read the logs.** The same two handlers log at `ERROR` level
   (`Client provided an invalid api key` / `Client does not satisfy protocol
   requirements`), so the rejections are visible in `journalctl -u hivemind-core` even
   without a bus listener.

Normal `hive.client.connect` / `hive.client.disconnect` events on the same bus give you a
running picture of which satellites are attached.

---

## Scaling and multi-instance

A single `hivemind-core` instance keeps client and permission state in its [database
backend](../concepts/databases.md). The backend choice determines whether you can run
more than one instance against the same state:

- **SQLite** (default) and **JSON** are single-node — the database is a local file; only
  one `hivemind-core` process should own it.
- **Redis** ([`hivemind-redis-database`](../concepts/databases.md)) is the shared backend.
  Point multiple `hivemind-core` instances at the same Redis and they share client
  records and permissions, so you can run several instances (behind a load balancer, or for
  redundancy) without each maintaining its own client list. Add a client once and every
  instance sees it.

Select the backend in `server.json` under the `database` block; the same configuration is
read by `listen` and by every client-management command, so they all operate on the same
store. See [Database Backends](../concepts/databases.md) for the full configuration and
the `migrate-db` workflow.

---

## Health checking

`hivemind-core` does not expose a dedicated health endpoint, so health checks are
practical rather than built-in:

- **Is the WebSocket port accepting handshakes?** The most direct liveness probe is a TCP
  connect to the configured port (`5678` by default) — if the socket accepts the
  connection the listener is up. A fuller check is to attempt a real client handshake
  with a known-good API key and confirm it succeeds; a `hive.client.connect` event on the
  bus (and no `hive.client.connection.error`) confirms an end-to-end healthy path.
- **Service liveness.** Under systemd, `systemctl is-active hivemind-core` and the
  `Restart=on-failure` policy cover process-level health; pair that with the socket probe
  above for the network path.
- **Presence announce/scan.** If [HiveMind-presence](../concepts/discovery.md) is running
  alongside `hivemind-core`, a `hivemind-presence scan` should find its announcement on the
  local network — a coarse confirmation that the node is reachable and advertising. Note
  presence is an optional, separate process from `hivemind-core listen`, so its absence
  does not by itself mean `hivemind-core` is down.

---

## Next

- [Docker Deployment](docker.md) — containerized `hivemind-core`, Redis, and the reverse-proxy setup.
- [Database Backends](../concepts/databases.md) — single-node vs shared (Redis) state.
- [Security](../concepts/security.md) — certificates, keys, and permissions.
- [Discovery](../concepts/discovery.md) — presence announce/scan for health and reach.

---

## Source

Validated against the HiveMind source:

- [`hivemind_core/protocol.py`](https://github.com/JarbasHiveMind/HiveMind-core/blob/HEAD/hivemind_core/protocol.py) — the `hive.client.connect` / `disconnect` / `connection.error` events and the invalid-key vs protocol-version rejection paths
- [`hivemind_core/config.py`](https://github.com/JarbasHiveMind/HiveMind-core/blob/HEAD/hivemind_core/config.py) — the `server.json` schema, per-protocol TLS keys, and the database block
