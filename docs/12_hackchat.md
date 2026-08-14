# HiveMind - HackChat Bridge

[hack.chat](https://hack.chat/) is a small, anonymous, no-signup chat site. You open a URL with a channel name in it (`https://hack.chat/?your_channel`) and you are in that channel, with a random nickname you can change. There is no account, no password, and no registration for anyone, human or bot.

The HackChat bridge connects one hack.chat channel to a HiveMind hub. It joins the channel like any other participant, forwards every message it sees (except its own) to the hub as an utterance, and posts the hub's reply back as `@user , <answer>`.

```
hack.chat channel  ⇄  HiveMind-HackChatBridge  ⇄  HiveMind hub  ⇄  OVOS skills
```

## Get an account

There is nothing to get. Pick a channel name — something obscure, since anyone who knows it can join — and a nickname for the bot. That's the whole "account" step.

## Install

```bash
pip install .
```

from a checkout of the repository, or from PyPI once published. This provides the `hivemind-hackchat-bridge` command.

## Register the bridge on the hub

On the machine running `hivemind-core`:

```bash
hivemind-core add-client --name hackchat-bridge \
  --access-key "your-access-key" --password "your-password"
```

A freshly registered client is denied every message type until you whitelist it. Skipping this is the most common reason a bridge connects but never replies:

```bash
hivemind-core allow-msg recognizer_loop:utterance hackchat-bridge
hivemind-core allow-msg speak hackchat-bridge
```

If you run more than one HiveMind bridge on the same machine, give each its own credentials. Bridges that share an identity share a Noise session pin, and the hub treats reconnects from either as the same client, breaking encryption for both.

## Run it

```bash
hivemind-hackchat-bridge \
  --channel your_channel \
  --username Jarbas_BOT \
  --access-key "your-access-key" \
  --password "your-password" \
  --host ws://127.0.0.1 \
  --port 5678
```

## Try it

Open `https://hack.chat/?your_channel` in a browser and type:

```
what time is it?
```

The bridge forwards the message to the hub and posts the reply back as `@you , <answer>`.

## Troubleshooting

- **Bridge connects but never replies**: the client is registered but not whitelisted. Run the two `allow-msg` commands above.
- **Bot joins but never answers**: confirm the hub is reachable and the access key is registered (`hivemind-core list-clients`), and that the hub produces a `speak` for the answer.
- **Wrong channel**: the bot and the user must be in the same hack.chat channel name — open the same `https://hack.chat/?<channel>` URL.
- **`invalid api key` at connect time**: the bridge, or its `hivemind-bus-client` dependency, is older than the hub. Upgrade it.
- **`reconnect worker already running` in the log**: a known issue in older `hivemind-bus-client` releases when a dropped connection's retries overlap. Fixed upstream; upgrade.
- **Handshake fails after the hub was reinstalled or the client's key changed**: the bridge is holding a stale Noise session pin. Clear it on the hub with `hivemind-core reset-noise-pin hackchat-bridge` and restart the bridge.

## More

The bridge's own repository has a full [setup walkthrough](https://github.com/JarbasHiveMind/HiveMind-HackChatBridge/blob/dev/docs/setup.md) and [operator setup guide](https://github.com/JarbasHiveMind/HiveMind-HackChatBridge/blob/dev/docs/operator-setup.md).

---
[← DeltaChat](10_deltachat.md) · [Home](index.md) · [Home Assistant →](07_homeassistant.md)
