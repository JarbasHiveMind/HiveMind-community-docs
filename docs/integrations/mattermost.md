# Mattermost Bridge

[HiveMind-mattermost-bridge](https://github.com/JarbasHiveMind/HiveMind_mattermost_bridge) connects a Mattermost workspace to a HiveMind hub. Users can interact with the AI assistant through direct messages or channel mentions.

## Install

```bash
pip install HiveMind-mattermost-bridge
```

## Configuration

Create `~/.config/hivemind-mattermost-bridge/config.json`:

```json
{
  "mattermost_url": "https://mattermost.example.com",
  "bot_token": "xxxxx",
  "bot_username": "hivemind",
  "hivemind_host": "192.168.1.10",
  "hivemind_port": 5678,
  "hivemind_key": "access-key",
  "hivemind_password": "password"
}
```

Set up a Mattermost outgoing webhook (System Console → Integrations → Outgoing Webhooks) pointing to `http://<bridge-server>:5000/webhook`.

## Usage

**Direct message the bot:**

```
User: hello
HiveMind Bot: Hello! How can I help?
```

**Mention the bot in a channel:**

```
User: @hivemind what's the weather?
```

## How it works

Each Mattermost user maps to a unique HiveMind peer. Incoming messages become `recognizer_loop:utterance` OVOS messages sent to the hub. Responses (`speak`) are posted back to the originating conversation.
