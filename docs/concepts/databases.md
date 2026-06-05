# Database Backends

`hivemind-core` stores client credentials and settings in a pluggable database backend.

| Backend | Default location | Best for |
|---|---|---|
| **SQLite** | `~/.local/share/hivemind-core/clients.db` | New installations (default) |
| **JSON** | `~/.local/share/hivemind-core/clients.json` | Existing installs; manual inspection |
| **Redis** | `localhost:6379` | Distributed or multi-instance deployments |

## Choosing a backend

**SQLite** is the default for new installations. It is transactional, concurrency-safe, and requires no additional infrastructure — just the Python standard library.

**JSON** is kept for installations that already have a `clients.json` file. It is human-readable and easy to inspect or back up manually, but it is not safe under concurrent writes.

**Redis** is appropriate when you need to share client state across multiple `hivemind-core` processes (multi-instance deployments) or when you want high-throughput credential lookups.

## Migrating between backends

```bash
hivemind-core migrate-db --to sqlite
```

Migrate from the current backend (detected automatically) to SQLite. Replace `sqlite` with `json` or `redis` as needed.

## Selecting a backend at launch

```bash
# SQLite (default)
hivemind-core listen --db-backend sqlite

# JSON
hivemind-core listen --db-backend json

# Redis
hivemind-core listen \
  --db-backend redis \
  --redis-host 192.168.1.10 \
  --redis-port 6379 \
  --redis-password myredispassword
```

> Use the same `--db-backend` flags for both `listen` and `add-client` (and all other client-management commands). If they differ, the commands will look in different databases and clients will not be found.

## Security notes

**SQLite and JSON**: The database file lives on disk. Restrict file permissions to the user running `hivemind-core`. Back up and encrypt backup copies.

**Redis**: Configure Redis authentication (`requirepass` in `redis.conf`). Use TLS between `hivemind-core` and Redis in untrusted network environments. Bind Redis to `127.0.0.1` or a trusted interface; do not expose it to the internet.

General:

- Audit database access logs periodically
- Store backup files encrypted
- Monitor for unexpected access patterns
