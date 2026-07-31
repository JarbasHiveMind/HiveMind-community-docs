# Network Plugins for HiveMind

HiveMind uses network plugins to decide how nodes communicate with each other. Agent plugins define what to do with a message. Network plugins define how that message travels.

This design makes HiveMind adaptable: depending on your environment, you can switch between network transports without changing your agents.

---

## What is a Network Plugin?

A Network Plugin implements the transport layer for HiveMind. It determines how BUS messages travel between nodes, such as hubs, satellites, and peers.

Each HiveMind node loads one network plugin, which defines how it connects to the hive.

---

## Available network plugins

### WebSocket Plugin (default)

- The default network plugin for HiveMind.
- Uses persistent WebSocket connections between nodes.
- Well suited for:
  - satellite-to-hub communication.
  - local networks and internet deployments.
  - low-latency, bidirectional messaging.
- Recommended for most users.

### HTTP Plugin

- Provides a stateless, request-response transport layer.
- Useful when:
  - WebSockets are blocked or unavailable.
  - a simpler one-off communication model is enough.
- Less efficient for real-time streaming, but useful for constrained networks or integrations.

---

## Why network plugins matter

- They make HiveMind flexible across environments.
- They abstract away the transport layer, so agents do not need to care how messages travel.
- They let HiveMind run anywhere, from a LAN with WebSockets to a cloud-relayed HTTP setup, and to future transports such as MQTT or mesh networks.

HiveMind will continue to add transport options to support more networking environments.

---
[← Binary](binary_plugins.md) · [Home](index.md) · [Database →](db_plugins.md)
