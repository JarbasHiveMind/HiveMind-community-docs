# Matrix Bridge

**[HiveMind-matrix-bridge](https://github.com/JarbasHiveMind/HiveMind-matrix-bridge) connects a [Matrix](https://matrix.org/) chat room to a HiveMind.** Messages posted in the room are forwarded to `hivemind-core` as utterances; responses come back as messages in the room.

!!! abstract "In a nutshell"
    - Logs into Matrix as a bot with an access token and relays room messages to `hivemind-core`.
    - HiveMind credentials come from the stored node identity (`hivemind-client set-identity`); the `run` command takes no HiveMind flags.
    - The bridge client needs at minimum `allow-msg "speak"`.

Matrix is a federated, end-to-end-encrypted chat protocol. You register a bot account at any Matrix provider and the bridge logs in as that bot.

!!! tip "Beginner's mental model"
    The bridge logs into Matrix as a bot using an access token, joins a room, and
    relays whatever is said there to `hivemind-core`. Its HiveMind credentials come from the
    [identity](../reference/glossary.md#node) you set once with `hivemind-client set-identity`.

---

## Install

```bash
pip install HiveMind-matrix-bridge
```

---

## Usage

```
Usage: HiveMind-matrix run [OPTIONS]

  Connect a Matrix chatroom to HiveMind

Options:
  --botname TEXT      Bot username (default: thehivebot)
  --matrixtoken TEXT  Matrix access token
  --matrixhost TEXT   Matrix homeserver URL (default: https://matrix.org)
  --room TEXT         Matrix room ID (e.g. #hivemind-bots:matrix.org)
```

The `run` command takes no HiveMind credential flags. HiveMind credentials are
read from the stored node identity (set once with `hivemind-client set-identity`),
which the bridge loads internally via `HiveMindSolver(autoconnect=True)`.

Example:

```bash
HiveMind-matrix run \
  --botname thehivebot \
  --matrixtoken syt_dGhl..... \
  --matrixhost https://matrix.org \
  --room "#hivemind-bots:matrix.org"
```

---

## How it works

The bridge runs as a HiveMind client. Each Matrix message from a room member is wrapped in a `recognizer_loop:utterance` OVOS message and sent to `hivemind-core` via `BUS`. `hivemind-core` processes the utterance and returns a `speak` response; the bridge posts the spoken text back to the Matrix room.

---

## Permissions required

The bridge client needs at minimum:

```bash
hivemind-core allow-msg "speak" <id>
```

Add further permissions if your use case requires access to specific skills or message types.

---

## Source

Validated against the HiveMind source:

- [`pyproject.toml`](https://github.com/JarbasHiveMind/HiveMind-matrix-bridge/blob/HEAD/pyproject.toml) — package name and the `HiveMind-matrix` console script
- [`hm_matrix_bridge/__main__.py`](https://github.com/JarbasHiveMind/HiveMind-matrix-bridge/blob/HEAD/hm_matrix_bridge/__main__.py) — the `run` subcommand and its options
