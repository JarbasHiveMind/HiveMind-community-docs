# HiveMind CLI

The simplest HiveMind satellite — text only, no audio hardware required.

**What runs locally**: A CLI interface. No audio processing whatsoever.

**What the hub provides**: All processing (STT, TTS, skills, intents). Responses arrive as text.

## When to use it

- Testing and debugging a hub or a skill
- Scripting and automation from a server
- IoT devices with network connectivity but no audio hardware
- Any scenario where you want to interact via keyboard or piped input

## Install

```bash
pip install HiveMind-cli
```

This provides the `hivemind-cli` console script — a terminal client with no
subcommands, just flags.

## Usage

Interactive mode — type utterances and see responses:

```bash
hivemind-cli --access-key <key> --password <password> --host wss://192.168.1.10 --port 5678
```

The host **must** include the protocol prefix (`ws://` or `wss://`). If `--host` is
omitted, the CLI scans the local network for a HiveMind node via UDP broadcast and
asks before connecting.

Plain output (no curses UI) for scripting or SSH sessions:

```bash
hivemind-cli --access-key <key> --password <password> --host wss://192.168.1.10 --no-curses
```

## CLI flags

| Flag | Default | Description |
|---|---|---|
| `--access-key` | *(required)* | Client access key issued by `hivemind-core add-client`. |
| `--password` | `None` | Optional client password. |
| `--host` | *(scan)* | HiveMind host URI, including protocol: `ws://…` or `wss://…`. |
| `--port` | `5678` | WebSocket port. |
| `--no-curses` | off | Disable the curses UI; use plain stdout/stdin instead. |
| `--self-signed` | off | Accept self-signed SSL certificates. |

## See also

For scriptable, non-interactive use, the `hivemind-client` command from the
[hivemind-bus-client](https://github.com/JarbasHiveMind/hivemind-websocket-client)
package offers subcommands such as `set-identity`, `terminal`, `send-mycroft`,
`escalate`, `propagate`, and `ping`. That is a separate package from `HiveMind-cli`.
