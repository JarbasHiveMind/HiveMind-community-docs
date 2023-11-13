# HiveMind Voice Satellite

OpenVoiceOS Satellite, connect to HiveMind

Built on top of [ovos-dinkum-listener](https://github.com/OpenVoiceOS/ovos-dinkum-listener), [ovos-audio](https://openvoiceos.github.io/ovos-technical-manual/audio_service/) and [PHAL](https://openvoiceos.github.io/ovos-technical-manual/PHAL/)

![img_19.png](img_19.png)

## Install

Install dependencies (if needed)

```bash
sudo apt-get install -y libpulse-dev libasound2-dev
```

Install with pip (hivemind pypi version is VERY outdated)

```bash
$ pip install git+https://github.com/JarbasHiveMind/HiveMind-voice-sat
```

## Usage

```bash
Usage: hivemind-voice-sat [OPTIONS]

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

Voice satellite uses the default OpenVoiceOS configuration `~/.config/mycroft/mycroft.conf`
    
See configuration from [ovos-dinkum-listener](https://github.com/OpenVoiceOS/ovos-dinkum-listener) for default values

### Plugins

Voice Sat supports plugins for:

- Microphone

- [VAD](https://openvoiceos.github.io/ovos-technical-manual/vad_plugins/)

- [WakeWord](https://openvoiceos.github.io/ovos-technical-manual/ww_plugins/)

- [STT](https://openvoiceos.github.io/ovos-technical-manual/stt_plugins/)

- Audio Transformers

- Dialog Transformers

- [G2P](https://openvoiceos.github.io/ovos-technical-manual/g2p_plugins/)

- [TTS](https://openvoiceos.github.io/ovos-technical-manual/tts_plugins/)

- [Audio Service](https://openvoiceos.github.io/ovos-technical-manual/audio_plugins/)

- TTS Transformers

- [PHAL](https://openvoiceos.github.io/ovos-technical-manual/PHAL/)


You can optimize your voice satellite for a variety of platforms by selecting different plugin combinations