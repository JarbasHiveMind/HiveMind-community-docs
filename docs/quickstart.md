# Quick Start

This guide takes you from zero to a working hub with one connected satellite in under ten minutes.

> **Prerequisites for an OVOS hub**: `ovos-core` and `ovos-messagebus` must already be running on the hub machine. See the [OVOS documentation](https://openvoiceos.github.io/community-docs) for setup. For an LLM/chatbot hub without OVOS, use `hivemind-persona` — see [Persona Hub](server/persona-hub.md).

---

## Step 1 — Install HiveMind Core on the hub

```bash
pip install hivemind-core
```

## Step 2 — Start the server

```bash
hivemind-core listen --port 5678
```

The server accepts satellite connections on port 5678. Full options:

```
Options:
  --host TEXT                      HiveMind host (default: 0.0.0.0)
  --port INTEGER                   HiveMind port number (default: 5678)
  --ssl BOOLEAN                    Use wss:// instead of ws://
  --cert_dir TEXT                  SSL certificate directory
  --cert_name TEXT                 SSL certificate file name
  --db-backend [redis|json|sqlite] Database backend (default: sqlite)
  --db-name TEXT                   [json/sqlite] Database file name
  --db-folder TEXT                 [json/sqlite] Database folder
  --redis-host TEXT                [redis] Redis host (default: localhost)
  --redis-port INT                 [redis] Redis port (default: 6379)
  --redis-password TEXT            [redis] Redis password
```

Server configuration can also be written to `~/.config/hivemind-core/server.json` so you do not have to repeat flags on every launch. See [Server Configuration](reference/config.md).

## Step 3 — Register credentials for a satellite

On the hub machine, add a client entry:

```bash
hivemind-core add-client --name "my-satellite"
```

Output:

```
Credentials added to database!

Node ID: 2
Friendly Name: my-satellite
Access Key: 5a9e580a2773a262cbb23fe9759881ff
Password: 9b247ca66c7cd2b6388ad49ca504279d
Encryption Key: 4185240103de0770
WARNING: Encryption Key is deprecated, only use if your client does not support password
```

Note the **Access Key** and **Password** — you will need them on the satellite.

> Use `hivemind-core add-client --help` to supply your own key and password instead of generated ones.

## Step 4 — Install and configure the satellite

On the satellite device, choose a satellite package. For a full voice satellite:

```bash
# Linux audio dependencies
sudo apt-get install -y libpulse-dev libasound2-dev

pip install HiveMind-voice-sat
```

Not sure which satellite fits your hardware? See [Choosing a Satellite](satellites/index.md).

Write the identity file:

```bash
hivemind-client set-identity \
  --key 5a9e580a2773a262cbb23fe9759881ff \
  --password 9b247ca66c7cd2b6388ad49ca504279d \
  --host 192.168.1.10 \
  --port 5678 \
  --siteid living-room
```

This writes `~/.config/hivemind/_identity.json`. All satellite commands read this file by default.

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

Or pass credentials directly:

```bash
hivemind-voice-sat --host 192.168.1.10 --key <key> --password <password>
```

---

## Managing clients

```bash
# List all registered clients
hivemind-core list-clients

# Remove a client
hivemind-core delete-client --node-id 2

# Allow a specific OVOS message type
hivemind-core allow-msg "speak" --node-id 2

# Blacklist a skill
hivemind-core blacklist-skill "skill-homeassistant.openvoiceos" --node-id 2
```

For the full CLI reference see [CLI Reference](reference/cli.md) and [Permissions](concepts/security.md#permissions).

---

## Next steps

- [Choosing a Satellite](satellites/index.md) — compare all satellite options
- [Permissions](concepts/security.md#permissions) — restrict what each client can do
- [Auto Discovery](concepts/discovery.md) — find the hub automatically over the local network
- [Docker Deployment](server/docker.md) — run the hub in a container
- [Nested Hives](concepts/mesh.md#nested-hives) — connect hubs to other hubs
