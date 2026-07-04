# Media Player

[hivemind-media-player](https://github.com/JarbasHiveMind/hivemind-media-player) turns
a headless device into a **network-controlled media player** driven over HiveMind.
Remote controllers send standard [OCP](../reference/glossary.md#ocp-ovos-common-play)
playback commands across the encrypted HiveMind connection, and this device plays the
audio locally.

!!! tip "Beginner's mental model"
    Turn a Raspberry Pi or a spare speaker into a player you control from your hive.
    Tell it to play a track from anywhere in your network — it does the actual
    playing, you do the controlling.

This is for devices that are **not** running a full voice assistant — a dedicated
networked speaker, for example. It complements the
[Home Assistant integration](home-assistant.md): with both installed, your HiveMind
player devices show up as media players in Home Assistant (and Music Assistant can
browse and play music to them).

## How it fits in

This is a HiveMind **agent-protocol plugin**, not an OVOS skill. The installed package
is `hivemind-player-protocol`, which registers a `hivemind.agent.protocol` entry point
exposing the `HiveMindPlayerProtocol` class. When `hivemind-core` loads it, the
protocol boots a local `ovos-audio` stack (TTS + OCP playback, plus `ovos-PHAL` if
available) and routes incoming playback messages to it.

??? note "Advanced: why an agent protocol and not a skill"
    The player answers **no** natural-language questions — its
    `natural_language_query` declines so the node escalates such queries upstream to a
    real agent. Its only job is to receive OCP/audio bus messages forwarded by
    `hivemind-core` and play them through the local audio stack. Running it as an agent
    protocol inside `hivemind-core` (rather than a skill inside `ovos-core`) is what
    lets a device with no assistant of its own act purely as a remote player.

## Install

On the device that will become the player:

```bash
pip install hivemind-player-protocol
```

For the VLC / MPV playback backends, install the extras:

```bash
pip install "hivemind-player-protocol[extras]"
```

Then enable the protocol in `hivemind-core` by setting it as the
`agent_protocol` in `~/.config/hivemind-core/server.json`, configure `ovos-audio`,
register a client credential, and start `hivemind-core listen`. See the repository
README for the full step-by-step quickstart and the bundled `hivemind-player-ctl`
control CLI.

## Permissions

The player client needs at minimum the core audio and OCP playback messages, for
example:

```bash
hivemind-core allow-msg "speak" <id>
hivemind-core allow-msg "ovos.common_play.play" <id>
hivemind-core allow-msg "ovos.common_play.pause" <id>
hivemind-core allow-msg "ovos.common_play.resume" <id>
hivemind-core allow-msg "ovos.common_play.stop" <id>
hivemind-core allow-msg "ovos.common_play.next" <id>
hivemind-core allow-msg "ovos.common_play.previous" <id>
```

The repository's permissions reference lists the full OCP, audio, and (optional) PHAL
volume message sets.

## Related projects

- [hivemind-homeassistant](home-assistant.md) — exposes HiveMind player devices as Home Assistant media players
- [ovos-skill-music-assistant](https://github.com/HiveMindInsiders/ovos-skill-music-assistant) — OVOS skill for Music Assistant media search
- [ovos-media-plugin-mass](https://github.com/HiveMindInsiders/ovos-media-plugin-mass) — OVOS plugin to control Music Assistant players

## Source

Validated against the HiveMind source:

- [`README.md`](https://github.com/JarbasHiveMind/hivemind-media-player/blob/HEAD/README.md) — architecture, install, quickstart, and permissions reference
- [`pyproject.toml`](https://github.com/JarbasHiveMind/hivemind-media-player/blob/HEAD/pyproject.toml) — the `hivemind-player-protocol` package and its `hivemind.agent.protocol` entry point
