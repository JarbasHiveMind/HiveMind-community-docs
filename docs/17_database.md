# Database Backends

`hivemind-core` supports multiple database backends to store client credentials and settings. Each has its own use case:

| Backend            | Use Case                                       | Default Location                            | Command Line options                               |
|--------------------|------------------------------------------------|---------------------------------------------|----------------------------------------------------|
| **JSON** (default) | Simple, file-based setup for local use         | `~/.local/share/hivemind-core/clients.json` | Configurable via `--db-name` and `--db-folder`     |
| **SQLite**         | Lightweight relational DB for single instances | `~/.local/share/hivemind-core/clients.db`   | Configurable via `--db-name` and `--db-folder`     |
| **Redis**          | Distributed, high-performance environments     | `localhost:6379`                            | Configurable via `--redis-host` and `--redis-port` |

**How to Choose?**

- For **scalability** or multi-instance setups, use Redis.
- For **simplicity** or single-device environments, use SQLite.
- For **development** or to be able to edit the database by hand, use JSON.

## Security Considerations

When using any of these backends, it’s important to implement security practices to safeguard sensitive data. Below are some considerations:

#### 1. **JSON (File-Based Storage)**
- **Security Risks**: As JSON files are stored locally, they can be accessed directly by anyone with access to the file system. Without encryption, the data is vulnerable to unauthorized access.
- **Best Practices**:
    - **File Permissions**: Set restrictive permissions on the `.json` file to limit access to the user running `hivemind-core`.
    - **Backups**: Regularly back up this file to ensure recovery in case of data loss or corruption, while also securing backups with encryption.

#### 2. **SQLite (Lightweight Relational Database)**
- **Security Risks**: SQLite databases are stored in a file, making them susceptible to unauthorized access if file permissions are not properly configured.
- **Best Practices**:
  - **File Permissions**: Ensure the SQLite file is owned by a specific user or group, with read and write access limited to only the user running `hivemind-core`.
    - **Database Backups**: Always back up SQLite files securely and store backups in encrypted form.
  
#### 3. **Redis (Distributed High-Performance)**
- **Security Risks**: Redis is commonly used in distributed setups, which can introduce risks if the Redis server is exposed to the internet or local networks without proper security measures.
- **Best Practices**:
  - **Authentication**: Always configure **Redis authentication** by setting a strong password using the `requirepass` directive in the Redis configuration file.
    - **Encryption**: Use **TLS/SSL** encryption (`--ssl` flag) for data in transit. This ensures that data is encrypted between clients and Redis servers.
    - **Access Control**: Limit access to Redis to trusted clients and IP addresses by configuring the `bind` and `protected-mode` settings in the Redis configuration file.
    - **Firewall**: Use a firewall to restrict access to Redis from unauthorized networks, ensuring that only trusted systems can communicate with the Redis server.
    - **Backups**: Redis does not encrypt its persistent storage by default, so ensure that backup files (RDB/AOF) are stored securely and encrypted if necessary.

#### General Database Security Tips:
- **Sensitive Data Storage**: Ensure that sensitive data, such as database backups, is stored securely (using encryption)
- **Regular Audits**: Periodically audit your database access logs and configurations to ensure no unauthorized access has occurred.
- **Monitoring**: Implement monitoring on your database systems to detect any unusual access patterns or unauthorized attempts to connect.

By following these best practices, you can ensure that your `hivemind-core` installation is secure and that client credentials and settings remain protected.
