
## OVOS Plugins Compatibility

Hivemind leverages [ovos-plugin-manager](), bringing compatibility with hundreds of plugins.

> 💡 **Tip**: OVOS plugins can be used both on *client* and *server* side

| Plugin Type         | Description                                             | Documentation                                                                                   |
|---------------------|---------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Microphone          | Captures voice input                                    | [Microphone Documentation](https://openvoiceos.github.io/ovos-technical-manual/mic_plugins/)    |
| VAD                 | Voice Activity Detection                                | [VAD Documentation](https://openvoiceos.github.io/ovos-technical-manual/vad_plugins/)           |
| WakeWord            | Detects wake words for interaction                      | [WakeWord Documentation](https://openvoiceos.github.io/ovos-technical-manual/ww_plugins/)       |
| STT                 | Speech-to-text (STT)                                    | [STT Documentation](https://openvoiceos.github.io/ovos-technical-manual/stt_plugins/)           |
| TTS                 | Text-to-speech (TTS)                                    | [TTS Documentation](https://openvoiceos.github.io/ovos-technical-manual/tts_plugins)            |
| G2P                 | Grapheme-to-phoneme (G2P), used to simulate mouth movements | [G2P Documentation](https://openvoiceos.github.io/ovos-technical-manual/g2p_plugins)           |
| Media Playback      | Enables media playback (e.g., "play Metallica")         | [Media Playback Documentation](https://openvoiceos.github.io/ovos-technical-manual/media_plugins/) |
| OCP Plugins         | Provides playback support for URLs (e.g., YouTube)      | [OCP Plugins Documentation](https://openvoiceos.github.io/ovos-technical-manual/ocp_plugins/)   |
| Audio Transformers  | Processes audio before speech-to-text (STT)             | [Audio Transformers Documentation](https://openvoiceos.github.io/ovos-technical-manual/transformer_plugins/) |
| Dialog Transformers | Processes text before text-to-speech (TTS)              | [Dialog Transformers Documentation](https://openvoiceos.github.io/ovos-technical-manual/transformer_plugins/) |
| TTS Transformers    | Processes audio after text-to-speech (TTS)              | [TTS Transformers Documentation](https://openvoiceos.github.io/ovos-technical-manual/transformer_plugins/) |
| PHAL                | Provides platform-specific support (e.g., Mark 1)       | [PHAL Documentation](https://openvoiceos.github.io/ovos-technical-manual/PHAL/)                |

### Client side plugins

The tables below illustrates how plugins from the OVOS ecosystem relate to the various satellites and where they should
be installed and configured

**Audio input**:

| Supported Plugins                 | **Microphone**   | **VAD**          | **Wake Word**      | **STT**          |
|-----------------------------------|------------------|------------------|--------------------|------------------|
| **HiveMind Voice Satellite**      | ✔️<br>(Required) | ✔️<br>(Required) | ✔️<br>(Required *) | ✔️<br>(Required) | 
| **HiveMind Voice Relay**          | ✔️<br>(Required) | ✔️<br>(Required) | ✔️<br>(Required)   | 📡<br>(Remote)   | 
| **HiveMind Microphone Satellite** | ✔️<br>(Required) | ✔️<br>(Required) | 📡<br>(Remote)     | 📡<br>(Remote)   | 

* can be skipped
  with [continuous listening mode](https://openvoiceos.github.io/ovos-technical-manual/speech_service/#modes-of-operation)

**Audio output**:

| Supported Plugins                 | **TTS**          | **Media Playback** | **OCP extractors** | 
|-----------------------------------|------------------|--------------------|--------------------| 
| **HiveMind Voice Satellite**      | ✔️<br>(Required) | ✔️<br>(Optional)   | ✔️<br>(Optional)   |  
| **HiveMind Voice Relay**          | 📡<br>(Remote)   | ✔️<br>(Optional)   | ✔️<br>(Optional)   | 
| **HiveMind Microphone Satellite** | 📡<br>(Remote)   | ✔️<br>(Optional)   | ✔️<br>(Optional)   |  

**Transformers**:

| Supported Plugins                 | **Audio**          | **Utterance**      | **Metadata**       | **Dialog**         | **TTS**            |
|-----------------------------------|--------------------|--------------------|--------------------|--------------------|--------------------|
| **HiveMind Voice Satellite**      | ✔️<br>(Optional)   | ✔️<br>(Optional)   | ✔️<br>(Optional)   | ✔️<br>(Optional)   | ✔️<br>(Optional)   |
| **HiveMind Voice Relay**          | ❌<br>(Unsupported) | 🚧<br>(TODO)       | 🚧<br>(TODO)       | 🚧<br>(TODO)       | ❌<br>(Unsupported) |
| **HiveMind Microphone Satellite** | ❌<br>(Unsupported) | ❌<br>(Unsupported) | ❌<br>(Unsupported) | ❌<br>(Unsupported) | ❌<br>(Unsupported) |

**Other**:

| Supported Plugins                 | **G2P**<br>(mouth movements) | **PHAL**         |
|-----------------------------------|------------------------------|------------------|
| **HiveMind Voice Satellite**      | ✔️<br>(Optional)             | ✔️<br>(Optional) |
| **HiveMind Voice Relay**          | ❌<br>(Unsupported)           | ✔️<br>(Optional) |
| **HiveMind Microphone Satellite** | ❌<br>(Unsupported)           | ✔️<br>(Optional) |


### Server side plugins

The tables below illustrates how plugins from the OVOS ecosystem relate to the various server setups and where they should
be installed and configured

**Audio input**:

| Supported Plugins           | **Microphone**     | **VAD**            | **Wake Word**      | **STT**            |
|-----------------------------|--------------------|--------------------|--------------------|--------------------|
| **Hivemind Skills Server**  | ❌<br>(Unsupported) | ❌<br>(Unsupported) | ❌<br>(Unsupported) | ❌<br>(Unsupported) | 
| **Hivemind Sound Server**   | ✔️<br>(Required)   | ✔️<br>(Required)   | ✔️<br>(Required)   | ✔️<br>(Required)   | 
| **Hivemind Persona Server** | ❌<br>(Unsupported) | ❌<br>(Unsupported) | ❌<br>(Unsupported) | ❌<br>(Unsupported) | 

**Audio output**:

| Supported Plugins           | **TTS**            | **Media Playback** | **OCP extractors** | 
|-----------------------------|--------------------|--------------------|--------------------| 
| **Hivemind Skills Server**  | ❌<br>(Unsupported) | ❌<br>(Unsupported) | ✔️<br>(Optional)   |  
| **Hivemind Sound Server**   | ✔️<br>(Required)   | ❌<br>(Unsupported) | ✔️<br>(Optional)   | 
| **Hivemind Persona Server** | ❌<br>(Unsupported) | ❌<br>(Unsupported) | ❌<br>(Unsupported) |  

**Transformers**:

| Supported Plugins           | **Audio**          | **Utterance**      | **Metadata**       | **Dialog**         | **TTS**            |
|-----------------------------|--------------------|--------------------|--------------------|--------------------|--------------------|
| **Hivemind Skills Server**  | ❌<br>(Unsupported) | ❌<br>(Unsupported) | ❌<br>(Unsupported) | ❌<br>(Unsupported) | ❌<br>(Unsupported) |
| **Hivemind Sound Server**   | 🚧<br>(TODO)       | ✔️<br>(Optional)   | ✔️<br>(Optional)   | ✔️<br>(Optional)   | 🚧<br>(TODO)       |
| **Hivemind Persona Server** | ❌<br>(Unsupported) | 🚧<br>(TODO)       | ❌<br>(Unsupported) | 🚧<br>(TODO)       | ❌<br>(Unsupported) |

**Other**:

| Supported Plugins           | **G2P**<br>(mouth movements) | **PHAL**           |
|-----------------------------|------------------------------|--------------------|
| **Hivemind Skills Server**  | ❌<br>(Unsupported)           | ❌<br>(Unsupported) |
| **Hivemind Sound Server**   | ❌<br>(Unsupported)           | ❌<br>(Unsupported) |
| **Hivemind Persona Server** | ❌<br>(Unsupported)           | ❌<br>(Unsupported) |

