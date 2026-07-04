# WebSpeech Browser Satellite

A JavaScript-based satellite that runs in any modern web browser. Captures microphone audio in the browser and streams it to the hub for processing.

**What runs locally**: Browser microphone + VAD (JavaScript)

**What the hub provides**: STT, TTS, skills, intents

## When to use it

- Web applications or chat interfaces
- Temporary client access from any device with a browser
- Users already in a browser context

## Requirements

This browser client ships its captured audio as **base64 over the bus** (the
`recognizer_loop:b64_audio` message), not as the binary `RAW_AUDIO` protocol. So the
hub does **not** need `hivemind-audio-binary-protocol`. Instead the hub must:

1. Run a listener new enough to decode that message — `ovos-dinkum-listener >= 0.0.3a19`.
2. Be told to accept the message:

   ```bash
   hivemind-core allow-msg "recognizer_loop:b64_audio"
   ```

Without both, the WebSocket connects but the audio is silently dropped and you get no
reply.

!!! warning "The TLS / mixed-content rule (read this first)"
    Browsers refuse to open an insecure WebSocket from a secure page. In practice:

    - A page served over **`https://`** may only open a **`wss://`** (TLS) WebSocket —
      a plain `ws://` target is blocked.
    - A plain **`ws://`** WebSocket is only allowed to **`127.0.0.1`** / `localhost`.

    So for **local** testing, serve the page on `http://localhost` and connect to a hub
    on `ws://127.0.0.1`. To reach a **remote** hub from an `https://` page (such as the
    hosted demo), the hub's WebSocket must terminate TLS — use `wss://`. Microphone
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
      (PBKDF2-HMAC-SHA256 + AES-GCM) against older hubs, or when a minimal bundle
      ships without `@noble`.
    - **No wakeword.** Capture is push-to-talk, gated by a **Start VAD** / **Stop VAD**
      toggle. Voice activity detection uses the **Silero VAD** model run in the browser
      via **onnxruntime-web** (`@ricky0123/vad-web`), loaded from a CDN — so the page
      needs network access on first load.

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

## See also

For a text-only browser experience, [hivemind-flask-chatroom](https://github.com/JarbasHiveMind/hivemind-flask-chatroom)
is a Flask web app (default port `8985`, no audio) that fans one server-side credential
out to many browser visitors as a shared chatroom.

## Next

New to HiveMind? Start with the [Quick Start](../quickstart.md), then prepare the hub
side per the requirements above.

## Source

Validated against the HiveMind source:

- [`docs/configuration.md`](https://github.com/JarbasHiveMind/hivemind-webspeech/blob/HEAD/docs/configuration.md) — hub requirements, the TLS/mixed-content rule, the credential form
- [`README.md`](https://github.com/JarbasHiveMind/hivemind-webspeech/blob/HEAD/README.md) — v3 Noise crypto (ChaChaPoly + in-browser argon2id PSK) with V1 (PBKDF2 + AES-GCM) fallback, Silero VAD via onnxruntime-web, `recognizer_loop:b64_audio` transport
- [`readme.md`](https://github.com/JarbasHiveMind/HiveMind-js/blob/HEAD/readme.md) — the JavaScript client library this page is built on
