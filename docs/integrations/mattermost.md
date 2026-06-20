# Mattermost Bridge

[HiveMind-mattermost-bridge](https://github.com/JarbasHiveMind/HiveMind_mattermost_bridge) connects a Mattermost server to a HiveMind hub. Users can interact with the AI assistant through direct messages or channel mentions.

!!! note "Legacy bridge"
    This bridge is built on the deprecated `jarbas_hive_mind` stack rather than the
    current `hivemind-bus-client`. It is unmaintained and provided as-is.

## Install

There is no published package and no `setup.py`/`pyproject.toml`, so `pip install`
is not available. Clone the repo and install the runtime dependencies:

```bash
git clone https://github.com/JarbasHiveMind/HiveMind_mattermost_bridge
cd HiveMind_mattermost_bridge
pip install -r requirements.txt
```

## Configuration

There is no config file. Configuration is done by editing the
`connect_mattermost_to_hivemind(...)` call at the bottom of
`mattermost_bridge/__main__.py`:

```python
connect_mattermost_to_hivemind(
    mail="bot@example.com",   # Mattermost login (mattermostdriver auth)
    pswd="bot-password",      # Mattermost password
    url="chat.example.com",   # Mattermost server host (no scheme)
    tags=["@bot"],            # trigger tags for channel mentions
    host="127.0.0.1",         # HiveMind hub host
    port=5678,                # HiveMind hub port
    key="unsafe",             # HiveMind access key
)
```

Authentication is a Mattermost user login (`mail`/`pswd` via `mattermostdriver`),
not a bot token. The channel mention trigger is whatever is in `tags`, defaulting
to `["@bot"]`.

The bridge connects to Mattermost as an outbound websocket client
(`mattermostdriver` `init_websocket`) — it does **not** run an HTTP server, so no
outgoing webhook or listener port needs to be configured on the Mattermost side.

## Usage

Run the bridge as a module:

```bash
python -m mattermost_bridge
```

**Direct message the bot:**

```
User: hello
HiveMind Bot: Hello! How can I help?
```

**Mention the bot in a channel** (using a tag from `tags`):

```
User: @bot what's the weather?
```

Both direct messages and channel mentions are handled.

## How it works

Each Mattermost user maps to a unique HiveMind peer. Incoming direct messages and
mentions become `recognizer_loop:utterance` OVOS messages sent to the hub.
Responses (`speak`) are posted back to the originating conversation.
