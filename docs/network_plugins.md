
# Network Plugins for HiveMind

HiveMind uses **network plugins** to decide *how nodes communicate with each other*.
While **agent plugins** define *what to do* with messages, network plugins define *how those messages travel*.

This makes HiveMind adaptable: depending on your environment, you can switch between different network transports without changing your agents.

---

## What is a Network Plugin?

A **Network Plugin** implements the **transport layer** for HiveMind.
It determines how BUS messages are sent and received between nodes (e.g., hubs, satellites, peers).

Each HiveMind node loads one network plugin, which defines how it connects to the Hive.

---

## Available Network Plugins

### 🔹 WebSocket Plugin (Default)

* The **default network plugin** for HiveMind.
* Uses persistent **WebSocket connections** between nodes.
* Well-suited for:
  * Satellite ↔ Hub communication
  * Local networks and internet deployments
  * Low-latency, bidirectional messaging
* Recommended for most users.

---

### 🔹 HTTP Plugin

* Provides a **stateless, request–response** transport layer.
* Useful in environments where:
  * WebSockets are blocked or not available
  * A simpler one-off communication model is sufficient
* Less efficient for real-time streaming but useful for constrained networks or integrations.

---

## Why Network Plugins Matter

* They make HiveMind **flexible** across environments.
* They **abstract away the transport layer** so agents don’t care how messages travel.
* They enable HiveMind to run anywhere from:
  * A LAN with WebSockets
  * To cloud-relayed HTTP setups
  * To potential future transports (MQTT, mesh, etc.).


HiveMind will continue to expand its transport options over time to support even more diverse networking environments.

## MQTT Plugin

Entry point: `hivemind-mqtt-plugin` (package `hivemind-mqtt-protocol`). Not yet on
PyPI — install from git.

Transports HiveMind messages over an MQTT broker, for deployments that already run
one. Configure it as a key under `network_protocol` in `server.json`, the same way as
the websocket and HTTP plugins.
