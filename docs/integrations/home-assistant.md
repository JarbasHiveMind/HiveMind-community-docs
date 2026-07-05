# Home Assistant Integration

Home Assistant runs your house; your hive runs your voice. This integration introduces
them. Once it's installed, your OVOS device shows up in Home Assistant as a first-class
entity — play and pause its audio, nudge the volume, mute the mic, read its status, even
power-cycle it — right alongside your lights and thermostats. From HA's side it's just
another device on the dashboard; underneath, every button press rides the encrypted
HiveMind connection to the OVOS instance and back. It's a manual-install custom
integration ([hivemind-homeassistant](https://github.com/JarbasHiveMind/hivemind-homeassistant)),
added like any other from **Settings → Devices & Services**.

!!! abstract "In a nutshell"
    - A `custom_components/hivemind` integration installed by hand, then added via **Settings → Devices & Services**.
    - Its HiveMind client must have **admin privileges** and explicit access to a broad set of bus message types.
    - Grant those message types with `hivemind-core allow-msg` and `hivemind-core make-admin`, passing the node ID positionally.

---

## Installation

1. Copy the `hivemind` folder into your Home Assistant `custom_components` directory:

```bash
mkdir -p /config/custom_components
cp -r custom_components/hivemind /config/custom_components/
```

2. Restart Home Assistant.

3. Add the integration via **Settings → Devices & Services → Add Integration → HiveMind**.

---

## Required permissions

This is the one place this integration asks more of you than a chat bridge does — so
here's *why* before the long lists. A bridge only needs to hear `speak`. This integration
actually drives the device: it mutes the mic, reads whether a service is alive, changes
the volume, controls playback. Each of those is a distinct bus message, and HiveMind's
deny-by-default posture means you have to allow every one you want to use.

You don't have to grant them all — grant the ones behind the controls you care about. The
lists below are grouped by the OVOS service each message belongs to, so you can pick whole
capabilities at a time (skip the OCP block if you don't need media controls, skip the PHAL
blocks if you don't need volume or power). This client also needs the **admin** flag.

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
recognizer_loop:awoken
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
mycroft.audio.is_speaking
mycroft.audio.speak.status
```

!!! note "Readiness and speaking probes"
    The integration polls each service over its device type with
    `mycroft.<service>.is_alive` / `mycroft.<service>.is_ready` readiness probes — the
    `<service>` is one of `skills`, `voice`, `gui_service`, `audio`, or `PHAL`,
    depending on whether the device is registered as a *voice assistant* or a *media
    player*. The **Speaking** sensor additionally tracks `mycroft.audio.is_speaking`,
    so make sure that message type is allowed for the audio service.

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

### Audio Service

*(only if enabled manually — for systems without the OCP Audio Plugin)*

```
mycroft.audio.service.play
mycroft.audio.service.resume
mycroft.audio.service.pause
mycroft.audio.service.stop
mycroft.audio.service.prev
mycroft.audio.service.next
mycroft.audio.service.set_track_position
```

### PHAL

```
mycroft.phal.is_alive
mycroft.phal.is_ready
```

#### ovos-phal-plugin-alsa

```
mycroft.volume.get
mycroft.volume.set
mycroft.volume.increase
mycroft.volume.decrease
mycroft.volume.mute
mycroft.volume.unmute
```

#### ovos-phal-plugin-system

```
system.reboot
system.shutdown
system.mycroft.service.restart
system.ssh.status
system.ssh.enable
system.ssh.disable
```

#### ovos-phal-plugin-camera

*(work in progress)*

```
ovos.phal.camera.ping
ovos.phal.camera.get
ovos.phal.camera.open
ovos.phal.camera.close
```

However many of those you decided you need, you grant them the same way — one
`allow-msg` per message type, then the admin flag once at the end (the node ID goes last,
positionally):

```bash
hivemind-core allow-msg "speak" <id>
hivemind-core allow-msg "mycroft.volume.get" <id>
# ... repeat for each message type
hivemind-core make-admin <id>
```

---

## Related projects

- [Media Player](media-player.md) — turn any device into a standalone HiveMind OCP player
- [ovos-skill-music-assistant](https://github.com/HiveMindInsiders/ovos-skill-music-assistant) — OVOS skill for Music Assistant media search
- [ovos-media-plugin-mass](https://github.com/HiveMindInsiders/ovos-media-plugin-mass) — OVOS plugin to control Music Assistant players

---

## Source

Validated against the HiveMind source:

- [`custom_components/hivemind/__init__.py`](https://github.com/JarbasHiveMind/hivemind-homeassistant/blob/HEAD/custom_components/hivemind/__init__.py) — the `hivemind` domain and bus client setup
- [`custom_components/hivemind/config_flow.py`](https://github.com/JarbasHiveMind/hivemind-homeassistant/blob/HEAD/custom_components/hivemind/config_flow.py) — config-flow fields (host, access_key, password, port, allow_self_signed, device_type)
- [`custom_components/hivemind/binary_sensor.py`](https://github.com/JarbasHiveMind/hivemind-homeassistant/blob/HEAD/custom_components/hivemind/binary_sensor.py) — connection / speaking / alive / ready sensors and their probes
- [`custom_components/hivemind/switch.py`](https://github.com/JarbasHiveMind/hivemind-homeassistant/blob/HEAD/custom_components/hivemind/switch.py) — SSH, volume mute, mic mute, and sleep-mode switches
- [`custom_components/hivemind/media_player.py`](https://github.com/JarbasHiveMind/hivemind-homeassistant/blob/HEAD/custom_components/hivemind/media_player.py) — OCP media-player entity
