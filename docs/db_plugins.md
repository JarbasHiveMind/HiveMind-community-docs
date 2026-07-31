# Database Plugins for HiveMind

HiveMind uses database plugins to provide persistent storage for client credentials, permissions, and other node state.

Network and agent plugins handle message transport and behavior. Database plugins let HiveMind nodes remember clients, authorization, and configuration across restarts.

---

## What is a Database Plugin?

A Database Plugin defines how and where HiveMind stores persistent data, including:

- client credentials, such as API keys and certificates.
- permissions and access control.

Each node loads a single database plugin, which acts as the persistent backend for that HiveMind instance.

---

## Available database plugins

### JSON Plugin (default)

- Provided natively by the [`json-database`](https://github.com/TigreGotico/json_database) package.
- Stores all data as local JSON files.
- Simple to configure.
- Suitable for testing, small deployments, or single-node setups.
- Limitations: not ideal for multi-node environments, and lacks advanced caching and scaling.

### Redis Plugin (recommended for production)

- Provided by the [`hivemind-redis-database`](https://github.com/JarbasHiveMind/hivemind-redis-database) package.
- Uses Redis as the backend.
- Provides fast, in-memory storage with optional persistence.
- Suited for multi-node HiveMind deployments, where all nodes share credentials and permissions.
- Supports high-throughput scenarios and distributed systems.

### SQLite Plugin (experimental)

- Provided by the [`hivemind-sqlite-database`](https://github.com/JarbasHiveMind/hivemind-sqlite-database) package.
- Stores data in a local SQLite database instead of JSON files.
- Aims to provide a lightweight, transactional alternative to JSON for single-node setups.
- Currently experimental, suitable for testing or small-scale deployments.

---

## Why database plugins matter

- **Security**: store client credentials and permissions safely.
- **Persistence**: keep state across node restarts.
- **Scalability**: Redis or similar backends let HiveMind operate in multi-node or cloud environments.

---

## Recommendations

- Use the JSON plugin for local testing or single-node setups.
- Use the SQLite plugin as a lightweight, experimental alternative to JSON.
- Use the Redis plugin for production environments, multi-node networks, or setups that need fast, shared access to client credentials.

---
[← Network](network_plugins.md) · [Home](index.md) · [Satellites →](satellites.md)
