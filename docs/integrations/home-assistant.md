# Home Assistant Integration

[hivemind-homeassistant](https://github.com/JarbasHiveMind/hivemind-homeassistant) is a manual-install Home Assistant custom integration that connects Home Assistant to an OVOS instance via HiveMind.

It exposes the OVOS device as a Home Assistant entity with controls for audio playback, volume, microphone state, system power, and status sensors.

## Installation

1. Copy the `hivemind` folder into your Home Assistant `custom_components` directory:

```bash
mkdir -p /config/custom_components
cp -r custom_components/hivemind /config/custom_components/
```

2. Restart Home Assistant.

3. Add the integration via **Settings → Devices & Services → Add Integration → HiveMind**.

## Required permissions

This integration does more than send voice queries — it injects and reads bus messages at a system level. The HiveMind client used by this integration must have **admin privileges** and must be granted explicit access to the following message types.

### ovos-core

```
mycroft.stop
mycroft.skills.is_alive
mycroft.skills.is_ready
```

### ovos-dinkum-listener

```
mycroft.voice.is_alive
mycroft.voice.is_ready
mycroft.mic.listen
mycroft.mic.mute
mycroft.mic.unmute
mycroft.mic.get_status
recognizer_loop:sleep
recognizer_loop:wake_up
recognizer_loop:state.get
recognizer_loop:state.set
```

### ovos-gui

```
mycroft.gui_service.is_alive
mycroft.gui_service.is_ready
```

### ovos-audio

```
speak
mycroft.audio.is_alive
mycroft.audio.is_ready
mycroft.audio.speak.status
```

### OCP (OpenVoiceOS Common Play)

```
ovos.common_play.player.status
ovos.common_play.track_info
ovos.common_play.get_track_length
ovos.common_play.get_track_position
ovos.common_play.playlist.queue
ovos.common_play.resume
ovos.common_play.pause
ovos.common_play.stop
ovos.common_play.previous
ovos.common_play.next
ovos.common_play.set_track_position
ovos.common_play.playlist.clear
ovos.common_play.shuffle.set
ovos.common_play.shuffle.unset
ovos.common_play.repeat.set
ovos.common_play.repeat.unset
ovos.common_play.repeat.one
```

### PHAL — volume control (ovos-phal-plugin-alsa)

```
mycroft.volume.get
mycroft.volume.increase
mycroft.volume.decrease
mycroft.volume.mute
mycroft.volume.unmute
```

### PHAL — system control (ovos-phal-plugin-system)

```
system.reboot
system.shutdown
system.mycroft.service.restart
system.ssh.status
```

### PHAL — status

```
mycroft.phal.is_alive
mycroft.phal.is_ready
```

Grant these permissions using `hivemind-core allow-msg` for the client node ID associated with this integration:

```bash
hivemind-core allow-msg "speak" --node-id <id>
hivemind-core allow-msg "mycroft.volume.get" --node-id <id>
# ... repeat for each message type
hivemind-core make-admin --node-id <id>
```

## Related projects

- [hivemind-player-protocol](https://github.com/HiveMindInsiders/hivemind-player-protocol) — turn any device into a standalone HiveMind OCP player
- [ovos-skill-music-assistant](https://github.com/HiveMindInsiders/ovos-skill-music-assistant) — OVOS skill for Music Assistant media search
- [ovos-media-plugin-mass](https://github.com/HiveMindInsiders/ovos-media-plugin-mass) — OVOS plugin to control Music Assistant players
