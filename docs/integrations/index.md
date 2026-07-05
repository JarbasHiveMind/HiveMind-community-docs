# Integrations

A **bridge** connects HiveMind to a chat network or smart-home platform, so people can
talk to your hive from apps they already use — Matrix, a Twitch stream, Home Assistant,
and more. Under the hood every bridge is just another [satellite](../reference/glossary.md#roles):
it holds an access key and relays messages to `hivemind-core`.

!!! note "Bridges are satellites too"
    Anything in this section connects the same way the [Quick Start](../quickstart.md)
    satellite did — register it with `hivemind-core add-client`, give it the access key
    and password, point it at `hivemind-core`. The integration-specific part is only *which
    outside service* it relays to.

---

## What's in this section

| Integration | Connects HiveMind to | Credentials needed |
|---|---|---|
| [Home Assistant](home-assistant.md) | Your smart home — control HiveMind from HA, or HA from HiveMind | HiveMind only |
| [Media Player](media-player.md) | A headless device, as a network-controlled OCP player | HiveMind only |
| [Matrix](matrix.md) | A Matrix room (open, federated chat) | Matrix bot token |
| [DeltaChat](deltachat.md) | Email-based encrypted messaging | Email account |
| [Mattermost](mattermost.md) | A Mattermost team channel | Mattermost bot login |
| [Twitch](twitch.md) | Twitch stream chat | Twitch chat OAuth token |
| [HackChat](hackchat.md) | An anonymous hack.chat channel | HiveMind only |

!!! tip "Start with the simplest"
    [HackChat](hackchat.md) and the [Media Player](media-player.md) need no platform
    credentials at all — only your HiveMind identity — so they are the quickest to try
    first. Each page documents its exact CLI and the message permissions the bridge
    needs.

---

**Next:** pick a platform above, or see [Choosing a Satellite](../satellites/index.md)
for direct (non-bridge) clients.
