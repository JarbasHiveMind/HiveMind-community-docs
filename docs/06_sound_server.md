# HiveMind Sound Server

`hivemind-listener` extends `hivemind-core` and integrates with [ovos-simple-listener](https://github.com/TigreGotico/ovos-simple-listener), enabling audio-based communication with advanced features for **secure, distributed voice assistant functionality**.

> 💡 If you are not running a full OVOS install but instead a home server this is the best option


#### 🌟 Key Features of HiveMind Listener

- **Audio Stream Handling**:  
  Accepts encrypted binary audio streams, performing **WakeWord detection**, **Voice Activity Detection (VAD)**, **Speech-to-Text (STT)**, and **Text-to-Speech (TTS)** directly on the `hivemind-listener` instance.  
  *(Lightweight clients like [hivemind-mic-satellite](https://github.com/JarbasHiveMind/hivemind-mic-satellite) only run a microphone and VAD plugin.)*

- **STT Service**:  
  Provides **STT** via the [hivemind-websocket-client](https://github.com/JarbasHiveMind/hivemind-websocket-client), accepting Base64-encoded audio inputs.

- **TTS Service**:  
  Provides **TTS** via the [hivemind-websocket-client](https://github.com/JarbasHiveMind/hivemind-websocket-client), returning Base64-encoded audio outputs.

- **Secure Plugin Access**:  
  Running **TTS/STT via HiveMind Listener** requires an access key, offering fine-grained **access control** compared to non-authenticated server plugins.

#### 🚀 Getting Started with HiveMind Listener

1. **Install HiveMind Listener**:

```bash
pip install hivemind-listener
```

2. **Start the HiveMind Listener**:

```bash
$ hivemind-listener --help
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

This command will run the HiveMind Listener, with configurable plugins for wakeword detection, STT, TTS, and VAD, as well as access control via SSL.

---

#### Example Use Cases

1. **Microphone Satellite**:  
   Use [hivemind-mic-satellite](https://github.com/JarbasHiveMind/hivemind-mic-satellite) to stream raw audio to the `hivemind-listener`. Microphones handle audio capture and VAD, while the Listener manages WakeWord, STT, and TTS processing.

2. **Authenticated STT/TTS Services**:  
   Connect clients securely using access keys for transcribing or synthesizing audio via the HiveMind Listener, ensuring robust access control.
