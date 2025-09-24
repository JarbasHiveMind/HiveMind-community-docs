# Binary Plugins for HiveMind

While **Agent Plugins** handle text-based messages, HiveMind also supports **Binary Plugins** for transmitting and processing **binary payloads** such as audio streams.

These plugins make it possible to provide **secure services** like **Speech-to-Text (STT)** and **Text-to-Speech (TTS)** across HiveMind without exposing them directly.

Importantly, **transport always happens securely over HiveMind**, binary data uses the same encrypted HiveMind transport as all other messages.

---

## What is a Binary Plugin?

A **Binary Plugin** defines how HiveMind nodes exchange and process **non-text payloads**.
They allow nodes to forward microphone input, receive synthesized audio, or otherwise handle binary data within the Hive.

Unlike agent plugins, which process BUS messages, binary plugins work directly with **binary streams**.

---

## Current Implementation

The reference plugin is the [hivemind-audio-binary-protocol](https://github.com/JarbasHiveMind/hivemind-audio-binary-protocol), built on top of OpenVoiceOS plugins

Right now, the binary plugin system is focused on **STT and TTS**, but the design allows for more use cases in the future.

* The first binary plugin implementation loads existing **OpenVoiceOS TTS and STT plugins**.
* This makes HiveMind immediately compatible with **hundreds of plugins in the OVOS ecosystem**, without requiring special adaptations.
* Support also exists for **microphone plugins**:
  * Audio streams can be sent to the HiveMind server when **VAD (Voice Activity Detection)** triggers.
  * In this mode, the **wake word detection** also runs centrally on the HiveMind server.

This architecture keeps HiveMind satellites lightweight while delegating heavy-lifting tasks to a hub or powerful node.

---

## Example Use Cases

### 🔹 Speech-to-Text (STT)

1. Satellite streams microphone audio as a binary payload.
2. A HiveMind server running an STT plugin transcribes the audio.
3. Transcription is returned as a standard text BUS message to the satellite.

### 🔹 Text-to-Speech (TTS)

1. Node sends a text string to a HiveMind TTS-capable server.
2. The TTS service generates speech audio.
3. Audio is returned as a binary payload, ready for playback on the requesting node.

### 🔹 Microphone Plugins

* HiveMind nodes can act as remote microphones.
* Audio is streamed upstream only when **speech is detected** (via VAD).
* Wake word detection is centralized on the HiveMind server.

---

## Why Binary Plugins Matter

* ✅ **Secure transport** — all binary data flows over HiveMind’s encrypted transport.
* ✅ **Ecosystem compatibility** — directly supports existing OVOS STT/TTS plugins.
* ✅ **Lightweight satellites** — offload heavy STT/TTS to a hub.
* ✅ **Flexibility** — microphone streaming, wake word offloading, and future binary tasks (images, embeddings, etc.).

---

👉 In short:

* **Agent Plugins** → handle text input/output.
* **Binary Plugins** → securely handle binary payloads (audio streams) over HiveMind, enabling distributed STT/TTS and microphone offloading.

