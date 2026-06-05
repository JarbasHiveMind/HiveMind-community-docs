# Quick Start Guide

This guide takes you from zero to a working hub with one connected satellite.

> **Hub Requirements**: Choose one:
> - **OVOS Hub**: OVOS must already be running (`ovos-core` and `ovos-messagebus`). See the [OVOS documentation](https://openvoiceos.github.io/community-docs) for setup.
> - **Persona Hub**: Use `hivemind-persona` for LLM/chatbot mode without OVOS. See [Persona Server](08_persona.md) for setup.

---

## Step 1 — Install HiveMind Core on the hub

On the hub machine:

```bash
pip install hivemind-core
```

> **Note**: Use your preferred package installer (`pip`, `uv pip`, etc.).

## Step 2 — Start the HiveMind server

```bash
hivemind-core listen --port 5678
```

The server now accepts satellite connections on port 5678.

```bash
# Full options
hivemind-core listen --help

Options:
  --host TEXT        HiveMind host
  --port INTEGER     HiveMind port number (default: 5678)
  --ssl BOOLEAN      Use wss:// instead of ws://
  --cert_dir TEXT    SSL certificate directory
  --cert_name TEXT   SSL certificate file name
  --db-backend [redis|json|sqlite]
                     Database backend (default: sqlite)
  --db-name TEXT     [json/sqlite] Database file name
  --db-folder TEXT   [json/sqlite] Database folder
  --redis-host TEXT  [redis] Redis host (default: localhost)
  --redis-port INT   [redis] Redis port (default: 6379)
  --redis-password TEXT  [redis] Redis password
```

> See [Database Backends](17_database.md) for guidance on choosing `sqlite`, `json`, or `redis`.

## Step 3 — Add credentials for a satellite

On the hub, register the satellite device:

```bash
hivemind-core add-client --name "my-satellite"
```

The output shows the credentials the satellite will need:

```
Credentials added to database!

Node ID: 2
Friendly Name: my-satellite
Access Key: 5a9e580a2773a262cbb23fe9759881ff
Password: 9b247ca66c7cd2b6388ad49ca504279d
Encryption Key: 4185240103de0770
WARNING: Encryption Key is deprecated, only use if your client does not support password
```

Keep the **Access Key** and **Password** — you need them on the satellite.

> **Tip**: use `hivemind-core add-client --help` to set a custom access key and password instead of generated ones.

## Step 4 — Install and configure the satellite

On the satellite device, choose and install a satellite package. For a full-featured voice satellite:

```bash
sudo apt-get install -y libpulse-dev libasound2-dev
pip install HiveMind-voice-sat  # or: uv pip install HiveMind-voice-sat
```

> Not sure which satellite to use? See [Choosing a Satellite](satellite_comparison.md).

Set the identity file on the satellite with the credentials from Step 3:

```bash
hivemind-client set-identity \
  --key 5a9e580a2773a262cbb23fe9759881ff \
  --password 9b247ca66c7cd2b6388ad49ca504279d \
  --host 192.168.1.10 \
  --port 5678 \
  --siteid living-room
```

This writes `~/.config/hivemind/_identity.json`.

## Step 5 — Test the connection

```bash
hivemind-client test-identity
```

Expected output:

```
== Identity successfully connected to HiveMind!
```

## Step 6 — Start the satellite

```bash
hivemind-voice-sat
```

The satellite reads credentials from the identity file automatically. You can also pass them on the command line:

```bash
hivemind-voice-sat --host 192.168.1.10 --key <key> --password <pass>
```

---

## Managing clients

```bash
# List all registered clients
hivemind-core list-clients

# Remove a client
hivemind-core delete-client --node-id 2

# Allow a specific OVOS message type for a client
hivemind-core allow-msg "speak" --node-id 2
```

See [Permissions](16_permissions.md) for the full permission management reference.

---

## Permissions overview

By default each new client has access to basic message types only. You can expand or restrict permissions per client:

- Only essential bus messages are allowed by default
- Skills and intents are accessible but individual ones can be blacklisted
- Message types, skills, and intents can each be allowed or denied independently

```bash
# Allow a message type
hivemind-core allow-msg "mycroft.volume.set" --node-id 2

# Blacklist a skill
hivemind-core blacklist-skill "skill-homeassistant.openvoiceos" --node-id 2
```

See [Permissions](16_permissions.md) for the complete CLI reference.
