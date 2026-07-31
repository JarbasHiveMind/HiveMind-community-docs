# Quick Start Guide

This guide gets you started with HiveMind. HiveMind extends your OpenVoiceOS (OVOS) setup across multiple devices, even low-resource hardware. It connects lightweight devices as satellites to a central OVOS hub, with centralized control and fine-grained permissions.

![imagem](https://github.com/JarbasHiveMind/HiveMind-community-docs/assets/33701864/fb241c4d-ca84-4b47-b917-b398b16f93bd)

## Installation

Install the `hivemind-core` package on your OVOS device:

```bash
pip install hivemind-core
```

## Adding a satellite device

Once the server runs, add client credentials for each satellite device you want to connect.

Run this command to add a satellite device:

```bash
hivemind-core add-client
```

The output shows these details:

- Node ID
- Friendly Name
- Access Key
- Password
- Encryption Key (deprecated, only used for legacy clients)

Enter these credentials on the client device to connect it.

## Running the HiveMind server

Start the HiveMind server to accept client connections on a chosen port:

```bash
hivemind-core listen --port 5678
```

The server now listens for incoming satellite connections.

> `hivemind-core` must run on the same device as OVOS.

## Permissions

HiveMind Core uses a flexible permissions system. Each client's permissions are configurable. By default:

- Only essential bus messages are allowed.
- Skills and intents are accessible, but you can blacklist or restrict them.

Manage permissions for a client with commands like `allow-msg`, `blacklist-msg`, `allow-skill`, and `blacklist-skill`.

Example use cases:

- Basic AI integration: let a simple client send natural language instructions.
- Custom permissions: restrict an IoT device so it only sends specific message types, such as `temperature.set`.

## HiveMind Core commands overview

These are the basic commands for managing clients and their permissions:

Add a new client:

```bash
hivemind-core add-client --name "satellite_1" --access-key "mykey123" --password "mypass"
```

List all registered clients:

```bash
hivemind-core list-clients
```

Start listening for client connections:

```bash
hivemind-core listen --port 5678
```

For detailed help on a command, use `--help` (for example, `hivemind-core add-client --help`).

---
[Home](index.md) · [Terminology →](02_terminology.md)
