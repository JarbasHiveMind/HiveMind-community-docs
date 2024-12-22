# HiveMind Voice Relay

OpenVoiceOS Relay, connect to [HiveMind](https://github.com/JarbasHiveMind/HiveMind-listener)

A lightweight version of [voice-satellite](https://github.com/JarbasHiveMind/HiveMind-voice-sat), but STT and TTS are sent to HiveMind instead of handled on device

## Server requirements

> ⚠️ `hivemind-listener` is required server side, the default `hivemind-core` does not provide STT and TTS capabilities.

> Alternatively run `hivemind-core` together with `ovos-audio` and `ovos-dinkum-listener`

The regular voice satellite is built on top of [ovos-dinkum-listener](https://github.com/OpenVoiceOS/ovos-dinkum-listener) and is full featured supporting all plugins

This repo is built on top of [ovos-simple-listener](https://github.com/TigreGotico/ovos-simple-listener), while it needs less resources it is also **missing** some features

- STT plugin
- TTS plugin
- Audio Transformers plugins
- Continuous Listening
- Hybrid Listening
- Recording Mode
- Sleep Mode
- Multiple WakeWords

If you need an even lighter implementation, consider [hivemind-mic-satellite](https://github.com/JarbasHiveMind/hivemind-mic-satellite) to also offload wake word to the server

## Install

Install with pip

```bash
$ pip install HiveMind-voice-relay
```

## Usage

```bash
Usage: hivemind-voice-relay [OPTIONS]

  connect to HiveMind

Options:
  --host TEXT      hivemind host
  --key TEXT       Access Key
  --password TEXT  Password for key derivation
  --port INTEGER   HiveMind port number
  --selfsigned     accept self signed certificates
  --help           Show this message and exit.

```

## Configuration

Voice relay is built on top of [ovos-simple-listener](https://github.com/TigreGotico/ovos-simple-listener) and [ovos-audio](https://github.com/OpenVoiceOS/ovos-audio), it uses the default OpenVoiceOS configuration `~/.config/mycroft/mycroft.conf`

Supported plugin types:

| Plugin Type | Description | Required | Link |
|-------------|-------------|----------|------|
| Microphone | Captures voice input | Yes | [Microphone](https://openvoiceos.github.io/ovos-technical-manual/mic_plugins/) |
| VAD | Voice Activity Detection | Yes | [VAD](https://openvoiceos.github.io/ovos-technical-manual/vad_plugins/) |
| WakeWord | Detects wake words for interaction | Yes | [WakeWord](https://openvoiceos.github.io/ovos-technical-manual/ww_plugins/) |
| G2P | grapheme-to-phoneme (G2P), used to simulate mouth movements  | No | [G2P](https://openvoiceos.github.io/ovos-technical-manual/g2p_plugins) |
| Media Playback Plugins | Enables media playback (e.g., "play Metallica") | No | [Media Playback Plugins](https://openvoiceos.github.io/ovos-technical-manual/media_plugins/) |
| OCP Plugins | Provides playback support for URLs (e.g., YouTube) | No | [OCP Plugins](https://openvoiceos.github.io/ovos-technical-manual/ocp_plugins/) |
| Dialog Transformers | Processes text before text-to-speech (TTS) | No | [Dialog Transformers](https://openvoiceos.github.io/ovos-technical-manual/transformer_plugins/) |
| TTS Transformers | Processes audio after text-to-speech (TTS) | No | [TTS Transformers](https://openvoiceos.github.io/ovos-technical-manual/transformer_plugins/) |
| PHAL | Provides platform-specific support (e.g., Mark 1) | No | [PHAL](https://openvoiceos.github.io/ovos-technical-manual/PHAL/) |
