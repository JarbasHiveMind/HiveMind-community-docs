## Creating a Hive

### Minds

When creating a Hive the first step is to add a Mind to it, this is "the brain" of the hive and is expected to handle natural language queries

- [hivemind-core](https://github.com/JarbasHiveMind/HiveMind-core/) -> connects to an existing ovos-core setup
- [hivemind-persona](https://github.com/JarbasHiveMind/hivemind-persona) -> launch a OVOS persona behind a hivemind server, this can for example be ChatGPT
- [MultiMind](https://github.com/JarbasHiveMind/MultiMind) -> launch a dedicated ovos-core per access key
- [LocalHive](https://github.com/JarbasHiveMind/LocalHive) -> a mind that loads OVOS skills fully isolated, usable on localhost only
- [DuckDuckGo](https://github.com/JarbasHiveMind/Fakecroft-DDG) -> proof of concept of how to turn anything into a Mind

Once you have at least 1 Mind in your hive you can start connecting things to it!

![imagem](https://github.com/JarbasHiveMind/HiveMind-community-docs/assets/33701864/fb241c4d-ca84-4b47-b917-b398b16f93bd)


### Terminals / Satellites

A terminal is something that allows you to interact with your hive, it connects to a Mind setup in the previous step

- [Voice Satellite](https://github.com/OpenJarbas/HiveMind-voice-sat) - processes audio on satellite and sends natural language queries to a mind
- [Webchat](https://github.com/OpenJarbas/HiveMind-webchat) -> webchat to connect from browser directly to a mind
- [Flask Chatroom](https://github.com/JarbasHiveMind/HiveMind-flask-template) - reference implementation for a web application that connects to a mind backend side
- [Remote Cli](https://github.com/OpenJarbas/HiveMind-cli) - a command line application to chat with a Mind


### Bridges

A bridge connects some existing service to a Mind, it is like a terminal but depends on some intermediate service

- [Matrix Bridge](https://github.com/JarbasHiveMind/HiveMind-matrix-bridge)
- [Mattermost Bridge](https://github.com/OpenJarbas/HiveMind_mattermost_bridge)
- [HackChat Bridge](https://github.com/OpenJarbas/HiveMind-HackChatBridge)
- [Twitch Bridge](https://github.com/OpenJarbas/HiveMind-twitch-bridge)
- [DeltaChat Bridge](https://github.com/JarbasHiveMind/HiveMind-deltachat-bridge)

