# Telegram Bridge

Give your hive a Telegram bot. The
[hivemind-telegram-bridge](https://github.com/JarbasHiveMind/hivemind-telegram-bridge)
turns any text message sent to the bot into a HiveMind utterance, and posts the hive's
spoken reply back as a text message in the same chat.

!!! abstract "In a nutshell"
    - Runs the `hivemind-telegram-bridge` console command, using `python-telegram-bot`'s async, long-polling `Application`.
    - Forwards each text message as `recognizer_loop:utterance`; posts `speak` replies back into the originating chat.
    - Waits for the HiveMind handshake to complete before forwarding anything, so an early message can't get the connection killed.

!!! tip "Beginner's mental model"
    Message the bot like you'd message a friend. Whatever you type is forwarded to your
    hive, and the hive's answer comes back as the bot's reply.

---

## Get a bot token

1. Open Telegram and start a chat with [@BotFather](https://t.me/BotFather).
2. Send `/newbot` and follow the prompts: pick a display name, then a username ending
   in `bot` (it must be unique across Telegram).
3. BotFather replies with a token, e.g.
   `123456789:AAExampleTokenDoNotShareThisInAnyForm`.

!!! warning "Treat the token like a password"
    Anyone who has it can send messages as your bot and read what's sent to it. Don't
    commit it to a repo, paste it into a public issue, or pass it on a shared shell's
    command line — use an environment variable instead.

---

## Install

```bash
git clone https://github.com/JarbasHiveMind/hivemind-telegram-bridge
cd hivemind-telegram-bridge
pip install .
```

Or with Docker — see the repository README for the full `docker run` /
`docker-compose.yml` invocation.

---

## Permissions

```bash
hivemind-core add-client --name telegram-bridge
hivemind-core allow-msg "recognizer_loop:utterance" <id>
hivemind-core allow-msg "speak" <id>
```

A freshly added client is denied every message type by default. Skipping this step is
the classic failure mode: the bot receives your message fine, but nothing ever comes
back. The hub does send an explicit `hive.policy.denied` error over the connection, but
the bridge doesn't currently surface that error to the chat, so it looks silent.

---

## Run

```bash
hivemind-telegram-bridge \
  --token <your-botfather-token> \
  --access-key <key> --password <password> \
  --host ws://core.example.com --port 5678 \
  --site-id telegram-bridge
```

Give it its own `--site-id` if you run other HiveMind clients on the same host, so it
doesn't collide over the same identity file and pinned peer keys.

By default the bridge answers in any chat it's part of. Restrict it to specific chats
with repeated `--allowed-chat <chat_id>` flags.

Message your bot to try it. You should see the bridge log the incoming message, the
hub log the utterance and its `speak` reply, and the bot post that reply back into the
chat.

---

## Source

Validated against the HiveMind source:

- [`readme.md`](https://github.com/JarbasHiveMind/hivemind-telegram-bridge/blob/HEAD/readme.md) — architecture, install, Docker, and message flow
- [`docs/setup.md`](https://github.com/JarbasHiveMind/hivemind-telegram-bridge/blob/HEAD/docs/setup.md) — full BotFather walkthrough and troubleshooting
