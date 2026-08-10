# Quick Start Guide

This guide gets you from nothing to a satellite talking to a hub. HiveMind lets you
connect lightweight devices as satellites to a central OpenVoiceOS (OVOS) hub, with
per-client permissions.

![imagem](https://github.com/JarbasHiveMind/HiveMind-community-docs/assets/33701864/fb241c4d-ca84-4b47-b917-b398b16f93bd)

## 🧩 Prerequisites

`hivemind-core` is an add-on to a running OVOS install, not a replacement for one. It
talks to the OVOS message bus on `127.0.0.1:8181`. On the hub device:

```bash
pip install ovos-core ovos-messagebus hivemind-core
```

Start `ovos-messagebus` and `ovos-core` before the hub. Without them the hub accepts
connections but nothing answers.

## 1️⃣ Register a satellite

```bash
hivemind-core add-client --name "satellite_1"
```

The output shows:

- Node ID
- Friendly Name
- Access Key
- Password
- Encryption Key

Keep the **Access Key** and **Password** — the satellite needs both.

> ⚠️ The Encryption Key is a protocol v0/v1 pre-shared key. The default
> `min_protocol_version` is `2`, so a client offering only that key is **refused**.
> Use the Password. See [Handshake](./12_handshake.md).

Let the server generate the password. If you supply your own it must pass a strength
check (40 bits by default), so `--password "mypass"` is rejected:

```bash
# supply your own only if it is strong
hivemind-core add-client --name "satellite_1" \
  --password "$(python3 -c 'import os; print(os.urandom(16).hex())')"
```

## 2️⃣ Grant it permission to speak

**A new client is denied on every message.** Its allow-list starts empty, and the
message-type whitelist cannot be turned off. Grant what the satellite needs, using the
Node ID from step 1:

```bash
hivemind-core allow-msg recognizer_loop:utterance 1
hivemind-core allow-msg speak 1
```

> ⚠️ Skipping this step is the most common cause of "my satellite connects but nothing
> happens". Admin status does **not** exempt a client from this whitelist.

## 3️⃣ Start the hub

```bash
hivemind-core listen
```

`listen` takes no options. It binds `0.0.0.0:5678` by default; change that in
`~/.config/hivemind-core/server.json`:

```json
{"network_protocol": {"hivemind-websocket-plugin": {"host": "0.0.0.0", "port": 5678}}}
```

Run `hivemind-core print-config` to see the effective configuration.

> 💡 `hivemind-core` must run on the same device as OVOS.

## 4️⃣ Connect the satellite

On the satellite device:

```bash
pip install hivemind-websocket-client

hivemind-client set-identity \
  --key <ACCESS_KEY> --password <PASSWORD> \
  --host <HUB_IP> --port 5678 --siteid kitchen

hivemind-client test-identity
```

`== Identity successfully connected to HiveMind!` means you are paired. Then
`hivemind-client terminal` opens a chat against the hub.

For a real voice device, pick a satellite in [Satellite Overview](./satellites.md).
Pairing details: [Pairing devices](./03_pairing.md).

## 🔑 Permissions

Every client carries its own permissions:

- `allowed_types` is a **whitelist** and starts **empty** — deny by default.
- Skills and intents can be restricted per client on top of that.
- One predefined role exists: **admin**. It grants the `default` session and
  `BROADCAST`, and it does not bypass `allowed_types`.

Manage them with `allow-msg`, `blacklist-msg`, `blacklist-skill`, `blacklist-intent`,
and the routing family (`allow-broadcast`, `allow-escalate`, `allow-propagate` and
their `blacklist-` counterparts). Full reference: [Permissions](./16_permissions.md).

### Example use cases

- **Basic AI integration**: let a client send natural language instructions.
- **Custom permissions**: restrict an IoT device to specific message types, such as
  `temperature.set`.

## Common commands

```bash
hivemind-core add-client --name "satellite_1"   # register a satellite
hivemind-core list-clients                      # list registered clients
hivemind-core allow-msg <message.type> <node_id> # grant a message type
hivemind-core rename-client <node_id> --name "new name"
hivemind-core print-config                      # effective configuration
hivemind-core listen                            # start the hub
```

`hivemind-core --help` lists every command, and `--help` works per command
(`hivemind-core add-client --help`).

## Next steps

- [Permissions](./16_permissions.md) — narrowing what each client may send
- [Configuration](./config.md) — ports, TLS, database, plugins
- [OVOS Skills Server](./06_skills_server.md) — the hub in an OVOS deployment
- [Nested Hives](./15_nested.md) — connecting Minds to each other
