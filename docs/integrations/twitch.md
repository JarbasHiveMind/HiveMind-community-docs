# Twitch Bridge

[HiveMind-twitch-bridge](https://github.com/JarbasHiveMind/HiveMind-twitch-bridge) connects a Twitch channel's live chat to a HiveMind hub. Chat messages are relayed to the AI assistant and responses appear in the stream chat.

## Install

```bash
pip install HiveMind-twitch-bridge
```

## Configuration

Register a Twitch application at [dev.twitch.tv](https://dev.twitch.tv/console/apps) and obtain OAuth credentials.

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

`response_rate_limit` sets the minimum seconds between bot responses (prevents spam).

## Usage

```
Viewer: weather for seattle
[AI]: The weather in Seattle is...
```

## Security notes

- Use minimal OAuth scopes (`chat:read`, `chat:edit`)
- Rotate OAuth tokens periodically
- Configure `response_rate_limit` to prevent response abuse
