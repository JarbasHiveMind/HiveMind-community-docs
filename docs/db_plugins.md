# Database Plugins for HiveMind

HiveMind uses **database plugins** to provide **persistent storage** for client credentials, permissions, and other critical node state.

While network and agent plugins handle message transport and behavior, database plugins ensure **HiveMind nodes can remember clients, authorization, and configuration across restarts**.

---

## What is a Database Plugin?

A **Database Plugin** defines **how and where HiveMind stores persistent data**.
This includes:

* Client credentials (API keys, certificates)
* Permissions and access control

Each node loads a single database plugin, which acts as the **persistent backend** for that HiveMind instance.

---

## Available Database Plugins

### 🔹 JSON Plugin (Default)

* Provided natively by the [`json-database`](https://github.com/TigreGotico/json_database) package.
* Stores all data as local **JSON files**.
* Simple and easy to configure.
* Suitable for testing, small deployments, or single-node setups.
* **Limitations:** Not ideal for multi-node environments, lacks advanced caching and scaling.

---

### 🔹 Redis Plugin (Recommended for Production)

* Provided by the [`hivemind-redis-database`](https://github.com/JarbasHiveMind/hivemind-redis-database) package.
* Uses **Redis** as the backend.
* Provides **fast, in-memory storage** with optional persistence.
* Ideal for **multi-node HiveMind deployments**, enabling all nodes to share credentials and permissions.
* Supports high-throughput scenarios and distributed systems.

---

### 🔹 SQLite Plugin (Experimental)

* Provided by the [`hivemind-sqlite-database`](https://github.com/JarbasHiveMind/hivemind-sqlite-database) package.
* Stores data in a **local SQLite database** instead of JSON files.
* Aims to provide a **lightweight, transactional alternative** to JSON for single-node setups.
* Currently **experimental** — suitable for testing or small-scale deployments.

---

## Why Database Plugins Matter

* **Security:** Store client credentials and permissions safely.
* **Persistence:** Keep state across node restarts.
* **Scalability:** Redis or similar backends allow HiveMind to operate in multi-node or cloud environments.

---

## Recommendations

* Use the **JSON plugin** for local testing or single-node setups.
* Use the **SQLite plugin** for lightweight, experimental alternative to JSON.
* Use the **Redis plugin** for production environments, multi-node networks, or setups requiring fast and shared access to client credentials.

