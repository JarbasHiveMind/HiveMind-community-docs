# WebSpeech Browser Satellite

A lightweight browser-native voice satellite that turns any web browser into a HiveMind voice-activated client. No server installation needed — runs entirely in the browser using Web Audio API and WebSpeech.

**Source**: `hivemind-webspeech` package

## Overview

`hivemind-webspeech` is a JavaScript/HTML5 application that:
- Listens for speech in the browser using WebSpeech API (fallback to Web Audio API + Silero VAD)
- Detects wake words locally using ONNX-based Precise engine
- Streams audio to the HiveMind hub for STT/intent processing
- Plays back TTS responses

It's ideal for laptops, tablets, or any device with a browser and microphone.

## Operating Modes

The satellite supports three operational modes, configurable via environment variables or UI:

### 1. VAD Mode (`MODE_VAD`)

- **Trigger**: Local VAD (Voice Activity Detection) activation
- **Behavior**: Acts like a push-to-talk button that's always "pressed" when speech is detected
- **Use Case**: Fast, responsive mode for testing or environments where constant listening is acceptable
- **Source**: `hivemind-webspeech/src/index.js` (line 7)

### 2. WakeWord Mode (`MODE_WAKEWORD`)

The full "Voice Assistant" mode.

- **Trigger**: Local detection of "Hey Mycroft" (or other configured wake word) using ONNX-based Precise engine
- **Behavior**: Waits for wake word, then enters active listening state; uses VAD to capture the subsequent command
- **Accuracy**: Precise engine provides ~95% accuracy with minimal false positives
- **Use Case**: Always-on voice assistant; production deployments
- **Source**: `hivemind-webspeech/src/index.js` (line 8)

### 3. Streaming Mode (`MODE_STREAMING`)

Real-time audio streaming (equivalent to `hivemind-mic-satellite`).

- **Trigger**: Local VAD activation
- **Behavior**: Instead of waiting for end-of-speech to send a complete WAV file, streams raw PCM chunks to the hub in real-time
- **Use Case**: Continuous audio transmission for low-latency processing
- **Source**: `hivemind-webspeech/src/index.js` (line 9)

### State Machine

The client manages these modes through internal states:
- `idle` — Waiting for wake word or VAD activation
- `ww_listening` — Detecting wake word
- `active_listening` — Capturing user command after wake word
- `processing` — Awaiting hub response
- `speaking` — Playing back TTS

**Source**: `hivemind-webspeech/src/index.js` (lines 11-15)

## Audio Pipeline

**Source**: `hivemind-webspeech/docs/audio_pipeline.md`

The browser satellite processes audio in stages:

1. **Microphone Capture** — Web Audio API `getUserMedia()` captures raw PCM at 16kHz mono
2. **Silero VAD** — WASM-based voice detection filters silence and background noise
3. **Wake Word Detection** — ONNX Precise engine detects "Hey Mycroft" locally
4. **Audio Buffering** — Complete utterances are collected before transmission
5. **Streaming to Hub** — Raw PCM or compressed audio sent via HiveMind `BIN` messages
6. **STT Processing** — Hub transcribes using configured STT plugin
7. **Intent Matching** — Transcription routed through OVOS intent pipeline
8. **TTS Response** — Hub synthesizes spoken response via configured TTS plugin
9. **Playback** — Browser plays back `BIN` payload type 6 (TTS_AUDIO) via Web Audio API

## Wake Word Configuration

**Source**: `hivemind-webspeech/docs/wakeword.md`

Wake word detection uses the ONNX Precise engine, portable across browsers without server-side processing.

**Configuration** (in `config.js` or environment):

```javascript
{
  "wakeword_engine": "precise",
  "wakeword_model": "hey-mycroft.onnx",  // ONNX model path
  "wakeword_threshold": 0.5,              // 0.0-1.0; lower = more sensitive
  "wakeword_listen_time": 3000            // ms to listen after wake word
}
```

Models are loaded from browser local storage or CDN. Pre-trained models for common phrases:
- "hey mycroft" (default)
- "hey google"
- "hey siri"
- Custom models trainable with `ovos-wakeword-plugin-precise`

## TTS Configuration

**Source**: `hivemind-webspeech/docs/tts.md`

The hub's configured TTS plugin synthesizes speech responses. The browser receives audio as `BIN` payload type 6 and plays it back via Web Audio API.

**Supported TTS Plugins** (configured on hub):
- Google Cloud TTS
- Festival (local)
- ESpeak (local)
- Any OVOS TTS plugin

## Installation & Usage

### Browser Installation

No server needed. Simply open the HTML file in a modern browser:

```bash
# Clone or download
git clone https://github.com/JarbasHiveMind/hivemind-webspeech.git
cd hivemind-webspeech

# Install JavaScript dependencies
npm install

# Start development server
npm run dev

# Or build for production
npm run build
```

Then open `index.html` in your browser and configure:
- HiveMind hub address (host:port)
- Access key and password (from identity file)
- Operating mode (VAD/WakeWord/Streaming)

### Configuration

Edit `src/config.js` to set:

```javascript
export const CONFIG = {
  hivemind_host: "192.168.1.10",
  hivemind_port: 5678,
  mode: "MODE_WAKEWORD",           // VAD, WAKEWORD, STREAMING
  vad_engine: "silero",
  wakeword_threshold: 0.5,
  audio_sample_rate: 16000,
  chunk_size: 4096
};
```

## Browser Compatibility

| Browser | WebSpeech API | Web Audio API | Status |
|---------|---------------|---------------|--------|
| Chrome / Chromium | ✅ | ✅ | Fully supported |
| Firefox | ⚠️ (non-standard) | ✅ | Partial (Web Audio fallback) |
| Safari | ❌ | ✅ | Partial (Web Audio only) |
| Edge | ✅ | ✅ | Fully supported |

**Note**: For cross-browser compatibility, the application gracefully falls back to Web Audio API + Silero VAD when WebSpeech is unavailable.

## Security & Privacy

- **Local Processing**: Wake word and VAD run in the browser; no audio sent until wake word is detected
- **Encryption**: All audio sent to the hub is encrypted via HiveMind's AES-256-GCM (see [Encryption](/HiveMind-community-docs/docs/19_crypto.md))
- **No Cloud TTS**: If using local TTS plugins on the hub (Festival, ESpeak), no audio leaves your network
- **User Consent**: Requests microphone permission (browser native)

## Comparison with Other Satellites

| Satellite | Target | Processing | Installation |
|-----------|--------|-----------|--------------|
| **WebSpeech** | Browsers (web) | Browser + Hub | None (web-based) |
| **Voice Satellite** | Linux SBC (Pi) | Local + Hub | Python + system packages |
| **Voice Relay** | Linux SBC (Pi) | Local + Hub | Python + system packages |
| **Mic Satellite** | Minimal IoT | Hub only | Minimal Python |

## See Also

- [Satellites: Choosing a Satellite](/HiveMind-community-docs/docs/satellite_comparison.md)
- [Protocol: Binarization & Audio Streaming](/HiveMind-community-docs/docs/18_binarization.md#audio-streaming--satellites)
- [Client Libraries: HiveMessageBusClient](/HiveMind-community-docs/docs/11_devs.md)
