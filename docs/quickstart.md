# Quick Start

By the end of this page you'll say "hey mycroft, what time is it?" to one device and
hear the answer come back — proof that the whole path works. It takes about ten minutes:
stand up hivemind-core on a machine you own, hand a satellite a key, and watch it join.
Everything after that is just adding more devices.

## What you're building

One server does the thinking; one satellite does the listening and speaking. They talk
over your LAN:

```
            ON THE SERVER                              ON THE SATELLITE
 ┌─────────────────────────────┐           ┌──────────────────────────┐
 │        hivemind-core        │  LAN      │     Voice Satellite      │
 │  (hivemind-core listen)     │◄─ ws ────►│  (hivemind-voice-sat)    │
 │                             │  :5678    │                          │
 │  • OVOS skills              │           │  • microphone            │
 │  • STT (speech → text)      │           │  • wakeword              │
 │  • TTS (text → speech)      │           │  • VAD                   │
 │  • intents / reasoning      │           │  • local STT + TTS       │
 └─────────────────────────────┘           └──────────────────────────┘
```

The voice satellite captures audio and turns it into text, then sends that text to
hivemind-core; hivemind-core runs the skill and sends a spoken reply back. (Other
satellite types push more of that work onto the server — see
[Choosing a Satellite](satellites/index.md).)

!!! tip "Just want the simplest possible first run — an LLM to chat with, no OVOS?"
    This guide builds a full **voice assistant** on OpenVoiceOS. If you'd rather stand up
    a plain **LLM/chatbot** with nothing extra to install, follow the
    [Persona Server](server/persona-server.md) guide instead. The HiveMind half — pairing
    a client, connecting a satellite — is identical; you just skip the OVOS install and
    point hivemind-core at a persona backend. Pick one path and follow it start to finish.

!!! abstract "In a nutshell"
    - Install `ovos-core` + `ovos-messagebus` (the assistant), then `hivemind-core` (the gateway), then register a client and connect a satellite with its access key and password.
    - `hivemind-core listen` runs with sensible defaults — no config file needed for a first run.
    - Success is a full round trip: wakeword → speech → the server's skill → a spoken answer on the satellite.

**Scope:** this guide sets up **one hivemind-core server + one voice satellite on the same LAN**. To try it all on a single machine, use `127.0.0.1` everywhere. Find your server's LAN IP with `hostname -I`. Each step below is labelled **(ON THE SERVER)** or **(ON THE SATELLITE)** so you always know where to run it.

New to the terms here? Keep the [Glossary](reference/glossary.md) open in another tab.

---

## Step 1 — Install and start OVOS (ON THE SERVER)

hivemind-core is a gateway to an assistant, not the assistant itself, so the assistant
goes up first. The default backend is OpenVoiceOS, which runs as two pieces — a message
bus and the skills core — that must be running before HiveMind can reach them:

```bash
pip install ovos-core ovos-messagebus

# start the OVOS messagebus (keep it running, e.g. in its own terminal)
ovos-messagebus

# start ovos-core (in another terminal)
ovos-core
```

See the [OVOS documentation](https://openvoiceos.github.io/community-docs) for a full OVOS setup.

---

## Step 2 — Install hivemind-core (ON THE SERVER)

With the assistant running, add the gateway in front of it:

```bash
pip install hivemind-core
```

---

## Step 3 — Start the server (ON THE SERVER)

```bash
hivemind-core listen
```

That's the whole command — **no configuration required for a first run.** `listen` takes
no flags; it comes up with sensible defaults and accepts satellite connections on port
5678 (WebSocket) and 5679 (HTTP). When you later want to change ports, turn on TLS, or
switch the database backend, those live in `~/.config/hivemind-core/server.json` — see
[Server Configuration](reference/config.md), or run `hivemind-core print-config` to see
what's currently in effect. For now, leave it on defaults and move on.

---

## Step 4 — Register credentials for a satellite (ON THE SERVER)

A satellite needs its own identity before it can connect. Create one on the server:

```bash
hivemind-core add-client --name "my-satellite"
```

Output:

```
Credentials added to database!

Node ID: 2
Friendly Name: my-satellite
Access Key: 5a9e580a2773a262cbb23fe9759881ff
Password: 9b247ca66c7cd2b6388ad49ca504279d
Encryption Key: 4185240103de0770
WARNING: Encryption Key is deprecated, only use if your client does not support password
```

Note the **Access Key** and **Password** — you'll hand them to the satellite next. In the
steps below, replace `<ACCESS_KEY>` and `<PASSWORD>` with these values, and `<SERVER_IP>`
with the server's LAN IP from `hostname -I` (or `127.0.0.1` on a single machine).

> Want to choose your own key and password instead of generated ones? `hivemind-core add-client --help` lists the options: `--name`, `--access-key`, `--password`, `--crypto-key`, `--admin`, `--metadata`.

---

## Step 5 — Install and configure the satellite (ON THE SATELLITE)

Now move to the device that will do the listening. Pick a satellite package — for a full
voice satellite:

```bash
# Linux audio dependencies
sudo apt-get install -y libpulse-dev libasound2-dev

pip install HiveMind-voice-sat
```

Not sure which satellite fits your hardware? See [Choosing a Satellite](satellites/index.md).

Then hand it the credentials from Step 4 by writing its identity file:

```bash
hivemind-client set-identity \
  --key <ACCESS_KEY> \
  --password <PASSWORD> \
  --host <SERVER_IP> \
  --port 5678 \
  --siteid living-room
```

This writes `~/.config/hivemind/_identity.json`. Every satellite command reads it by default, so you only do this once.

---

## Step 6 — Test the connection (ON THE SATELLITE)

Before starting the microphone, confirm the satellite can actually reach the server:

```bash
hivemind-client test-identity
```

Expected output:

```
== Identity successfully connected to HiveMind!
```

If that fails, fix it here — it's the cheapest place to catch a wrong IP, key, or a server that isn't listening.

---

## Step 7 — Start the satellite (ON THE SATELLITE)

```bash
hivemind-voice-sat
```

Or pass credentials directly instead of relying on the identity file:

```bash
hivemind-voice-sat --host <SERVER_IP> --key <ACCESS_KEY> --password <PASSWORD>
```

`hivemind-voice-sat` also accepts `--port`, `--siteid`, and `--selfsigned` (accept self-signed certificates).

---

## Step 8 — Talk to it (ON THE SATELLITE)

Everything is running: hivemind-core on the server, the satellite listening. Speak to the
satellite:

> **"hey mycroft, what time is it?"**

**Success looks like:** the satellite plays a spoken reply — the current time. That round
trip — wakeword → your speech → the server's skill → a spoken answer — means the whole
path works. From here, adding a second device is just Step 4 (register) plus Steps 5–7
(connect) again.

**No reply?**

- Check the satellite's terminal logs for connection or audio errors.
- Re-run `hivemind-client test-identity` on the satellite to confirm it still reaches the server.
- `hivemind-voice-sat`'s **default STT and TTS are remote services** at `*.openvoiceos.org`, so the satellite needs internet access on first run. To go fully local, install local STT/TTS plugins on the satellite — see [Voice Satellite](satellites/voice-sat.md).
- Confirm `ovos-core` and `ovos-messagebus` are still running on the server (Step 1).

---

## Managing clients

That first satellite won't be your last. Everything you do to manage the ones that follow
happens with the same `hivemind-core` command you've already been using — listing who's
registered, removing a device, or loosening what a client may do. A taste of the common
ones:

```bash
# List all registered clients
hivemind-core list-clients

# Remove a client (node id is a positional argument)
hivemind-core delete-client 2

# Allow a specific OVOS message type
hivemind-core allow-msg "speak" 2

# Blacklist a skill
hivemind-core blacklist-skill "skill-homeassistant.openvoiceos" 2
```

For the full CLI reference see [CLI Reference](reference/cli.md) and [Permissions](concepts/security.md#permissions).

---

## Next steps

- [Choosing a Satellite](satellites/index.md) — compare all satellite options
- [Permissions](concepts/security.md#permissions) — restrict what each client can do
- [Auto Discovery](concepts/discovery.md) — find hivemind-core automatically over the local network
- [Docker Deployment](server/docker.md) — run hivemind-core in a container
- [Nested Hives](concepts/nested-hives.md) — connect hivemind-core instances to other instances

---

**Next:** [Choosing a Satellite](satellites/index.md) to pick the right device for your hardware, or [Core Concepts](concepts/protocol.md) to understand how the protocol works.
