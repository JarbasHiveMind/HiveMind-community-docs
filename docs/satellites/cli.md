# HiveMind CLI

**HiveMind-cli is the simplest satellite — a text-only terminal client that needs no audio hardware.** You type utterances and hivemind-core does everything; responses arrive as text.

!!! abstract "In a nutshell"
    - **Runs locally**: a terminal interface — no audio processing at all.
    - **hivemind-core provides**: STT, TTS, skills, and intents; replies come back as text.
    - Installs as the `hivemind-cli` console script (`pip install HiveMind-cli`).
    - Ideal for testing hivemind-core instances and skills, scripting, and audio-less IoT devices.

---

## When to use it

- Testing and debugging a hivemind-core instance or a skill
- Scripting and automation from a server
- IoT devices with network connectivity but no audio hardware
- Any scenario where you want to interact via keyboard or piped input

---

## Install

```bash
pip install HiveMind-cli
```

This provides the `hivemind-cli` console script — a terminal client with no
subcommands, just flags.

---

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

---

## CLI flags

| Flag | Default | Description |
|---|---|---|
| `--access-key` | *(required)* | Client access key issued by `hivemind-core add-client`. |
| `--password` | `None` | Optional client password. |
| `--host` | *(scan)* | HiveMind host URI, including protocol: `ws://…` or `wss://…`. |
| `--port` | `5678` | WebSocket port. |
| `--no-curses` | off | Disable the curses UI; use plain stdout/stdin instead. |
| `--self-signed` | off | Accept self-signed SSL certificates. |

---

## See also

For scriptable, non-interactive use, the `hivemind-client` command from the
[hivemind-bus-client](https://github.com/JarbasHiveMind/hivemind-websocket-client)
package offers subcommands such as `set-identity`, `terminal`, `send-mycroft`,
`escalate`, `propagate`, and `ping`. That is a separate package from `HiveMind-cli`.

---

## Next

New to HiveMind? Start with the [Quick Start](../quickstart.md), or learn how the pieces fit in [Core Concepts](../concepts/mesh.md).

---

## Source

Validated against the HiveMind source:

- [`hivemind_cli_terminal/__main__.py`](https://github.com/JarbasHiveMind/HiveMind-cli/blob/HEAD/hivemind_cli_terminal/__main__.py) — flags (`--access-key --password --host --port --no-curses --self-signed`), the `ws://`/`wss://` host requirement, and the UDP-scan fallback when `--host` is omitted
- [`docs/usage.md`](https://github.com/JarbasHiveMind/HiveMind-cli/blob/HEAD/docs/usage.md) — usage walkthrough
