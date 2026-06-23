# Mattermost Bridge

[HiveMind-mattermost-bridge](https://github.com/JarbasHiveMind/HiveMind_mattermost_bridge)
connects a [Mattermost](https://mattermost.com/) team server to a HiveMind hub. People
talk to the assistant through direct messages or by mentioning the bot in a channel;
the hub's reply is posted back into the same conversation.

!!! tip "Beginner's mental model"
    The bridge logs into Mattermost as a normal **bot user account** (an email/login
    and password, not an API token), and connects to your hub using your stored
    HiveMind identity. It relays messages between the two.

## Install

```bash
pip install HiveMind-mattermost-bridge
```

This installs the `hivemind-mattermost-bridge` console command.

## Set your HiveMind identity

HiveMind connection details (access key, password, host, port) default to the values
stored by [`hivemind-client set-identity`](../reference/cli.md). Set it once on the
machine that runs the bridge, then you only need to pass the Mattermost flags:

```bash
hivemind-client set-identity
```

## Usage

```
Usage: hivemind-mattermost-bridge [OPTIONS]

Mattermost (required):
  --mail TEXT      Mattermost bot account email / login id
  --pswd TEXT      Mattermost bot account password
  --url TEXT       Mattermost server host (no scheme, e.g. chat.example.com)

Mattermost (optional):
  --tag TEXT       Tag that triggers the bot in a channel (repeatable,
                   default: @bot)

HiveMind:
  --key TEXT       HiveMind access key (default: from identity file)
  --password TEXT  HiveMind password (default: from identity file)
  --host TEXT      HiveMind host, e.g. ws://127.0.0.1 (default: from identity file)
  --port INTEGER   HiveMind port (default: 5678)
  --self-signed    Accept self-signed SSL certificates
  --lang TEXT      Utterance language (default: en-us)
```

Example:

```bash
hivemind-mattermost-bridge \
  --mail bot@example.com \
  --pswd "bot-password" \
  --url chat.example.com
```

You can also run it as a module:

```bash
python -m mattermost_bridge --mail bot@example.com --pswd "bot-password" --url chat.example.com
```

**Direct-message the bot:**

```
User: hello
Bot: @user , Hello! How can I help?
```

**Mention the bot in a channel** (using a trigger tag):

```
User: @bot what's the weather?
Bot: @user , The weather is...
```

??? note "Advanced: triggers, transport, and threading"
    `--tag` is repeatable, so you can give the bot several trigger mentions; it
    defaults to `@bot`. The bridge connects to Mattermost as an outbound websocket
    client via `mattermostdriver` — it runs **no** HTTP server, so no incoming
    webhook or listener port is needed on the Mattermost side. The Mattermost listen
    loop runs in a daemon thread alongside the HiveMind bus client.

## How it works

The bridge runs as a HiveMind client. A direct message or mention becomes a
`recognizer_loop:utterance` [BUS message](../reference/glossary.md#bus-message)
tagged with the sender and channel. The hub's `speak` response is posted back to that
channel, addressed to the sender as `@user , <answer>`.

## Permissions

The bridge client needs at minimum:

```bash
hivemind-core allow-msg "speak" <id>
```

## Source

Validated against the HiveMind source:

- [`pyproject.toml`](https://github.com/JarbasHiveMind/HiveMind_mattermost_bridge/blob/HEAD/pyproject.toml) — package name and the `hivemind-mattermost-bridge` console script
- [`mattermost_bridge/__main__.py`](https://github.com/JarbasHiveMind/HiveMind_mattermost_bridge/blob/HEAD/mattermost_bridge/__main__.py) — CLI arguments and connection handling
