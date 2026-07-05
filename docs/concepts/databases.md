# Database Backends

Somewhere, hivemind-core has to remember the guest list: which devices are allowed to
knock, what key each one holds, and what each is permitted to say once inside. For a
home setup that's a single file sitting quietly on disk — nothing to install, nothing to
run. Scale up to several servers sharing one guest list and that file becomes a shared
Redis instead. Same information either way; only *where it lives* changes, and swapping
that out never touches the rest of your setup.

!!! abstract "In a nutshell"
    - Three places to keep the guest list: **SQLite** (the default — a transactional file, nothing extra to run), **JSON** (the same idea but human-readable, and not safe if two processes write at once), and **Redis** (a shared store for when several servers need one list).
    - You pick the backend in the `database` block of `~/.config/hivemind-core/server.json` — never a command-line flag. `listen` and every client command read that same file, so they always agree on where the list lives.
    - Move between them with `migrate-db`, and lock either file backend at rest with a `password` key.

| Backend | Module (package) | Default location | Best for |
|---|---|---|---|
| **SQLite** | `hivemind-sqlite-db-plugin` (`hivemind-sqlite-database`) | XDG data dir, e.g. `~/.local/share/hivemind-core/clients.db` | New installations (default) |
| **JSON** | `hivemind-json-db-plugin` | XDG data dir, e.g. `~/.local/share/hivemind-core/clients.json` | Existing installs; manual inspection |
| **Redis** | `hivemind-redis-db-plugin` (`hivemind-redis-database`) | `127.0.0.1:6379` | Distributed or multi-instance deployments |

---

## Choosing a backend

**SQLite** is the default for new installations. It is transactional, concurrency-safe, and requires no additional infrastructure — just the Python standard library.

**JSON** is kept for installations that already have a `clients.json` file. It is human-readable and easy to inspect or back up manually, but it is not safe under concurrent writes.

**Redis** is appropriate when you need to share client state across multiple `hivemind-core` processes (multi-instance deployments) or when you want high-throughput credential lookups.

??? note "Advanced: how the default backend is actually chosen"
    When no `database` block is configured, `_default_database()` in `HiveMind-core/hivemind_core/config.py` decides:

    - A **brand-new install** defaults to **SQLite** (`clients.db`).
    - **But** if a legacy `clients.json` exists and there is no `clients.db` yet, it auto-keeps the **JSON** backend — so upgrading an older deployment never silently strands existing credentials.

    Relatedly, `migrate-db` defaults to `--from json --to sqlite`, matching the most common upgrade path.

---

## Migrating between backends

`migrate-db` takes the full plugin module names for source and target. `--from` defaults to the JSON backend; `--to` defaults to SQLite. There is no auto-detection.

```bash
hivemind-core migrate-db \
  --from hivemind-json-db-plugin \
  --to hivemind-sqlite-db-plugin
```

Records are copied with their full credentials and metadata; the source database is left untouched.

---

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

---

## Security notes

**SQLite and JSON**: The database file lives on disk. Restrict file permissions to the user running `hivemind-core`. Back up and encrypt backup copies. **Both file backends support encryption at rest** by setting a `password` key in their config sub-block — SQLite uses SQLCipher AES-256, and JSON switches to `EncryptedJsonStorageXDG`. (JSON remains unsafe under concurrent writes regardless of encryption — encryption protects the file at rest, not concurrent access.)

**Redis**: Configure Redis authentication (`requirepass` in `redis.conf`, mirrored by the `password` key in the config sub-block). Bind Redis to `127.0.0.1` or a trusted interface; do not expose it to the internet.

??? note "Advanced: full Redis connection and TLS keys"
    The Redis plugin's config sub-block accepts more than just `password`:

    | Key | Purpose |
    |---|---|
    | `username` | Redis ACL username (default `"default"`) |
    | `password` | Redis auth password |
    | `use_ssl` | Enable TLS to Redis |
    | `ssl_certfile` | Client certificate path |
    | `ssl_keyfile` | Client private key path |
    | `ssl_ca_certs` | CA bundle path |
    | `ssl_cert_reqs` | `"required"` / `"optional"` / `"none"` (default `"required"`) |
    | `ssl_check_hostname` | Verify the server hostname (default `True`) |

    Use TLS in untrusted network environments.

General:

- Audit database access logs periodically
- Store backup files encrypted
- Monitor for unexpected access patterns

---

## Source

Validated against the HiveMind source:

- [`hivemind_core/config.py`](https://github.com/JarbasHiveMind/HiveMind-core/blob/HEAD/hivemind_core/config.py) — `_default_database()` SQLite/JSON auto-selection and `migrate-db` defaults
- [`hivemind_sqlite_database/__init__.py`](https://github.com/JarbasHiveMind/hivemind-sqlite-database/blob/HEAD/hivemind_sqlite_database/__init__.py) — SQLCipher AES-256 encryption via `password`
- [`hivemind_redis_database/__init__.py`](https://github.com/JarbasHiveMind/hivemind-redis-database/blob/HEAD/hivemind_redis_database/__init__.py) — `username` and the full `ssl_*` connection keys
- [`hivemind_json_database/__init__.py`](https://github.com/JarbasHiveMind/hivemind-json-db-plugin/blob/HEAD/hivemind_json_database/__init__.py) — `EncryptedJsonStorageXDG` when `password` is set
