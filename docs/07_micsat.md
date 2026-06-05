# HiveMind Microphone Satellite

OpenVoiceOS Microphone Satellite for offloading audio processing to a central hub.

A super lightweight version of [HiveMind-voice-sat](https://github.com/JarbasHiveMind/HiveMind-voice-sat), only Microphone and VAD plugins run on the mic-satellite. Voice activity is streamed to the hub and all speech processing (STT/TTS/wakeword) happens on the server.

## Server requirements

> **Note**: The hub must provide STT and TTS capabilities. You can either:
> - Run `hivemind-core` with the `hivemind-audio-binary-protocol` binary plugin (recommended)
> - Run `hivemind-core` with OVOS and `ovos-audio` + `ovos-dinkum-listener` for full voice processing

## Install

Install with pip

```bash
$ pip install hivemind-mic-satellite
```

## Configuration

Voice relay is built on top of [ovos-plugin-manager](https://github.com/OpenVoiceOS/ovos-plugin-manager), it uses the same OpenVoiceOS configuration `~/.config/mycroft/mycroft.conf`

Supported plugins:

| Plugin Type            | Description                                        | Required | Link                                                                                                       |
|------------------------|----------------------------------------------------|----------|------------------------------------------------------------------------------------------------------------|
| Microphone             | Captures voice input                               | Yes      | [Microphone](https://openvoiceos.github.io/ovos-technical-manual/mic_plugins/)                             |
| VAD                    | Voice Activity Detection                           | Yes      | [VAD](https://openvoiceos.github.io/ovos-technical-manual/vad_plugins/)                                    |
| PHAL                   | Platform/Hardware Abstraction Layer                | No       | [PHAL](https://openvoiceos.github.io/ovos-technical-manual/PHAL/)                                          |
| G2P                    | Generate visemes (mouth movements), eg. for Mk1    | No       | [G2P](https://openvoiceos.github.io/ovos-technical-manual/g2p_plugins/)                                    |
| Media Playback Plugins | Enables media playback (e.g., "play Metallica")    | No       | [Media Playback Plugins](https://openvoiceos.github.io/ovos-technical-manual/media_plugins/)               |
| OCP Plugins            | Provides playback support for URLs (e.g., YouTube) | No       | [OCP Plugins](https://openvoiceos.github.io/ovos-technical-manual/ocp_plugins/)                            |

The regular voice satellite is built on top of [ovos-dinkum-listener](https://github.com/OpenVoiceOS/ovos-dinkum-listener) and is full featured supporting all plugins

This repo needs less resources but it is also **missing** some features

- STT plugin (runs on server)
- TTS plugin (runs on server)
- WakeWord plugin (runs on server)
- Continuous Listening
- Hybrid Listening
- Recording Mode
- Sleep Mode
- Multiple WakeWords
- Audio Transformers plugins
- Dialog Transformers plugins
