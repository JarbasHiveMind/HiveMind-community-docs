# Quick Start Guide

This guide gets you from nothing to a satellite that talks to a hub. HiveMind extends your OpenVoiceOS (OVOS) setup across multiple devices, even low-resource hardware. It connects lightweight devices as satellites to a central OVOS hub, with centralized control and fine-grained permissions.

![imagem](https://github.com/JarbasHiveMind/HiveMind-community-docs/assets/33701864/fb241c4d-ca84-4b47-b917-b398b16f93bd)

## Prerequisites

`hivemind-core` is an add-on to a running OVOS install. It talks to the OVOS message bus on `127.0.0.1:8181`. Install both on the hub device:

```bash
pip install ovos-core ovos-messagebus hivemind-core
```

Start `ovos-messagebus` and `ovos-core` before you start the hub. Without them the hub accepts connections, but nothing answers.

## 1. Register a satellite

```bash
hivemind-core add-client --name "satellite_1"
```

The output shows these details:

- Node ID
- Friendly Name
- Access Key
- Password
- Encryption Key

Keep the Access Key and the Password. The satellite needs both.

Let the server generate the password. A password that you supply must pass a strength check of 40 bits by default, so a value such as `mypass` is refused. To supply your own:

```bash
hivemind-core add-client --name "satellite_1" \
  --password "$(python3 -c 'import os; print(os.urandom(16).hex())')"
```

The Encryption Key is a protocol v0/v1 pre-shared key. The default `min_protocol_version` is 2, so a client that offers only that key is refused. Use the Password. See [Handshake](12_handshake.md).

## 2. Grant the satellite permission to speak

A new client is denied on every message. Its allow-list starts empty, and you cannot turn the message-type whitelist off. Grant what the satellite needs. Use the Node ID from step 1:

```bash
hivemind-core allow-msg recognizer_loop:utterance 1
hivemind-core allow-msg speak 1
```

If you skip this step, the satellite connects and then does nothing. This is the most common cause of that symptom. Admin status does not exempt a client from the whitelist.

## 3. Start the hub

```bash
hivemind-core listen
```

`listen` takes no options. It binds `0.0.0.0:5678`. To change the address, edit `~/.config/hivemind-core/server.json`:

```json
{"network_protocol": {"hivemind-websocket-plugin": {"host": "0.0.0.0", "port": 5678}}}
```

Run `hivemind-core print-config` to see the effective configuration.

> `hivemind-core` must run on the same device as OVOS.

## 4. Connect the satellite

Run these commands on the satellite device. The repository is named HiveMind-websocket-client, and the distribution it publishes is `hivemind-bus-client`:

```bash
pip install hivemind-bus-client

hivemind-client set-identity \
  --key <ACCESS_KEY> --password <PASSWORD> \
  --host <HUB_IP> --port 5678 --siteid kitchen

hivemind-client test-identity
```

The message `== Identity successfully connected to HiveMind!` means the satellite is paired. Then run `hivemind-client terminal` to open a chat against the hub.

For a voice device, pick a satellite in [Satellite Overview](satellites.md). Pairing details are in [Pairing devices](03_pairing.md).

## Permissions

HiveMind Core uses a flexible permissions system. Each client's permissions are configurable. By default:

- `allowed_types` is a whitelist and starts empty. The default is deny.
- You can restrict skills and intents per client on top of that.
- One predefined role exists: admin. It grants the reserved `default` session and the right to originate `BROADCAST`. It does not bypass `allowed_types`.

Manage permissions for a client with `allow-msg`, `blacklist-msg`, `blacklist-skill` and `blacklist-intent`, and with the routing commands `allow-broadcast`, `allow-escalate` and `allow-propagate` and their `blacklist-` counterparts. See [Permissions](16_permissions.md).

Example use cases:

- Basic AI integration: let a simple client send natural language instructions.
- Custom permissions: restrict an IoT device so it only sends specific message types, such as `temperature.set`.

## HiveMind Core commands overview

These are the basic commands for managing clients and their permissions:

```bash
hivemind-core add-client --name "satellite_1"    # register a satellite
hivemind-core list-clients                       # list registered clients
hivemind-core allow-msg <message.type> <node_id> # grant a message type
hivemind-core rename-client <node_id> --name "new name"
hivemind-core print-config                       # show effective configuration
hivemind-core listen                             # start the hub
```

For detailed help on a command, use `--help` (for example, `hivemind-core add-client --help`).

---
[Home](index.md) · [Terminology →](02_terminology.md)
