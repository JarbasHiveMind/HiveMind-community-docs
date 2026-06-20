# CLI Reference

## hivemind-core

Server-side management commands for `hivemind-core`.

```
Usage: hivemind-core [OPTIONS] COMMAND [ARGS]...

Commands:
  print-config         Print HiveMind server configuration
  listen               Start accepting satellite connections
  add-client           Add credentials for a new client
  delete-client        Remove credentials for a client
  list-clients         List clients and credentials
  export-clients       Export clients and credentials to a CSV file
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
  policy               Inspect the policy admission chain
```

> `NODE_ID` (and `MSG_TYPE` / `SKILL_ID` / `INTENT_ID`) are **positional** arguments, not options. If you omit `NODE_ID`, the command prompts interactively with a list of clients.

### print-config

```
Usage: hivemind-core print-config
```

Prints the effective server configuration loaded from `~/.config/hivemind-core/server.json`.

### listen

```
Usage: hivemind-core listen
```

`listen` takes no options — all configuration comes from `~/.config/hivemind-core/server.json`. Edit that file (see the [Configuration Reference](config.md)) to change ports, database backend, agent/binary protocols, and the policy chain.

### add-client

```
Usage: hivemind-core add-client [OPTIONS]

Options:
  --name TEXT        Friendly name for the client
  --access-key TEXT  Custom access key (generated if omitted)
  --password TEXT    Custom password (generated if omitted)
  --crypto-key TEXT  Optional pre-shared encryption key
  --admin BOOLEAN    Grant admin powers to the client (default: false)
  --metadata TEXT    Client metadata as a JSON object
```

The node ID is auto-assigned; there is no `--node-id` option. Output includes the `Access Key`, `Password`, and deprecated `Encryption Key`.

### list-clients

Lists all registered clients with their node IDs, names, access keys, and permission summary.

### export-clients

```
Usage: hivemind-core export-clients [OPTIONS]

Options:
  --path TEXT  Output CSV file path
```

### delete-client

```
Usage: hivemind-core delete-client [NODE_ID]
```

`NODE_ID` is a positional argument; omit it to pick from a prompt.

### allow-msg / blacklist-msg

```
Usage: hivemind-core allow-msg MSG_TYPE [NODE_ID]
Usage: hivemind-core blacklist-msg MSG_TYPE [NODE_ID]
```

`MSG_TYPE` and `NODE_ID` are positional arguments; omit `NODE_ID` to pick from a prompt.

### blacklist-skill / allow-skill

```
Usage: hivemind-core blacklist-skill SKILL_ID [NODE_ID]
Usage: hivemind-core allow-skill SKILL_ID [NODE_ID]
```

`SKILL_ID` and `NODE_ID` are positional arguments. These are OVOS-policy-specific and require `OVOSAgentPolicy` in the server's `policy.chain`.

### blacklist-intent / allow-intent

```
Usage: hivemind-core blacklist-intent INTENT_ID [NODE_ID]
Usage: hivemind-core allow-intent INTENT_ID [NODE_ID]
```

`INTENT_ID` and `NODE_ID` are positional arguments. These are OVOS-policy-specific and require `OVOSAgentPolicy` in the server's `policy.chain`.

### make-admin / revoke-admin

```
Usage: hivemind-core make-admin [NODE_ID]
Usage: hivemind-core revoke-admin [NODE_ID]
```

`NODE_ID` is a positional argument; omit it to pick from a prompt.

### set-metadata

```
Usage: hivemind-core set-metadata [OPTIONS] [NODE_ID]

Options:
  --metadata TEXT  Metadata to merge, as a JSON object
  --key TEXT       Metadata key to set (requires --value)
  --value TEXT     Value for --key (parsed as JSON when possible)
  --unset TEXT     Remove a metadata key
```

`NODE_ID` is positional; omit it to pick from a prompt. Pass at least one of `--metadata`, `--key`/`--value`, or `--unset`. Metadata is read by policy plugins (e.g. `OVOSAgentPolicy`) to make per-client decisions.

### migrate-db

```
Usage: hivemind-core migrate-db [OPTIONS]

Options:
  --from MODULE  Source database backend module (default: hivemind-json-db-plugin)
  --to MODULE    Target database backend module (default: hivemind-sqlite-db-plugin)
```

`--from` and `--to` take full plugin module names (e.g. `hivemind-json-db-plugin`, `hivemind-sqlite-db-plugin`, `hivemind-redis-db-plugin`).

### policy

```
Usage: hivemind-core policy COMMAND [ARGS]...

Commands:
  list  Print the loaded policy chain
  test  Dry-run a message through the chain
```

```
Usage: hivemind-core policy list
Usage: hivemind-core policy test API_KEY MSG_TYPE
```

`policy list` prints the active policy chain (`MessageTypeACLPolicy` first, then plugins from `policy.chain`). `policy test` looks up the client by `API_KEY` and runs a fake message of `MSG_TYPE` through the full chain, printing the verdict.

---

## hivemind-client

Satellite-side commands provided by `hivemind-websocket-client`.

```
Usage: hivemind-client [OPTIONS] COMMAND [ARGS]...

Commands:
  set-identity       Write connection credentials to the identity file
  terminal           Interactive CLI: inject utterances and print speech
  test-identity      Test connection using the identity file
  reset-pgp          Recreate the RSA key pair for inter-node communication
  send-mycroft       Send a raw OVOS message to the hub
  escalate           Send an OVOS message wrapped in ESCALATE
  propagate          Send an OVOS message wrapped in PROPAGATE
  ping               Flood-ping the mesh and print the responding topology
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

### terminal

```
Usage: hivemind-client terminal [OPTIONS]

Options:
  --key TEXT       HiveMind access key (default: from identity file)
  --password TEXT  HiveMind password (default: from identity file)
  --host TEXT      HiveMind host (default: from identity file)
  --port INTEGER   HiveMind port (default: 5678)
  --siteid TEXT    Site identifier
```

Interactive CLI: type utterances to inject and the hub's speech is printed back.

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

### ping

```
Usage: hivemind-client ping [OPTIONS]

Options:
  --key TEXT       HiveMind access key (default: from identity file)
  --password TEXT  HiveMind password (default: from identity file)
  --host TEXT      HiveMind host (default: from identity file)
  --port INTEGER   HiveMind port (default: 5678)
  --siteid TEXT    Site identifier
  --timeout FLOAT  Seconds to collect responses (default: 5.0)
  --json           Output raw JSON topology
```

Flood-pings the mesh and prints the peers that respond.

### reset-pgp

Recreates the private RSA key used for inter-node communication. (The command name is `reset-pgp` for historical reasons, but it generates an RSA key pair.)

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
  --zeroconf BOOLEAN   Advertise via mDNS/Zeroconf (default: True)
  --upnp BOOLEAN       Advertise via UPnP (default: False)
  --ssl BOOLEAN        Report SSL support (default: False)
```

### scan

```
Options:
  --zeroconf BOOLEAN   Scan via mDNS/Zeroconf (default: True)
  --upnp BOOLEAN       Scan via UPnP (default: False)
  --service-type TEXT  Service type (default: HiveMind-websocket)
```
