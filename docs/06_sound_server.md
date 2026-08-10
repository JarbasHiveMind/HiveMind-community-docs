# HiveMind Sound Server

The standalone `hivemind-listener` package is superseded. Its last release, 2.0.0 from
December 2024, pins `hivemind-core<3.0.0`, and the current core is 4.x, so you cannot
install it beside a current hub.

Audio offloading, which covers streamed microphone audio, STT and TTS, is now
`hivemind-core` with a binary protocol plugin loaded.

## Install

```bash
pip install hivemind-core hivemind-audio-binary-protocol
```

## Configure

Point `binary_protocol` at the plugin in `~/.config/hivemind-core/server.json`:

```json
{
  "binary_protocol": {
    "module": "hivemind-audio-binary-protocol-plugin",
    "hivemind-audio-binary-protocol-plugin": {"wakeword": "hey_mycroft"}
  }
}
```

Then start the hub with `hivemind-core listen`.

Satellites that stream audio, such as [Mic Satellite](07_micsat.md) and
[Voice Relay](07_voice_relay.md), need this plugin on the hub. A satellite that runs its
own STT, such as [Voice Satellite](07_voicesat.md), does not.

See [Binary Plugins](binary_plugins.md) and [Binarization](18_binarization.md).

---
[Home](index.md)
