# Database Backends

`hivemind-core` stores client credentials and settings in a pluggable database backend.

| Backend | Module (package) | Default location | Best for |
|---|---|---|---|
| **SQLite** | `hivemind-sqlite-db-plugin` (`hivemind-sqlite-database`) | XDG data dir, e.g. `~/.local/share/hivemind-core/clients.db` | New installations (default) |
| **JSON** | `hivemind-json-db-plugin` | XDG data dir, e.g. `~/.local/share/hivemind-core/clients.json` | Existing installs; manual inspection |
| **Redis** | `hivemind-redis-db-plugin` (`hivemind-redis-database`) | `127.0.0.1:6379` | Distributed or multi-instance deployments |

## Choosing a backend

**SQLite** is the default for new installations. It is transactional, concurrency-safe, and requires no additional infrastructure — just the Python standard library.

**JSON** is kept for installations that already have a `clients.json` file. It is human-readable and easy to inspect or back up manually, but it is not safe under concurrent writes.

**Redis** is appropriate when you need to share client state across multiple `hivemind-core` processes (multi-instance deployments) or when you want high-throughput credential lookups.

## Migrating between backends

`migrate-db` takes the full plugin module names for source and target. `--from` defaults to the JSON backend; `--to` defaults to SQLite. There is no auto-detection.

```bash
hivemind-core migrate-db \
  --from hivemind-json-db-plugin \
  --to hivemind-sqlite-db-plugin
```

Records are copied with their full credentials and metadata; the source database is left untouched.

## Selecting a backend

Backend selection is **not** a command-line flag — `hivemind-core listen` takes no arguments. The backend is chosen in `~/.config/hivemind-core/server.json` under the `database` block:

```json
{
  "database": {
    "module": "hivemind-redis-db-plugin",
    "hivemind-redis-db-plugin": {
      "host": "127.0.0.1",
      "port": 6379,
      "password": null,
      "username": null,
      "db": 0
    }
  }
}
```

Set `module` to the desired plugin (`hivemind-sqlite-db-plugin`, `hivemind-json-db-plugin`, or `hivemind-redis-db-plugin`) and provide a same-named sub-block with that backend's connection settings. The same configuration is read by `listen` and by every client-management command (`add-client`, `allow-msg`, …), so they always operate on the same database.

## Security notes

**SQLite and JSON**: The database file lives on disk. Restrict file permissions to the user running `hivemind-core`. Back up and encrypt backup copies. The SQLite backend additionally supports SQLCipher AES-256 encryption at rest — set a `password` key in its config sub-block.

**Redis**: Configure Redis authentication (`requirepass` in `redis.conf`, mirrored by the `password` key in the config sub-block). The Redis backend supports TLS to `hivemind-core`; use it in untrusted network environments. Bind Redis to `127.0.0.1` or a trusted interface; do not expose it to the internet.

General:

- Audit database access logs periodically
- Store backup files encrypted
- Monitor for unexpected access patterns
