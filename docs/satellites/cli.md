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
pip install hivemind-cli
```

## Usage

Interactive mode — type utterances and see responses:

```bash
hivemind-client --host 192.168.1.10 --key <key> --password <password>
```

Or with an identity file set via `hivemind-client set-identity`:

```bash
hivemind-client
```

Send a single utterance and exit:

```bash
hivemind-client send-utterance "what time is it"
```

Send a raw OVOS message:

```bash
hivemind-client send-mycroft --msg "speak" --payload '{"utterance": "hello world"}'
```

## Other CLI commands

```bash
# Escalate a message up the hive chain
hivemind-client escalate --msg "recognizer_loop:utterance" --payload '{"utterances": ["hello"]}'

# Propagate a message to all nodes
hivemind-client propagate --msg "recognizer_loop:utterance" --payload '{"utterances": ["hello"]}'

# Test connection from the stored identity file
hivemind-client test-identity

# Set the identity file
hivemind-client set-identity \
  --key <key> --password <password> --host <host> --port 5678 --siteid myroom

# Reset PGP keys
hivemind-client reset-pgp
```

Run `hivemind-client --help` for a full list.
