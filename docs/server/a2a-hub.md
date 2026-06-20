# A2A Hub

An "A2A hub" is a regular `hivemind-core` server whose **agent protocol** is the
[hivemind-a2a-agent-plugin](https://github.com/TigreGotico/hivemind-a2a-agent-plugin).
Instead of routing utterances into a full OVOS skills stack, the hub forwards
natural-language queries to an external **A2A (Agent-to-Agent)** server and streams the
answer back — no `ovos-core` required.

[A2A](https://google.github.io/A2A/) is Google's open protocol for agent
interoperability. An A2A server publishes an **agent card** at
`GET /.well-known/agent.json` describing its skills and task URL, and accepts **task**
requests as JSON-RPC 2.0 — either a blocking `tasks/send` or a streaming
`tasks/sendSubscribe` (SSE). Any compliant A2A backend (LangChain agents, Google ADK,
CrewAI, a custom FastAPI service, …) works behind this plugin.

Satellites built for `hivemind-core` (text-based ones like HiveMind-cli, voice-sat, or
bridges) work against an A2A hub. Satellites that depend on the audio binary protocol
(`hivemind-audio-binary-protocol`) do **not** apply here — like a persona hub, an A2A
hub exposes a text/query agent, not server-side audio processing.

## Install

```bash
pip install hivemind-core hivemind-a2a-agent-plugin
```

The plugin registers itself under the `hivemind.agent.protocol` entry-point group, so
`hivemind-core` discovers it automatically once installed.

You also need a running A2A server to point it at. Any compliant A2A server works; the
plugin only owns the outbound HTTP connection to it.

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

Only `agent_url` is required. The configuration keys are:

| Key           | Default | Description                                              |
|---------------|---------|----------------------------------------------------------|
| `agent_url`   | —       | **Required.** Root URL of the A2A server.                |
| `auth_header` | —       | Optional `Authorization` header value (e.g. `Bearer …`). |
| `timeout`     | `60`    | HTTP timeout in seconds.                                 |
| `streaming`   | `false` | Prefer `tasks/sendSubscribe` (SSE) when `true`.          |

With `streaming` left at `false` the hub submits a blocking `tasks/send` and returns the
final text. Set it to `true` to use `tasks/sendSubscribe` and stream chunks back to the
satellite as the agent produces them, provided the A2A server advertises streaming.

If `agent_url` is missing, the plugin logs a warning at startup and answers every query
with an error string instead of contacting a server. The plugin never silences errors:
if the A2A server is unreachable, returns an empty response, or returns a JSON-RPC error
object, a human-readable error string is sent back so the satellite always gets a reply.

The HiveMind `session_id` is mapped 1-to-1 to the A2A `sessionId`, so multi-turn
conversation context is preserved on the A2A server side; the plugin itself stores no
conversation state.

## Start the server

```bash
hivemind-core listen
```

Satellites connecting to this hub now receive answers from the A2A agent — streamed
chunk by chunk when `streaming` is enabled, or as a single response otherwise.

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

See [OVOS Skills Hub](ovos-hub.md#managing-clients) for the full client workflow.

## Next

- [Operations](operations.md) — TLS, reverse proxy, systemd, observability, and scaling.
- [Persona Hub](persona-hub.md) — answer queries from a local LLM / solver chain instead
  of an external A2A server.
- [Database Backends](../concepts/databases.md) — where client and permission state lives.
