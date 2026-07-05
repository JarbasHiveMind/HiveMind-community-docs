# Twitch Bridge

Give your stream a co-host that answers viewers. The Twitch bridge sits in your
[Twitch](https://www.twitch.tv/) chat as a bot: when a viewer tags it, their message goes
to your hive and the answer is posted straight back into chat, addressed to them by name.
It only speaks when spoken to — a trigger tag keeps it quiet the rest of the time — so it
rides along with your stream without talking over it. Under the hood it's a HiveMind
satellite relaying [utterances](../reference/glossary.md#utterance) to hivemind-core.

!!! abstract "In a nutshell"
    - Runs as the `hivemind-twitch-bridge` console command, joining chat over IRC with a channel name and a Twitch chat OAuth token.
    - Only messages containing a trigger tag are relayed; the tag is stripped before forwarding.
    - HiveMind credentials are passed as flags; the bridge client needs at minimum `allow-msg "speak"`.

!!! tip "Beginner's mental model"
    The bridge joins your Twitch chat as a bot (using a chat OAuth token) and connects
    to `hivemind-core`. Only messages that contain a trigger tag are relayed, so it stays
    quiet until someone calls on it.

---

## Install

```bash
pip install HiveMind-twitch-bridge
```

This installs the `hivemind-twitch-bridge` console command.

---

## Get a Twitch chat token

Authentication is just the channel name plus a Twitch **chat OAuth token** (of the
form `oauth:...`), which you can generate at
[twitchapps.com/tmi](https://www.twitchapps.com/tmi/). No Twitch application or client
ID is required.

---

## Usage

```
Usage: hivemind-twitch-bridge [OPTIONS]

Twitch (required):
  --channel TEXT      Twitch channel name to join
  --oauth TEXT        Twitch chat OAuth token (oauth:...)

Twitch (optional):
  --tag TEXT          Trigger tag, repeatable (default: @bot)
  --nickname TEXT     Twitch bot nickname (default: bot)
  --lang TEXT         Utterance language (default: en-us)

HiveMind:
  --access-key TEXT   HiveMind access key
  --password TEXT     HiveMind password
  --crypto-key TEXT   HiveMind payload crypto key
  --host TEXT         HiveMind host, ws:// or wss:// (default: wss://127.0.0.1)
  --port INTEGER      HiveMind port (default: 5678)
  --self-signed       Accept self-signed SSL certificates
```

Example:

```bash
hivemind-twitch-bridge \
  --channel my_channel \
  --oauth "oauth:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" \
  --access-key <access_key> \
  --password <password> \
  --host wss://127.0.0.1
```

You can also run it as a module:

```bash
python -m twitch_bridge --channel my_channel --oauth "oauth:..."
```

A chat message is relayed only if it contains one of the trigger tags; the tag is
stripped before forwarding. The reply is posted back to the channel addressed to the
viewer as `@user , <answer>`:

```
Viewer: @bot weather for seattle
Bot: @viewer , The weather in Seattle is...
```

!!! warning "Host must include a scheme"
    `--host` must start with `ws://` or `wss://`, otherwise the command exits with an
    error. There is no rate limiting and the reply prefix is fixed.

---

## How it works

The bridge reads Twitch chat over IRC. A message containing a trigger tag becomes a
`recognizer_loop:utterance` [BUS message](../reference/glossary.md#bus-message) sent
to `hivemind-core`; the `speak` response is sent back to the channel as
`@user , <answer>`.

---

## Permissions

The bridge client needs at minimum:

```bash
hivemind-core allow-msg "speak" <id>
```

---

## Source

Validated against the HiveMind source:

- [`pyproject.toml`](https://github.com/JarbasHiveMind/HiveMind-twitch-bridge/blob/HEAD/pyproject.toml) — package name and the `hivemind-twitch-bridge` console script
- [`twitch_bridge/__main__.py`](https://github.com/JarbasHiveMind/HiveMind-twitch-bridge/blob/HEAD/twitch_bridge/__main__.py) — CLI arguments, reply format, and host validation
