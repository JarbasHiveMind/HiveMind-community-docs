# HiveMind Sound Server

`hivemind-listener` extends `hivemind-core` and integrates with [ovos-simple-listener](https://github.com/TigreGotico/ovos-simple-listener), enabling audio-based communication for a secure, distributed voice assistant setup.

> If you are running a home server, this is the best option. You only need to install `hivemind-listener`, `ovos-core`, and `ovos-messagebus`.

> If you run on a device that is also a full OVOS assistant by itself, use `hivemind-core` instead.

#### Key features of HiveMind Listener

- **Audio stream handling**: accepts encrypted binary audio streams, and performs wake word detection, Voice Activity Detection (VAD), Speech-to-Text (STT), and Text-to-Speech (TTS) directly on the `hivemind-listener` instance. Lightweight clients like [hivemind-mic-satellite](https://github.com/JarbasHiveMind/hivemind-mic-satellite) run only a microphone and VAD plugin.
- **STT service**: provides STT through the [hivemind-websocket-client](https://github.com/JarbasHiveMind/hivemind-websocket-client), accepting Base64-encoded audio input.
- **TTS service**: provides TTS through the [hivemind-websocket-client](https://github.com/JarbasHiveMind/hivemind-websocket-client), returning Base64-encoded audio output.
- **Secure plugin access**: running TTS/STT through HiveMind Listener requires an access key, giving finer-grained access control than a non-authenticated server plugin.

#### Usage

1. Install HiveMind Listener:

```bash
pip install hivemind-listener
```

2. Start the HiveMind Listener:

```bash
hivemind-listener --help
Usage: hivemind-listener [OPTIONS]

  Run the HiveMind Listener with configurable plugins.

  If a plugin is not specified, the defaults from mycroft.conf will be used.
  mycroft.conf will be loaded as usual for plugin settings.

Options:
  --wakeword TEXT                 Specify the wake word for the listener.
                                  Default is 'hey_mycroft'.
  --stt-plugin TEXT               Specify the STT plugin to use.
  --tts-plugin TEXT               Specify the TTS plugin to use.
  --vad-plugin TEXT               Specify the VAD plugin to use.
  --dialog-transformers TEXT      dialog transformer plugins to load.
                                  Installed plugins: None
  --utterance-transformers TEXT   utterance transformer plugins to load. 
                                  Installed plugins: ['ovos-utterance-plugin-cancel']
  --metadata-transformers TEXT    metadata transformer plugins to
                                  load. Installed plugins: None
  --ovos_bus_address TEXT         Open Voice OS bus address
  --ovos_bus_port INTEGER         Open Voice OS bus port number
  --host TEXT                     HiveMind host
  --port INTEGER                  HiveMind port number
  --ssl BOOLEAN                   use wss://
  --cert_dir TEXT                 HiveMind SSL certificate directory
  --cert_name TEXT                HiveMind SSL certificate file name
  --db-backend [redis|json|sqlite]
                                  Select the database backend to use. Options:
                                  redis, sqlite, json.
  --db-name TEXT                  [json/sqlite] The name for the database
                                  file. ~/.cache/hivemind-core/{name}
  --db-folder TEXT                [json/sqlite] The subfolder where database
                                  files are stored. ~/.cache/{db_folder}}
  --redis-host TEXT               [redis] Host for Redis. Default is
                                  localhost.
  --redis-port INTEGER            [redis] Port for Redis. Default is 6379.
  --redis-password TEXT           [redis] Password for Redis. Default None
  --help                          Show this message and exit.
```

This command runs the HiveMind Listener, with configurable plugins for wake word detection, STT, TTS, and VAD, plus access control over SSL.

---

#### Example use cases

1. **Microphone Satellite**: use [hivemind-mic-satellite](https://github.com/JarbasHiveMind/hivemind-mic-satellite) to stream raw audio to `hivemind-listener`. The microphone satellite handles audio capture and VAD, while the Listener manages wake word detection, STT, and TTS.

2. **Authenticated STT/TTS services**: connect clients securely with access keys to transcribe or synthesize audio through the HiveMind Listener, with access control enforced.

---
[Home](index.md)
