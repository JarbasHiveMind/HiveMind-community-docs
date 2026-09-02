# SIP / baresip Bridge

Turn your hive into something you can call on the phone. The
[HiveMind-baresip-bridge](https://github.com/JarbasHiveMind/HiveMind-baresip-bridge)
answers ordinary SIP calls with [baresip](https://github.com/baresip/baresip), a SIP
softphone, and turns the call into a voice conversation with your hive. There's no
wakeword — an answered call is the activation signal — and the caller's replies are
spoken back into the call.

!!! abstract "In a nutshell"
    - Runs the `hivemind-baresip-bridge` console command, wrapping a local `baresip` process.
    - No wakeword: an answered call *is* the activation. Local VAD segments the caller's speech.
    - The bridge asks `hivemind-core` to synthesize replies (`speak:b64_audio`) and plays the audio into the call — it does not synthesize speech itself.

!!! tip "Beginner's mental model"
    A phone call to your hive works like a voice satellite, except the microphone is a
    phone call instead of a room. Whoever calls in gets to talk to your assistant.

---

## Getting a SIP account

You need something that can route a call to baresip:

- **Self-host [Asterisk](https://www.asterisk.org/)**, a SIP server, and register two
  extensions on it — one for the bridge, one for whatever device calls in. Asterisk
  only needs to be reachable by baresip and the calling device, so a LAN or a Tailscale
  network is enough; it doesn't need to be public. The extension number and password
  become the bridge's `sip_user` / `sip_password`, and the Asterisk host becomes
  `sip_gateway`.
- **Use a SIP trunk provider.** Any VoIP reseller selling SIP trunking gives you a
  username, password, and gateway hostname — the same three values, at the cost of a
  subscription, but with a real phone number the public can dial.

---

## Install

!!! warning "Not on PyPI yet"
    `pip install HiveMind-baresip-bridge` fails
    ([issue #5](https://github.com/JarbasHiveMind/HiveMind-baresip-bridge/issues/5)).
    Install from source, or build the repo's Dockerfile, which also installs a working
    `baresip` binary.

```bash
git clone https://github.com/JarbasHiveMind/HiveMind-baresip-bridge
cd HiveMind-baresip-bridge
pip install .
```

A working `baresip` binary must be on `PATH` (`apt install baresip` on
Debian/Ubuntu), or use `docker build -t hivemind-baresip-bridge .` instead.

---

## Configuration

SIP settings live in a JSON file, `~/.hivemind_baresip_bridge.json` by default:

```json
{
  "sip_user": "1000",
  "sip_password": "secret",
  "sip_gateway": "sip.example.com",
  "sip_transport": "udp",
  "auto_answer": true,
  "allowlist": []
}
```

`allowlist` restricts which caller numbers get answered; leave it empty to accept any
caller. HiveMind hub credentials (access key, password, host) are set separately, the
same way as any other HiveMind client.

---

## Permissions

```bash
hivemind-core add-client --name baresip-bridge
hivemind-core allow-msg "recognizer_loop:utterance" <id>
hivemind-core allow-msg "speak:b64_audio" <id>
hivemind-core allow-msg "baresip.dtmf" <id>
hivemind-core allow-msg "recognizer_loop:speech.recognition.unknown" <id>
```

`<id>` is the Node ID printed by `add-client` above. `baresip.dtmf` lets the bridge report
phone keypad tones (DTMF) pressed during the call, for IVR-style menus.
`recognizer_loop:speech.recognition.unknown` lets the bridge report STT failures
(unintelligible or silent audio) back to the hub; without it, a caller whose speech can't
be transcribed just gets silence instead of a recognition-failure prompt.

Skipping `speak:b64_audio` is the classic failure mode: calls connect and get
transcribed fine, but the bridge never speaks back. The hub does send an explicit
`hive.policy.denied` error over the connection, but the bridge doesn't currently surface
that error to the call, so it looks silent.

---

## Run

```bash
hivemind-baresip-bridge --host ws://core.example.com --key <access-key> --password <password>
```

Call the SIP extension you configured. You should hear the assistant's reply spoken
back into the call.

---

## Source

Validated against the HiveMind source.

- [`README.md`](https://github.com/JarbasHiveMind/HiveMind-baresip-bridge/blob/HEAD/README.md) — architecture, install, configuration
- [`docs/setup.md`](https://github.com/JarbasHiveMind/HiveMind-baresip-bridge/blob/HEAD/docs/setup.md) — full Asterisk walkthrough and troubleshooting
