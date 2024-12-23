# Quick Start Guide

This guide will help you get started quickly with the HiveMind platform, allowing you to extend your OpenVoiceOS (OVOS) ecosystem across multiple devices, even with low-resource hardware. HiveMind lets you connect lightweight devices as satellites to a central OVOS hub, offering centralized control and fine-grained permissions.

![imagem](https://github.com/JarbasHiveMind/HiveMind-community-docs/assets/33701864/fb241c4d-ca84-4b47-b917-b398b16f93bd)

## 🚀 Installation

To begin using HiveMind Core, you need to install the `hivemind-core` package in your OVOS device. This can be done via pip:

```bash
pip install hivemind-core
```

## 🛰️ Adding a Satellite Device

Once the server is running, you'll need to add client credentials for each satellite device you want to connect.

Run the following command to add a satellite device:

```bash
hivemind-core add-client
```
   
The output wi*ll show you important details like:

- Node ID
- Friendly Name
- Access Key
- Password
- Encryption Key (deprecated, only used for legacy clients)

Provide these credentials on the client devices to enable the connection.

## 🖥️ Running the HiveMind Server

Start the HiveMind server to accept client connections on a specified port:

```bash
hivemind-core listen --port 5678
```

The server will now listen for incoming satellite connections.

> 💡 `hivemind-core` needs to be running in the same device as OVOS

## 🔑 Permissions

HiveMind Core uses a flexible permissions system, where each client's permissions are customizable. By default:
 
- Only essential bus messages are allowed.

- Skills and intents are accessible but can be blacklisted or restricted.

You can manage permissions for clients by using commands like `allow-msg`, `blacklist-msg`, `allow-skill`, and `blacklist-skill`.

### Example Use Cases:

- **Basic AI Integration**: Enable a simple client to send natural language instructions.
- **Custom Permissions**: Restrict an IoT device to only communicate with specific message types, such as `temperature.set`.

## HiveMind Core Commands Overview

Here are the basic commands for managing clients and their permissions:

- **Add a new client**:  

```bash
hivemind-core add-client --name "satellite_1" --access-key "mykey123" --password "mypass"
```

- **List all registered clients**:  

```bash
hivemind-core list-clients
```

- **Start listening for client connections**:  

```bash
hivemind-core listen --port 5678
```

For detailed help on each command, use `--help` (e.g., `hivemind-core add-client --help`).