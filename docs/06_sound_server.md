# HiveMind Sound Server

> ℹ️ The standalone `hivemind-listener` package is **superseded**. Its last release
> (2.0.0, December 2024) pins `hivemind-core<3.0.0`, while the current core is 4.x, so it cannot be
> installed alongside a current hub. Do not use it for new deployments.

Audio offloading — streamed microphone audio, STT and TTS — is now `hivemind-core`
with a **binary protocol plugin** loaded.

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
    "hivemind-audio-binary-protocol-plugin": {
      "wakeword": "hey_mycroft"
    }
  }
}
```

Then start the hub as usual:

```bash
hivemind-core listen
```

Satellites that stream audio — [Mic Satellite](./07_micsat.md) and
[Voice Relay](./07_voice_relay.md) — need this plugin on the hub. A satellite that
does its own STT, such as [Voice Satellite](./07_voicesat.md), does not.

See [Binary Plugins](./binary_plugins.md) for the plugin reference and
[Binarization](./18_binarization.md) for the wire format.
