# WebSpeech Browser Satellite

No install, no hardware, no app store — just open a web page. The WebSpeech satellite is
a satellite that lives entirely in a browser tab: it grabs the microphone right there in
the page and streams what it hears to hivemind-core. Anyone you can hand a URL to can
talk to your hive from their phone, their laptop, a borrowed computer — whatever has a
browser. And it's no toy in the security department: the browser runs the same modern
Noise handshake as a native client, deriving its key from the password in-page, so a
password alone is all it takes.

!!! abstract "In a nutshell"
    - **On the device (the browser):** microphone capture and VAD (JavaScript — Silero via onnxruntime-web), with three independently configurable options: wake word, audio transport, and text-to-speech.
    - **On hivemind-core:** STT, TTS, skills, and intents (unless in-browser TTS is enabled).
    - By default it ships audio as base64 over the bus (`recognizer_loop:b64_audio`) — the server needs `ovos-dinkum-listener >= 0.0.3a19` and a one-time `hivemind-core allow-msg "recognizer_loop:b64_audio"`. Switching the audio transport to `binary` instead sends a WIRE-1 `STT_AUDIO_HANDLE` frame, admitted through hivemind-core's binary policy rather than the bus allow-list.
    - Encryption defaults to the Noise v3 handshake with full parity to hivemind-core, deriving the key from the password in-browser. A password is enough — nothing to provision.

    | Setting | Options | Default |
    |---|---|---|
    | Wake word | `off`, `precise-onnx-js` | `off` — every VAD-segmented utterance streams; `precise-onnx-js` loads a Precise `.onnx` model in-browser and gates capture until the wake word fires |
    | Audio transport | `base64`, `binary` | `base64` |
    | Text to speech | `server-text`, `phoonnx-js` | `server-text` — the reply is only rendered as text; `phoonnx-js` also synthesizes and plays it in-browser |

---

## When to use it

- Web applications or chat interfaces
- Temporary client access from any device with a browser
- Users already in a browser context

---

## Requirements

By default this browser client ships its captured audio as **base64 over the bus** (the
`recognizer_loop:b64_audio` message). hivemind-core must:

1. Run a listener new enough to decode that message — `ovos-dinkum-listener >= 0.0.3a19`.
2. Be told to accept the message:

   ```bash
   hivemind-core allow-msg "recognizer_loop:b64_audio"
   ```

Without both, the WebSocket connects but no audio ever reaches hivemind-core. The hub
does send an explicit `hive.policy.denied` error over the connection, but the page
doesn't currently surface that error, so it looks silent.

Switching the page's **Audio transport** setting to `binary` instead sends each utterance
as a WIRE-1 `STT_AUDIO_HANDLE` raw-PCM binary frame, smaller on the wire with no base64
inflation. That path needs `hivemind-audio-binary-protocol` on hivemind-core and is
admitted through its binary policy, not the `allow-msg` whitelist.

!!! warning "The TLS / mixed-content rule (read this first)"
    Browsers refuse to open an insecure WebSocket from a secure page. In practice:

    - A page served over **`https://`** may only open a **`wss://`** (TLS) WebSocket —
      a plain `ws://` target is blocked.
    - A plain **`ws://`** WebSocket is only allowed to **`127.0.0.1`** / `localhost`.

    So for **local** testing, serve the page on `http://localhost` and connect to a
    hivemind-core instance on `ws://127.0.0.1`. To reach a **remote** hivemind-core from
    an `https://` page (such as the hosted demo), its WebSocket must terminate TLS — use
    `wss://`. Microphone
    access (`getUserMedia`) also requires a secure context (`https://` or
    `http://localhost`).

??? note "Advanced: crypto and VAD details"
    - **Encryption** defaults to the **Protocol v3 Noise handshake** with full parity
      to hivemind-core: the browser negotiates the default
      `Noise_XXpsk2_25519_ChaChaPoly_SHA256` suite and derives the PSK as
      `argon2id(password, SHA-256(node_id))` **in-browser**, pairing the native
      **Web Crypto** API with the pure-JS `@noble/ciphers` + `@noble/hashes` bundle.
      The **Password** field in the form is the password from
      `hivemind-core add-client` — a password alone is enough, with no server-side
      configuration and no provisioned key. It falls back to the legacy V1 handshake
      (PBKDF2-HMAC-SHA256 + AES-GCM) against older hivemind-core versions, or when a minimal bundle
      ships without `@noble`.
    - **Wake word is off by default,** so capture is push-to-talk, gated by a **Start
      VAD** / **Stop VAD** toggle. Setting **Wake word** to `precise-onnx-js` loads a
      Precise `.onnx` model in-browser and gates capture on it instead — nothing streams
      until the wake word fires. Voice activity detection uses the **Silero VAD** model
      run in the browser via **onnxruntime-web** (`@ricky0123/vad-web`), loaded from a
      CDN — so the page needs network access on first load.

---

## Installation

No installation required on the client side. Include the [HiveMind.js](https://github.com/JarbasHiveMind/HiveMind-js) library in your HTML.

---

## Resources

- [hivemind-webspeech](https://github.com/JarbasHiveMind/hivemind-webspeech) — reference implementation
- [HiveMind-js](https://github.com/JarbasHiveMind/HiveMind-js) — JavaScript client library
- [hivemind-flask-chatroom](https://github.com/JarbasHiveMind/hivemind-flask-chatroom) — Flask template for a browser-based chatroom

---

## Limitations

- Browser VAD capabilities vary by browser and platform
- Requires JavaScript support
- Microphone access requires user permission (browser security model)
- Wake word off by default — capture is VAD-gated push-to-talk via a VAD toggle button, unless `precise-onnx-js` wake word is enabled in the page's settings

---

## See also

For a text-only browser experience, [hivemind-flask-chatroom](https://github.com/JarbasHiveMind/hivemind-flask-chatroom)
is a Flask web app (default port `8985`, no audio) that fans one server-side credential
out to many browser visitors as a shared chatroom.

---

## Next

New to HiveMind? Start with the [Quick Start](../quickstart.md), then prepare the server
side per the requirements above.

---

## Source

Validated against the HiveMind source:

- [`docs/configuration.md`](https://github.com/JarbasHiveMind/hivemind-webspeech/blob/HEAD/docs/configuration.md) — server requirements, the TLS/mixed-content rule, the credential form
- [`README.md`](https://github.com/JarbasHiveMind/hivemind-webspeech/blob/HEAD/README.md) — v3 Noise crypto (ChaChaPoly + in-browser argon2id PSK) with V1 (PBKDF2 + AES-GCM) fallback, Silero VAD via onnxruntime-web, `recognizer_loop:b64_audio` transport
- [`readme.md`](https://github.com/JarbasHiveMind/HiveMind-js/blob/HEAD/readme.md) — the JavaScript client library this page is built on
