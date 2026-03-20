# Mattermost Bridge

A HiveMind integration bridge that connects a Mattermost workspace to the HiveMind network, enabling users to interact with the AI assistant through Mattermost chat.

**Source**: `HiveMind_mattermost_bridge` package

## Overview

The Mattermost bridge translates between Mattermost message events and HiveMind protocol messages. It allows direct message conversations with the AI bot or channel mentions for group interactions.

## Features

- Direct message conversations with the AI bot
- Channel mentions: `@hivemind hello world`
- Message threading and reply handling
- User context and session management
- Webhook-based event handling for real-time responsiveness

## Setup & Configuration

### Installation

```bash
pip install HiveMind-mattermost-bridge
```

Or via `uv`:

```bash
uv pip install HiveMind-mattermost-bridge
```

### Configuration

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

**Required fields**:
- **`mattermost_url`**: Mattermost server URL
- **`bot_token`**: Bot personal access token (created in Mattermost admin panel)
- **`bot_username`**: Display name for the bot in Mattermost
- **`hivemind_host`, `hivemind_port`**: HiveMind hub connection details
- **`hivemind_key`, `hivemind_password`**: Authentication credentials

### Webhook Setup

In Mattermost admin panel:
1. System Console → Integrations → Outgoing Webhooks
2. Create webhook for message events
3. Set URL to: `http://<bridge-server>:5000/webhook`

## Bridge Architecture

The bridge operates as a stateless relay:

1. **Mattermost → HiveMind**: User message becomes a HiveMind utterance
2. **HiveMind → Mattermost**: Bot response message is sent back to the user/channel
3. **Session Management**: Each Mattermost user maps to a unique HiveMind peer

## Usage

### Direct Message

Simply message the `@hivemind` bot directly:

```
User: hello
HiveMind Bot: Hello! How can I help?
```

### Channel Mention

Mention the bot in a channel:

```
User: @hivemind what's the weather?
HiveMind Bot: The weather is...
```

## See Also

- [Integrations: Matrix Bridge](/HiveMind-community-docs/docs/09_matrix.md)
- [Integrations: DeltaChat Bridge](/HiveMind-community-docs/docs/10_deltachat.md)
- [Integrations: Twitch Bridge](/HiveMind-community-docs/docs/twitch.md)
