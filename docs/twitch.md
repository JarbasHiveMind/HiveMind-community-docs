# Twitch Bridge

A HiveMind integration that connects a Twitch channel to the HiveMind network, enabling live stream chat interaction with the AI assistant.

**Source**: `HiveMind-twitch-bridge` package

## Overview

The Twitch bridge monitors a Twitch channel's chat messages and relays them to the HiveMind network. Responses are sent back as chat messages in the stream, creating an interactive AI-powered chat experience for viewers.

## Features

- Real-time chat message monitoring
- Twitch OAuth authentication
- User context tracking (username, user ID)
- Command filtering (ignore bot commands, system messages)
- Configurable response rate limiting to avoid spam
- Support for multiple simultaneous streams

## Setup & Configuration

### Installation

```bash
pip install HiveMind-twitch-bridge
```

Or via `uv`:

```bash
uv pip install HiveMind-twitch-bridge
```

### Twitch OAuth Configuration

The bridge uses OAuth 2.0 for secure authentication:

1. **Register Application**: Go to https://dev.twitch.tv/console/apps
2. **Create Application**: Set OAuth Redirect URL to `http://localhost:3000/oauth`
3. **Generate OAuth Token**: Use the client ID and secret for bot authentication
4. **Store Credentials**: Save in `~/.config/hivemind-twitch-bridge/oauth.json`

### Bridge Configuration

Create `~/.config/hivemind-twitch-bridge/config.json`:

```json
{
  "twitch_channel": "my_channel",
  "twitch_client_id": "xxxxx",
  "twitch_oauth_token": "xxxxx",
  "hivemind_host": "192.168.1.10",
  "hivemind_port": 5678,
  "hivemind_key": "access-key",
  "hivemind_password": "password",
  "response_rate_limit": 1.0,
  "message_prefix": "[AI]"
}
```

**Key fields**:
- **`twitch_channel`**: Target channel name
- **`twitch_client_id`**, **`twitch_oauth_token`**: OAuth credentials
- **`hivemind_*`**: HiveMind hub connection details
- **`response_rate_limit`**: Seconds between messages (prevents spam)
- **`message_prefix`**: Prefix for bot responses (e.g., `[AI]: response`)

## Bridge Architecture

The bridge operates as a chat bot:

1. **Chat Monitoring**: Watches Twitch chat using EventSub webhooks or IRC connection
2. **Message Processing**: Filters system messages, detects commands
3. **HiveMind Query**: Sends user message to the hub as an utterance
4. **Response Delivery**: Receives bot response and posts to chat
5. **Rate Limiting**: Enforces configurable delay between responses

## Usage

### In Live Chat

```
Viewer: hello bot
[AI]: Hello! Thanks for watching!
```

### Command Syntax

Standard utterance commands:

```
Viewer: weather for seattle
[AI]: The weather in Seattle is...
```

## Security Considerations

- **OAuth Scope**: Use minimal scopes (`chat:read`, `chat:edit`)
- **Token Rotation**: Periodically rotate OAuth tokens for security
- **Rate Limiting**: Configure `response_rate_limit` to prevent abuse
- **Moderation**: Use Twitch moderation tools alongside the bridge

## Troubleshooting

**Bridge not responding**:
- Verify OAuth token is valid (check expiration)
- Check HiveMind hub connectivity and credentials
- Review bridge logs for connection errors

**Messages not appearing**:
- Verify bot account has channel moderator privileges
- Check Twitch chat rate limits (standard accounts have limits)
- Review message filtering configuration

## See Also

- [Integrations: Mattermost Bridge](/HiveMind-community-docs/docs/mattermost.md)
- [Integrations: Matrix Bridge](/HiveMind-community-docs/docs/09_matrix.md)
- [Integrations: DeltaChat Bridge](/HiveMind-community-docs/docs/10_deltachat.md)
