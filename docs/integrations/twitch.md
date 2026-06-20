# Twitch Bridge

[HiveMind-twitch-bridge](https://github.com/JarbasHiveMind/HiveMind-twitch-bridge) connects a Twitch channel's live chat to a HiveMind hub. Chat messages are relayed to the AI assistant and responses appear in the stream chat.

!!! note "Legacy bridge"
    This bridge is built on the deprecated `jarbas_hive_mind` stack rather than the
    current `hivemind-bus-client`. It is unmaintained and provided as-is.

## Install

There is no published package and no `setup.py`/`pyproject.toml`, so `pip install`
is not available. Clone the repo and install the runtime dependencies:

```bash
git clone https://github.com/JarbasHiveMind/HiveMind-twitch-bridge
cd HiveMind-twitch-bridge
pip install -r requirements.txt
```

## Configuration

Authentication is just the channel name plus a Twitch chat OAuth token (of the form
`oauth:...`, generated at [twitchapps.com/tmi](https://www.twitchapps.com/tmi/)).
No Twitch application / client ID is required.

There is no config file. Configuration is done by editing the
`connect_twitch_to_hivemind(...)` call at the bottom of
`twitch_bridge/__main__.py`:

```python
connect_twitch_to_hivemind(
    channel="my_channel",            # Twitch channel to join
    oauth="oauth:xxxxx",             # Twitch chat OAuth token
    tags=["@bot"],                   # trigger tags
    host="wss://127.0.0.1",          # HiveMind hub host
    port=5678,                       # HiveMind hub port
    key="dummy_key",                 # HiveMind access key
)
```

The reply trigger is whatever is in `tags`, defaulting to `["@bot"]`. There is no
rate limiting and the reply prefix is not configurable.

## Usage

Run the bridge as a module:

```bash
python -m twitch_bridge
```

A chat message is only relayed if it contains one of the trigger tags; the tag is
stripped before forwarding. Replies are posted back to the channel addressed to the
sender as `@user , <answer>`.

```
Viewer: @bot weather for seattle
Bot: @viewer , The weather in Seattle is...
```

## How it works

Twitch chat is read over IRC. A message containing a trigger tag becomes a
`recognizer_loop:utterance` OVOS message sent to the hub; the hub's `speak`
response is sent back to the channel as `@user , <answer>`.
