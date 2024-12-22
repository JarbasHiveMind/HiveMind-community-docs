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
