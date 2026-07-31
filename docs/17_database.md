# Database Backends

`hivemind-core` supports multiple database backends to store client credentials and settings. Each fits a different use case:

| Backend            | Use Case                                       | Default Location                            | Command Line options                               |
|--------------------|------------------------------------------------|---------------------------------------------|------------------------------------------------------|
| **JSON** (default) | Simple, file-based setup for local use         | `~/.local/share/hivemind-core/clients.json` | Configurable via `--db-name` and `--db-folder`     |
| **SQLite**         | Lightweight relational DB for single instances | `~/.local/share/hivemind-core/clients.db`   | Configurable via `--db-name` and `--db-folder`     |
| **Redis**          | Distributed, high-performance environments     | `localhost:6379`                            | Configurable via `--redis-host` and `--redis-port` |

> Use the same database parameters when launching `hivemind-core` and registering clients.

**How to choose**:

- For scalability or multi-instance setups, use Redis.
- For simplicity or single-device environments, use SQLite.
- For development, or to edit the database by hand, use JSON.

## Security considerations

Follow security practices for each backend to protect sensitive data.

#### 1. JSON (file-based storage)

- **Security risks**: JSON files sit on local disk, so anyone with file system access can read them. Without encryption, the data is vulnerable to unauthorized access.
- **Best practices**:
    - **File permissions**: restrict the `.json` file to the user running `hivemind-core`.
    - **Backups**: back up this file regularly, and encrypt the backups too.

#### 2. SQLite (lightweight relational database)

- **Security risks**: SQLite databases live in a file, so they are vulnerable if file permissions are not set correctly.
- **Best practices**:
  - **File permissions**: restrict the SQLite file to the user or group running `hivemind-core`.
  - **Database backups**: back up SQLite files securely, and store the backups encrypted.

#### 3. Redis (distributed, high-performance)

- **Security risks**: Redis is common in distributed setups, which adds risk if the Redis server is exposed to the internet or a local network without proper security.
- **Best practices**:
  - **Authentication**: set a strong Redis password with the `requirepass` directive in the Redis configuration file.
  - **Encryption**: use TLS/SSL encryption (the `--ssl` flag) for data in transit.
  - **Access control**: limit Redis access to trusted clients and IP addresses through the `bind` and `protected-mode` settings.
  - **Firewall**: restrict access to Redis from unauthorized networks with a firewall.
  - **Backups**: Redis does not encrypt persistent storage by default, so store backup files (RDB/AOF) securely and encrypt them if needed.

#### General database security tips

- **Sensitive data storage**: store sensitive data, including database backups, encrypted.
- **Regular audits**: periodically audit database access logs and configuration for unauthorized access.
- **Monitoring**: monitor your database systems for unusual access patterns or unauthorized connection attempts.

Following these practices keeps your `hivemind-core` installation secure, and keeps client credentials and settings protected.

---
[Home](index.md)
