# HiveMind - Matrix Bridge

[Matrix](https://matrix.org/) is an open chat network, a bit like email but instant. You register an account with any provider ("homeserver"), and from that account you can join rooms and talk to people on other homeservers, the same way you can email someone on a different mail provider. You can also run your own homeserver instead of using a public one.

The Matrix bridge connects one Matrix room to a HiveMind hub. It acts as a HiveMind satellite: instead of a microphone, its input is anything said in the room that mentions the bot, and instead of a speaker, its output is the hub's reply posted back into the room. This turns a HiveMind hub, and the OVOS skills behind it, into a chatbot anyone in that room can talk to.

```
Matrix room  ⇄  HiveMind-matrix-bridge  ⇄  HiveMind hub  ⇄  OVOS skills
```

## Get a bot account and an access token

The bridge needs a Matrix account it can log in as, and the access token for that account. Two ways to get one:

**Use a public homeserver.** Register an account for the bot on `matrix.org` (or any other public homeserver), log in once with a client such as [Element](https://element.io), then copy the account's access token: *Settings → Help & About → Access Token*.

**Self-host.** Run a homeserver such as [Conduit](https://conduit.rs/) yourself, register the bot account on it, and get its token the same way. Self-hosting means the bot account is not tied to a third-party server, and you can keep the room off the public federation entirely.

Either way, create or pick a room, note its alias (for example `#hivemind-bots:matrix.org`), and invite the bot account into that room.

## Install

```bash
pip install git+https://github.com/JarbasHiveMind/HiveMind-matrix-bridge
```

This installs the `HiveMind-matrix` command.

## Register the bridge on the hub

On the machine running `hivemind-core`:

```bash
hivemind-core add-client --name matrix-bridge \
  --access-key "your-access-key" --password "your-password"
```

A freshly registered client is denied every message type until you whitelist it. This step is easy to miss, and skipping it is the most common reason a bridge connects but never replies:

```bash
hivemind-core allow-msg recognizer_loop:utterance matrix-bridge
hivemind-core allow-msg speak matrix-bridge
```

The first command lets the bridge send what it hears into the hub. The second lets the hub's spoken replies come back out. Without both, the bridge sits there connected and silent.

If you run more than one HiveMind bridge on the same machine (Matrix next to Mattermost, DeltaChat, or HackChat), give each one its own identity with `hivemind-client set-identity`. Bridges that share an identity share a Noise session pin, and the hub will treat reconnects from either one as the same client, which breaks encryption for both.

## Run it

```bash
HiveMind-matrix run \
  --botname thehivebot \
  --matrixtoken "syt_your_access_token" \
  --matrixhost "https://matrix.org" \
  --room "#hivemind-bots:matrix.org"
```

The bridge logs a line for the Matrix connection and a line for the HiveMind connection when both come up.

## Try it

In the joined room, mention the bot:

```
thehivebot what time is it?
```

The bridge strips the mention, sends the rest to the hub as a `recognizer_loop:utterance`, waits for the hub's `speak` reply, and posts it back into the room.

## Troubleshooting

- **Bridge connects but the room never gets a reply**: the client is registered but not whitelisted. Run the two `allow-msg` commands above.
- **`invalid api key` at connect time**: the bridge, or its `hivemind-bus-client` dependency, is older than the hub. Upgrade it.
- **`reconnect worker already running` in the log**: a known issue in older `hivemind-bus-client` releases when a dropped connection's retries overlap. It is fixed upstream; upgrade.
- **Handshake fails after the hub was reinstalled or the client's key changed**: the bridge is holding a stale Noise session pin. Clear it on the hub with `hivemind-core reset-noise-pin matrix-bridge` and restart the bridge.

## More

The bridge's own repository has a full [setup walkthrough](https://github.com/JarbasHiveMind/HiveMind-matrix-bridge/blob/dev/docs/setup.md) and [operator setup guide](https://github.com/JarbasHiveMind/HiveMind-matrix-bridge/blob/dev/docs/operator-setup.md), including how to self-host Conduit.

---
[← OpenVoiceOS Pipeline](pipeline.md) · [Home](index.md) · [Mattermost →](11_mattermost.md)
