# HackChat

**[HiveMind-HackChatBridge](https://github.com/JarbasHiveMind/HiveMind-HackChatBridge)
relays a [hack.chat](https://hack.chat/) channel to and from a HiveMind.** Whatever
is said in the channel is forwarded to `hivemind-core` as an
[utterance](../reference/glossary.md#utterance), and the spoken reply is posted
back into the same channel.

!!! abstract "In a nutshell"
    - Runs as the `hivemind-hackchat-bridge` console command; pick any channel name and a bot nickname.
    - Needs no platform credentials — only your HiveMind identity (stored, or passed via `--access-key` / `--password` / `--host`).
    - The bridge client needs at minimum `allow-msg "speak"`.

hack.chat is a minimal, **anonymous** public chat — you join a channel just by naming
it, with no sign-up. That makes this the simplest bridge to try: there are **no
platform credentials to obtain**, only your HiveMind identity.

!!! tip "Beginner's mental model"
    Pick any channel name, pick a nickname for the bot, point it at `hivemind-core`. The bot
    joins that hack.chat channel and relays messages to your hive. No account, no
    token, no API key on the chat side.

---

## Install

```bash
pip install HiveMind-HackChatBridge
```

This installs the `hivemind-hackchat-bridge` console command.

---

## Set your HiveMind identity

The bridge connects to `hivemind-core` with your stored
[node identity](../reference/glossary.md#node). Set it once on the machine that runs
the bridge:

```bash
hivemind-client set-identity
```

You can also pass `--access-key`, `--password`, and `--host` directly instead.

---

## Usage

```
Usage: hivemind-hackchat-bridge [OPTIONS]

Options:
  --channel TEXT      hack.chat channel name to join  [required]
  --username TEXT     Bot nickname shown in the channel (default: Jarbas_BOT)
  --access-key TEXT   HiveMind access key
  --password TEXT     HiveMind password
  --host TEXT         HiveMind host, ws:// or wss:// (default: ws://127.0.0.1)
  --port INTEGER      HiveMind port (default: 5678)
  --self-signed       Accept self-signed SSL certificates
  --lang TEXT         Utterance language (default: en-us)
```

Example:

```bash
hivemind-hackchat-bridge \
  --channel my-hive-channel \
  --username Jarbas_BOT \
  --access-key <access_key> \
  --password <password> \
  --host ws://127.0.0.1
```

Once running, open `https://hack.chat/?my-hive-channel` in a browser, type into the
channel, and the bot's replies appear inline.

??? note "Advanced: anonymity and addressing"
    Because hack.chat has no accounts, every participant is identified only by the
    nickname they choose for that session. The bridge forwards each message with the
    sender's nickname so `hivemind-core` can address its reply back to the right person.

---

## How it works

The bridge runs as a HiveMind client. Each channel message becomes a
`recognizer_loop:utterance` [BUS message](../reference/glossary.md#bus-message) sent
to `hivemind-core`; the `speak` response is posted back into the channel.

---

## Permissions

The bridge client needs at minimum:

```bash
hivemind-core allow-msg "speak" <id>
```

---

## Source

Validated against the HiveMind source:

- [`pyproject.toml`](https://github.com/JarbasHiveMind/HiveMind-HackChatBridge/blob/HEAD/pyproject.toml) — package name and the `hivemind-hackchat-bridge` console script
- [`hackchat_bridge/__main__.py`](https://github.com/JarbasHiveMind/HiveMind-HackChatBridge/blob/HEAD/hackchat_bridge/__main__.py) — CLI options and defaults
