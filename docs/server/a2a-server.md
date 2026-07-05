# A2A Server

Maybe you've already built an agent — a LangChain pipeline, a Google ADK bot, a CrewAI
crew, a hand-rolled FastAPI service — and you'd rather your satellites talk to *that*
than to OVOS skills. The A2A server is the bridge. hivemind-core forwards each question
to your external agent over Google's open **Agent-to-Agent** protocol and streams the
answer back to whoever asked. Your agent stays where it lives and keeps its own memory;
the hive just becomes a spread of microphones and screens in front of it.

!!! abstract "In a nutshell"
    - Swaps the `agent_protocol` for `hivemind-a2a-agent-plugin`, forwarding every query to any Google A2A-compliant server.
    - Only `agent_url` is required; `auth_header`, `timeout`, and `streaming` are optional.
    - HiveMind's `session_id` maps one-to-one to the A2A `sessionId`, so the back-and-forth of a conversation is remembered on the agent's side, not here.

[A2A](https://google.github.io/A2A/) is Google's open protocol for agent
interoperability. An A2A server publishes an **agent card** at
`GET /.well-known/agent.json` describing its skills and task URL, and accepts **task**
requests as JSON-RPC 2.0 — either a blocking `tasks/send` or a streaming
`tasks/sendSubscribe` (SSE). Any compliant A2A backend (LangChain agents, Google ADK,
CrewAI, a custom FastAPI service, …) works behind this plugin.

Satellites built for `hivemind-core` (text-based ones like HiveMind-cli, voice-sat, or
bridges) work against an A2A server. Satellites that depend on the audio binary protocol
(`hivemind-audio-binary-protocol`) do **not** apply here — like a persona server, an A2A
server exposes a text/query agent, not server-side audio processing.

---

## Install

```bash
pip install hivemind-core hivemind-a2a-agent-plugin
```

The plugin registers itself under the `hivemind.agent.protocol` entry-point group, so
`hivemind-core` discovers it automatically once installed.

You also need a running A2A server to point it at. Any compliant A2A server works; the
plugin only owns the outbound HTTP connection to it.

---

## Configure hivemind-core

Point the `agent_protocol` block in `~/.config/hivemind-core/server.json` at the A2A
plugin and give it the URL of your A2A server:

```json
{
  "agent_protocol": {
    "module": "hivemind-a2a-agent-plugin",
    "hivemind-a2a-agent-plugin": {
      "agent_url": "http://localhost:9999",
      "auth_header": "Bearer secret",
      "timeout": 60,
      "streaming": false
    }
  }
}
```

In practice you set one key and ignore the rest: point `agent_url` at your agent and
you're done. The others are there for when you need auth, a longer timeout, or streaming:

| Key           | Default | Description                                              |
|---------------|---------|----------------------------------------------------------|
| `agent_url`   | —       | **Required.** Root URL of the A2A server.                |
| `auth_header` | —       | Optional `Authorization` header value (e.g. `Bearer …`). |
| `timeout`     | `60.0`  | HTTP timeout in seconds.                                 |
| `streaming`   | `false` | Prefer `tasks/sendSubscribe` (SSE) when `true`.          |

With `streaming` left at `false` `hivemind-core` submits a blocking `tasks/send` and returns the
final text. Set it to `true` to use `tasks/sendSubscribe` and stream chunks back to the
satellite as the agent produces them, provided the A2A server advertises streaming.

If `agent_url` is missing, the plugin logs a warning at startup and answers every query
with an error string instead of contacting a server. The plugin never silences errors:
if the A2A server is unreachable, returns an empty response, or returns a JSON-RPC error
object, a human-readable error string is sent back so the satellite always gets a reply.

The HiveMind `session_id` is mapped 1-to-1 to the A2A `sessionId`, so multi-turn
conversation context is preserved on the A2A server side; the plugin itself stores no
conversation state.

??? note "Advanced: configuring from the OVOS global config"
    The same four keys can also be supplied through the OVOS global configuration
    under `hivemind` → `a2a_agent` (e.g. in `mycroft.conf`), instead of the
    `agent_protocol` block in `server.json`:

    ```yaml
    hivemind:
      a2a_agent:
        agent_url: "http://localhost:9999"
        auth_header: "Bearer secret"   # optional
        timeout: 60                    # optional
        streaming: false               # optional
    ```

    When both are present, explicit values passed in the `agent_protocol` block win
    over the OVOS-config fallback.

---

## Start the server

```bash
hivemind-core listen
```

Satellites connecting to this server now receive answers from the A2A agent — streamed
chunk by chunk when `streaming` is enabled, or as a single response otherwise.

---

## Managing clients

Client management is the normal `hivemind-core` CLI — there is no separate A2A command.
Node IDs are **positional**:

```bash
# Add a new client
hivemind-core add-client --name "my-client"

# List clients
hivemind-core list-clients

# Remove client with node ID 2
hivemind-core delete-client 2
```

See [OVOS Skills Server](ovos-server.md#managing-clients) for the full client workflow.

---

## Next

- [Operations](operations.md) — TLS, reverse proxy, systemd, observability, and scaling.
- [Persona Server](persona-server.md) — answer queries from a local LLM / solver chain instead
  of an external A2A server.
- [Database Backends](../concepts/databases.md) — where client and permission state lives.

---

## Source

Validated against the HiveMind source:

- [`hivemind_a2a_agent_plugin/__init__.py`](https://github.com/JarbasHiveMind/hivemind-a2a-agent-plugin/blob/HEAD/hivemind_a2a_agent_plugin/__init__.py) — the `agent_url` / `auth_header` / `timeout` / `streaming` keys, the OVOS-config fallback, and the `session_id` → A2A `sessionId` mapping
- [`hivemind_a2a_agent_plugin/_client.py`](https://github.com/JarbasHiveMind/hivemind-a2a-agent-plugin/blob/HEAD/hivemind_a2a_agent_plugin/_client.py) — the JSON-RPC `tasks/send` / `tasks/sendSubscribe` calls and error handling
