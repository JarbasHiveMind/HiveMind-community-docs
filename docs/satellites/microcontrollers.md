# Microcontrollers (ESP32)

The thinnest possible *hardware* satellite. These clients turn a bare
microcontroller — an ESP32, or a Raspberry Pi Pico W — into a HiveMind
[satellite](../reference/glossary.md), without running OVOS or a full Python desktop
on the device. The chip captures the microphone and ships audio to the
[hub](../reference/glossary.md); the hub does all the speech-to-text, intents, skills,
and text-to-speech.

There are two clients, depending on the language you want to work in:

- **[ESP32 (C / ESP-IDF)](#esp32-c-esp-idf)** — maximum control, supports on-device
  wakeword on the Voice PE board.
- **[MicroPython](#micropython)** — pure Python, simpler to hack on.

!!! tip "Which one?"
    Choose **C / ESP-IDF** for the most control, the best performance, and the only
    on-device wakeword path (ESP32-S3 Voice PE). Choose **MicroPython** if you would
    rather write Python and value quick iteration over raw efficiency. Both speak the
    same HiveMind protocol to the same hub, including **protocol v3 (Noise)** with a
    [provisioned PSK](#provisioned-psk), falling back to the legacy handshake on older hubs.

---

## Provisioned PSK

On protocol **v3** the session is protected by a **Noise** handshake
(`Noise_XXpsk2_25519_ChaChaPoly_SHA256`), which mixes in a 32-byte pre-shared key
derived from your password as `argon2id(password, SHA-256(node_id))`. argon2id is far
too heavy to run on a microcontroller, so constrained clients never derive it
on-device — you **pre-compute** it on the hub and flash the result:

```bash
hivemind-core derive-psk --password "your-password" --node-id "<hub-node-id>"
```

The printed hex string is the same PSK a full peer would derive, so the device
interoperates with no server-side distinction — and the device never stores the
password itself. Provision it as `noise_psk_hex` (ESP32) or the `psk=` argument
(MicroPython). If the hub does not advertise v3 (or no PSK is set), the client falls
back to the legacy password handshake automatically. Optionally pin the hub's static
X25519 key to enable the lighter `Noise_KKpsk0` pattern.

---

## ESP32 (C / ESP-IDF)

[hivemind-esp32-client](https://github.com/JarbasHiveMind/hivemind-esp32-client) is an
ESP-IDF **component** (written in C) that makes an ESP32 a HiveMind satellite. It
implements the HiveMind handshake, AEAD encryption, bus messaging, and binary audio
transport — sized to run on the chip.

**Targets:** ESP32, ESP32-S3 (including the Home Assistant **Voice PE** board), and
ESP32-C3. Requires **ESP-IDF 5.0+** on the build host.

**What runs on the device:**

- On-device **microphone** (INMP441 I2S mic in the mic example).
- On the **ESP32-S3 Voice PE** build: on-device **VAD** and an optional **wakeword**
  (ESP-SR **WakeNet9**). On a plain ESP32 the mic streams audio and the hub does the
  rest.
- **STT and TTS are off-device** — the hub provides them.

??? note "Advanced: audio transport modes"
    The component can ship audio three ways, and the example lets you pick per axis:

    - **HiveMind binary** — streams raw PCM as `HM_BIN_RAW_AUDIO` (binary type `1`)
      inside an `HM_MSG_BINARY` frame. TTS comes back as `HM_BIN_TTS_AUDIO`. This is the
      same family of binary `RAW_AUDIO` traffic the mic satellite uses.
    - **HiveMind base64** — batches a WAV over the bus via `recognizer_loop:b64_transcribe`,
      with TTS via `speak:b64_audio`.
    - **OVOS HTTP** — POST/GET against a standalone OVOS STT/TTS HTTP server, bypassing
      the hub for speech.

    For the two HiveMind modes the hub needs
    [`hivemind-audio-binary-protocol`](../server/audio-binary-protocol.md). Encryption
    is AES-256-GCM (hardware-accelerated on the ESP32) or ChaCha20-Poly1305, with the
    session key derived from your password via PBKDF2-HMAC-SHA256.

### Build and flash

```bash
# from an example directory, e.g. examples/mic_satellite
idf.py set-target esp32s3        # or esp32, esp32c3
idf.py menuconfig                # set Wi-Fi SSID/password and the hub host/key/password
idf.py build flash monitor
```

Register the device on the hub first, as with any satellite:

```bash
hivemind-core add-client --name esp32 \
  --access-key "your-access-key" --password "your-password"
```

!!! warning "No TLS — encryption is in the handshake"
    The ESP32 client connects over **plain WebSocket (`ws://`)**; confidentiality and
    authentication come from the HiveMind handshake itself — the **Noise** session on
    v3 (with a [provisioned PSK](#provisioned-psk)), or application-layer AEAD
    (AES-GCM / ChaCha20-Poly1305) on the legacy path — **not** from TLS. There is no
    server-certificate verification. Keep these devices on a trusted network.

??? note "Advanced: legacy first connection is slow"
    On the **legacy** (v1/v2) handshake, PBKDF2 at 100k iterations is heavy for a
    microcontroller, so the **first** handshake takes roughly **10–30 s** (reconnects
    reuse the derived key). The **v3 Noise** path avoids this entirely by using the
    pre-computed [provisioned PSK](#provisioned-psk).

---

## MicroPython

[hivemind-micropython-client](https://github.com/JarbasHiveMind/hivemind-micropython-client)
is a pure-Python satellite from a single shared codebase that runs on **MicroPython
1.20+** (ESP32, Raspberry Pi Pico W, …) **and** on **CPython 3.10+** for desktop
development — the runtime is auto-detected at import. It is for makers who would rather
write Python than C.

The device keeps a tiny footprint by running no OVOS on-device: it performs the full
HiveMind handshake and encrypted messaging, then streams **PCM audio over the binary
channel** (via the `machine.I2S` peripheral in the mic example) to the hub, which does
the speech work. On protocol **v3** the session is a **Noise** handshake using a
[provisioned PSK](#provisioned-psk); on the legacy path, encryption is AES-256-GCM or
ChaCha20-Poly1305 with PBKDF2-HMAC-SHA256 key derivation — byte-for-byte compatible
with `hivemind-core`.

### Install

```bash
mpremote mip install github:JarbasHiveMind/hivemind-micropython-client
```

Or from the device REPL:

```python
import mip
mip.install("github:JarbasHiveMind/hivemind-micropython-client")
```

For desktop testing on CPython, clone the repo and optionally
`pip install websockets cryptography z85base91` for faster crypto and the extra
encodings (all optional; the code falls back to pure Python).

!!! warning "Legacy first handshake is slow on-device"
    On the **legacy** (v1/v2) handshake, pure-Python PBKDF2 at 100k iterations makes the
    **first** handshake take roughly **10–30 s** on an ESP32. For production, freeze the
    `_hivemind_crypto` C module into the firmware — the handshake then drops to a couple
    of seconds. Reconnects reuse the derived key. The **v3 Noise** path sidesteps this by
    using a [provisioned PSK](#provisioned-psk) instead of on-device key stretching.

---

## Source

Validated against the HiveMind source:

- [`README.md`](https://github.com/JarbasHiveMind/hivemind-esp32-client/blob/HEAD/README.md) — ESP-IDF component, targets, ciphers, `HM_BIN_RAW_AUDIO` transport, no-TLS V1 design
- [`docs/getting-started.md`](https://github.com/JarbasHiveMind/hivemind-esp32-client/blob/HEAD/docs/getting-started.md) — ESP-IDF 5.0+, INMP441 mic, build/flash steps
- [`README.md`](https://github.com/JarbasHiveMind/hivemind-micropython-client/blob/HEAD/README.md) — dual MicroPython/CPython runtime, `mip` install, binary transport, PBKDF2 first-handshake caveat
- [`docs/getting-started.md`](https://github.com/JarbasHiveMind/hivemind-micropython-client/blob/HEAD/docs/getting-started.md) — prerequisites and first satellite
