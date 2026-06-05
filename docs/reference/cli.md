# CLI Reference

## hivemind-core

Server-side management commands for `hivemind-core`.

```
Usage: hivemind-core [OPTIONS] COMMAND [ARGS]...

Commands:
  listen               Start accepting satellite connections
  add-client           Add credentials for a new client
  delete-client        Remove credentials for a client
  list-clients         List clients and credentials
  rename-client        Rename a client in the database
  allow-msg            Allow a message type for a client
  blacklist-msg        Deny a message type for a client
  allow-escalate       Allow ESCALATE messages from a client
  blacklist-escalate   Deny ESCALATE messages from a client
  allow-propagate      Allow PROPAGATE messages from a client
  blacklist-propagate  Deny PROPAGATE messages from a client
  allow-skill          Remove a skill from a client's blacklist
  blacklist-skill      Blacklist a skill for a client
  allow-intent         Remove an intent from a client's blacklist
  blacklist-intent     Blacklist an intent for a client
  make-admin           Grant admin flag to a client
  revoke-admin         Revoke admin flag from a client
  set-metadata         Set arbitrary metadata on a client
  migrate-db           Migrate the database to a different backend
```

> If you do not specify `--node-id`, the command prompts interactively with a list of clients.

### listen

```
Usage: hivemind-core listen [OPTIONS]

Options:
  --host TEXT                      Listen host (default: 0.0.0.0)
  --port INTEGER                   WebSocket port (default: 5678)
  --ssl BOOLEAN                    Use wss:// (default: false)
  --cert_dir TEXT                  SSL certificate directory
  --cert_name TEXT                 SSL certificate file name
  --db-backend [redis|json|sqlite] Database backend (default: sqlite)
  --db-name TEXT                   [json/sqlite] Database file name
  --db-folder TEXT                 [json/sqlite] Database folder
  --redis-host TEXT                [redis] Redis host (default: localhost)
  --redis-port INTEGER             [redis] Redis port (default: 6379)
  --redis-password TEXT            [redis] Redis password
```

### add-client

```
Usage: hivemind-core add-client [OPTIONS]

Options:
  --name TEXT        Friendly name for the client
  --access-key TEXT  Custom access key (generated if omitted)
  --password TEXT    Custom password (generated if omitted)
  --node-id INTEGER  Explicit node ID (auto-assigned if omitted)
```

Output includes the `Access Key`, `Password`, and deprecated `Encryption Key`.

### list-clients

Lists all registered clients with their node IDs, names, access keys, and permission summary.

### delete-client

```
Usage: hivemind-core delete-client [OPTIONS]

Options:
  --node-id INTEGER  Node ID of the client to remove
```

### allow-msg / blacklist-msg

```
Usage: hivemind-core allow-msg [OPTIONS] MSG_TYPE

Options:
  --node-id INTEGER  Target client node ID

Usage: hivemind-core blacklist-msg [OPTIONS] MSG_TYPE

Options:
  --node-id INTEGER  Target client node ID
```

### blacklist-skill / allow-skill

```
Usage: hivemind-core blacklist-skill [OPTIONS] SKILL_ID

Options:
  --node-id INTEGER  Target client node ID
```

### blacklist-intent / allow-intent

```
Usage: hivemind-core blacklist-intent [OPTIONS] INTENT_NAME

Options:
  --node-id INTEGER  Target client node ID
```

### make-admin / revoke-admin

```
Usage: hivemind-core make-admin [OPTIONS]

Options:
  --node-id INTEGER  Target client node ID
```

### set-metadata

```
Usage: hivemind-core set-metadata [OPTIONS]

Options:
  --node-id INTEGER  Target client node ID
  --key TEXT         Metadata key
  --value TEXT       Metadata value
```

Metadata is read by policy plugins (e.g. `OVOSAgentPolicy`) to make per-client decisions.

### migrate-db

```
Usage: hivemind-core migrate-db [OPTIONS]

Options:
  --to [sqlite|json|redis]  Target backend
```

---

## hivemind-client

Satellite-side commands provided by `hivemind-websocket-client`.

```
Usage: hivemind-client [OPTIONS] COMMAND [ARGS]...

Commands:
  set-identity       Write connection credentials to the identity file
  test-identity      Test connection using the identity file
  reset-pgp          Generate a new PGP key pair
  send-utterance     Send a natural-language utterance to the hub
  send-mycroft       Send a raw OVOS message to the hub
  escalate           Send an OVOS message wrapped in ESCALATE
  propagate          Send an OVOS message wrapped in PROPAGATE
```

### set-identity

```
Usage: hivemind-client set-identity [OPTIONS]

Options:
  --key TEXT       Access key
  --password TEXT  Password
  --host TEXT      Hub host (ws:// or wss://)
  --port INTEGER   Hub port (default: 5678)
  --siteid TEXT    Site identifier for context routing
```

Writes `~/.config/hivemind/_identity.json`.

### send-utterance

```
Usage: hivemind-client send-utterance [OPTIONS] UTTERANCE

Options:
  --key TEXT       HiveMind access key (default: from identity file)
  --password TEXT  HiveMind password (default: from identity file)
  --host TEXT      HiveMind host (default: from identity file)
  --port INTEGER   HiveMind port (default: 5678)
  --siteid TEXT    Site identifier
```

### send-mycroft

```
Usage: hivemind-client send-mycroft [OPTIONS]

Options:
  --msg TEXT       OVOS message type
  --payload TEXT   OVOS message data (JSON string)
  --key TEXT       Access key
  --password TEXT  Password
  --host TEXT      Hub host
  --port INTEGER   Port (default: 5678)
  --siteid TEXT    Site identifier
```

---

## hivemind-presence

Discovery commands from the `hivemind-presence` package.

```
Usage: hivemind-presence [OPTIONS] COMMAND [ARGS]...

Commands:
  announce  Advertise this node on the local network
  scan      Scan for HiveMind nodes on the local network
```

### announce

```
Options:
  --port INTEGER       HiveMind port number (default: 5678)
  --name TEXT          Friendly device name (default: HiveMind-Node)
  --service-type TEXT  Service type (default: HiveMind-websocket)
  --beacon BOOLEAN     Advertise via UDP broadcast (default: True)
  --zeroconf BOOLEAN   Advertise via mDNS/Zeroconf (default: False)
```

### scan

```
Options:
  --beacon BOOLEAN     Scan via UDP broadcast (default: True)
  --zeroconf BOOLEAN   Scan via mDNS/Zeroconf (default: False)
  --service-type TEXT  Service type (default: HiveMind-websocket)
```
