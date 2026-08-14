# HiveMind - Mattermost Bridge

[Mattermost](https://mattermost.com/) is a self-hosted team chat app, similar to Slack: channels, direct messages, and bot accounts that post and read messages through an API.

The Mattermost bridge connects a HiveMind hub to a Mattermost channel through a bot account. It watches the channels the bot is a member of, forwards any message that mentions the bot to the hub as an utterance, and posts the hub's spoken reply back into the channel.

```
Mattermost channel  ⇄  HiveMind_mattermost_bridge  ⇄  HiveMind hub  ⇄  OVOS skills
```

## Get a server and a bot account

You need a Mattermost server (an existing team's server, or your own) and a bot account on it.

**Use an existing server.** Ask a Mattermost System Console admin to create a bot account, or create a regular user account for the bridge to log in as. Note its login email and password, and add it to the channels it should answer in.

**Self-host.** Run `mattermost-preview` (a single Docker container with the server and database bundled) if you just want to try the bridge without touching a production team's server. Create the team, channel, and bot account inside it the same way.

## Install

```bash
pip install .
```

from a checkout of the repository, or install from PyPI once published. This provides the `hivemind-mattermost-bridge` command.

## Register the bridge on the hub

On the machine running `hivemind-core`:

```bash
hivemind-core add-client --name mattermost-bridge \
  --access-key "your-access-key" --password "your-password"
```

A freshly registered client is denied every message type until you whitelist it. Skipping this is the most common reason a bridge connects but never replies:

```bash
hivemind-core allow-msg recognizer_loop:utterance mattermost-bridge
hivemind-core allow-msg speak mattermost-bridge
```

If you run more than one HiveMind bridge on the same machine, give each its own identity. Bridges that share an identity share a Noise session pin, and the hub treats reconnects from either as the same client, breaking encryption for both.

## Run it

```bash
hivemind-mattermost-bridge \
  --mail bot@example.com \
  --pswd bot-password \
  --url chat.example.com \
  --tag @bot \
  --host ws://127.0.0.1 \
  --port 5678 \
  --key your-access-key \
  --password your-hivemind-password
```

`--url` is the bare Mattermost host, no `https://` prefix.

## Try it

In a channel the bot is a member of, mention it:

```
@bot what time is it?
```

The bridge forwards the message to the hub and posts the hub's reply back into the channel.

## Troubleshooting

- **Bridge connects but never replies**: the client is registered but not whitelisted. Run the two `allow-msg` commands above.
- **Bot never answers**: confirm the bot account is a member of the channel and the message contains the trigger tag (`@bot` by default).
- **Mattermost login fails**: check `--url` is the bare host with no scheme, and the bot's login and password are correct.
- **`invalid api key` at connect time**: the bridge, or its `hivemind-bus-client` dependency, is older than the hub. Upgrade it.
- **`reconnect worker already running` in the log**: a known issue in older `hivemind-bus-client` releases when a dropped connection's retries overlap. Fixed upstream; upgrade.
- **Handshake fails after the hub was reinstalled or the client's key changed**: the bridge is holding a stale Noise session pin. Clear it on the hub with `hivemind-core reset-noise-pin mattermost-bridge` and restart the bridge.

## More

The bridge's own repository has a full [setup walkthrough](https://github.com/JarbasHiveMind/HiveMind_mattermost_bridge/blob/dev/docs/setup.md) and [operator setup guide](https://github.com/JarbasHiveMind/HiveMind_mattermost_bridge/blob/dev/docs/operator-setup.md).

---
[← Matrix](09_matrix.md) · [Home](index.md) · [DeltaChat →](10_deltachat.md)
