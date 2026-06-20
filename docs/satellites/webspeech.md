# WebSpeech Browser Satellite

A JavaScript-based satellite that runs in any modern web browser. Captures microphone audio in the browser and streams it to the hub for processing.

**What runs locally**: Browser microphone + VAD (JavaScript)

**What the hub provides**: STT, TTS, skills, intents

## When to use it

- Web applications or chat interfaces
- Temporary client access from any device with a browser
- Users already in a browser context

## Requirements

The hub must have `hivemind-audio-binary-protocol` installed and configured for STT and TTS.

## Installation

No installation required on the client side. Include the [HiveMind.js](https://github.com/JarbasHiveMind/HiveMind-js) library in your HTML.

## Resources

- [hivemind-webspeech](https://github.com/JarbasHiveMind/hivemind-webspeech) — reference implementation
- [HiveMind-js](https://github.com/JarbasHiveMind/HiveMind-js) — JavaScript client library
- [hivemind-flask-chatroom](https://github.com/JarbasHiveMind/hivemind-flask-chatroom) — Flask template for a browser-based chatroom

## Limitations

- Browser VAD capabilities vary by browser and platform
- Requires JavaScript support
- Microphone access requires user permission (browser security model)
- No wakeword — capture is VAD-gated push-to-talk via a VAD toggle button
