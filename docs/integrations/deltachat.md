# DeltaChat Bridge

Here's a quietly clever one: talk to your assistant over plain email. [DeltaChat](https://delta.chat/)
is an encrypted messenger that rides on ordinary email accounts, and this bridge gives it
a bot on the other end. Email the bot, and your message goes to the hive; the reply
arrives as a chat message in the same conversation. No chat server to run, no app to
mandate — if you can send an email, you can reach your hive. It's a HiveMind satellite
that happens to speak email, relaying [utterances](../reference/glossary.md#utterance) to
hivemind-core.

!!! abstract "In a nutshell"
    - Runs as the `hm-deltachat-bridge` console command, relaying between an email account and `hivemind-core`.
    - Needs two credential sets: an email account for DeltaChat and the stored HiveMind node identity.
    - The bridge client needs at minimum `allow-msg "speak"`.

DeltaChat is an end-to-end encrypted messenger that rides on ordinary **email** — any
mailbox works as its transport, so there is no separate chat server to run. To people
chatting with it, the bot looks like a normal DeltaChat contact.

!!! tip "Beginner's mental model"
    You give the bridge two sets of credentials: an **email account** (so it can send
    and receive DeltaChat messages) and your **HiveMind identity** (so it can talk to
    `hivemind-core`). It sits in the middle and relays between the two.

---

## Install

```bash
pip install HiveMind-deltachat-bridge
```

This installs the `hm-deltachat-bridge` console command.

---

## Set your HiveMind identity

The bridge reads its HiveMind connection (access key, password, host) from the stored
[node identity](../reference/glossary.md#node). Set it once on the machine that will
run the bridge:

```bash
hivemind-client set-identity
```

If no identity is stored and you don't pass the connection flags manually, the bridge
exits with `NodeIdentity not set`.

---

## Usage

```
Usage: hm-deltachat-bridge [OPTIONS]

Options:
  --email TEXT           DeltaChat email address for the bot
  --email-password TEXT  Password for that email account
  --key TEXT             HiveMind access key (default: from identity file)
  --password TEXT        HiveMind password (default: from identity file)
  --host TEXT            HiveMind host (default: from identity file)
  --port INTEGER         HiveMind port (default: 5678)
```

Example:

```bash
hm-deltachat-bridge \
  --email bot@example.com \
  --email-password "s3cr3t"
```

!!! note "HiveMind flags are optional"
    `--key`, `--password`, and `--host` only override what's already stored by
    `hivemind-client set-identity`. In the common case you set the identity once and
    pass just the two email flags.

??? note "Advanced: host normalization and shutdown"
    If `--host` is given without a scheme it is rewritten to `ws://<host>`. The
    command blocks until it receives an exit signal (Ctrl-C), then cleanly stops the
    bridge node.

---

## How it works

The bridge runs as a HiveMind client. An incoming DeltaChat message becomes a
`recognizer_loop:utterance` [BUS message](../reference/glossary.md#bus-message) sent
to `hivemind-core`. The `speak` response is delivered back to the originating DeltaChat
conversation.

---

## Permissions

The bridge client needs at minimum permission to receive spoken replies:

```bash
hivemind-core allow-msg "speak" <id>
```

Grant additional message types if your use case needs them.

---

## Source

Validated against the HiveMind source:

- [`pyproject.toml`](https://github.com/JarbasHiveMind/HiveMind-deltachat-bridge/blob/HEAD/pyproject.toml) — package name and the `hm-deltachat-bridge` console script
- [`hm_deltachat_bridge/__main__.py`](https://github.com/JarbasHiveMind/HiveMind-deltachat-bridge/blob/HEAD/hm_deltachat_bridge/__main__.py) — CLI options and identity handling
