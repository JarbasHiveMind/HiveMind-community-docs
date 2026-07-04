# Quick Start

This guide takes you from zero to a working hub with one connected satellite in under ten minutes.

**Scope:** this guide sets up **one hub + one voice satellite on the same LAN**. To try it all on a single machine, use `--host 127.0.0.1` everywhere. Find your hub's LAN IP with `hostname -I`. Each step below is labelled **(ON THE HUB)** or **(ON THE SATELLITE)** so you know where to run it.

New to the terms here? Keep the [Glossary](reference/glossary.md) open in another tab.

---

## Step 0 — Prerequisites (ON THE HUB)

The OVOS skills hub runs your skills on top of OpenVoiceOS, so `ovos-core` and `ovos-messagebus` must be installed and running on the hub machine before HiveMind can talk to them:

```bash
pip install ovos-core ovos-messagebus

# start the OVOS messagebus (keep it running, e.g. in its own terminal)
ovos-messagebus

# start ovos-core (in another terminal)
ovos-core
```

See the [OVOS documentation](https://openvoiceos.github.io/community-docs) for a full OVOS setup.

> **Want the simplest possible first run with no OVOS?** Use the [Persona hub](server/persona-hub.md) instead — it serves an LLM/chatbot via `hivemind-persona-agent-plugin` + `ovos-persona` and needs neither `ovos-core` nor `ovos-messagebus`. The HiveMind steps below are otherwise identical.

---

## Step 1 — Install HiveMind Core on the hub (ON THE HUB)

```bash
pip install hivemind-core
```

## Step 2 — Start the server (ON THE HUB)

```bash
hivemind-core listen
```

The server accepts satellite connections on port 5678 (WebSocket) and port 5679 (HTTP). `listen` takes no flags — all server settings (host, ports, SSL, database backend) are read from `~/.config/hivemind-core/server.json`. See [Server Configuration](reference/config.md), or run `hivemind-core print-config` to inspect the active configuration.

### What you are building

```
            ON THE HUB                              ON THE SATELLITE
 ┌─────────────────────────────┐           ┌──────────────────────────┐
 │        HiveMind Hub         │  LAN      │     Voice Satellite      │
 │  (hivemind-core listen)     │◄─ ws ────►│  (hivemind-voice-sat)    │
 │                             │  :5678    │                          │
 │  • OVOS skills              │           │  • microphone            │
 │  • STT (speech → text)      │           │  • wakeword              │
 │  • TTS (text → speech)      │           │  • VAD                   │
 │  • intents / reasoning      │           │  • local STT + TTS       │
 └─────────────────────────────┘           └──────────────────────────┘
```

The voice satellite captures audio and does its own speech-to-text, then sends the text utterance to the hub; the hub runs the skill and sends a spoken reply back. (Other satellite types push more of that work onto the hub — see [Choosing a Satellite](satellites/index.md).)

## Step 3 — Register credentials for a satellite (ON THE HUB)

On the hub machine, add a client entry:

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

Note the **Access Key** and **Password** — you will need them on the satellite. In the steps below, replace `<ACCESS_KEY>` and `<PASSWORD>` with the values printed by `add-client`, and `<HUB_IP>` with the hub's LAN IP from `hostname -I` (or `127.0.0.1` on a single machine).

> Use `hivemind-core add-client --help` to supply your own key and password instead of generated ones. Other options: `--name`, `--access-key`, `--password`, `--crypto-key`, `--admin`, `--metadata`.

## Step 4 — Install and configure the satellite (ON THE SATELLITE)

On the satellite device, choose a satellite package. For a full voice satellite:

```bash
# Linux audio dependencies
sudo apt-get install -y libpulse-dev libasound2-dev

pip install HiveMind-voice-sat
```

Not sure which satellite fits your hardware? See [Choosing a Satellite](satellites/index.md).

Write the identity file:

```bash
hivemind-client set-identity \
  --key <ACCESS_KEY> \
  --password <PASSWORD> \
  --host <HUB_IP> \
  --port 5678 \
  --siteid living-room
```

This writes `~/.config/hivemind/_identity.json`. All satellite commands read this file by default.

## Step 5 — Test the connection (ON THE SATELLITE)

```bash
hivemind-client test-identity
```

Expected output:

```
== Identity successfully connected to HiveMind!
```

## Step 6 — Start the satellite (ON THE SATELLITE)

```bash
hivemind-voice-sat
```

Or pass credentials directly instead of relying on the identity file:

```bash
hivemind-voice-sat --host <HUB_IP> --key <ACCESS_KEY> --password <PASSWORD>
```

`hivemind-voice-sat` also accepts `--port`, `--siteid`, and `--selfsigned` (accept self-signed certificates).

## Step 7 — Talk to it (ON THE SATELLITE)

With the hub running (Steps 0–2) and the satellite running (Step 6), speak to the satellite:

> **"hey mycroft, what time is it?"**

**Success looks like:** the satellite plays a spoken reply (the current time). That round trip — wakeword → your speech → the hub's skill → a spoken answer — means the whole path works.

**No reply?**

- Check the satellite's terminal logs for connection or audio errors.
- Re-run `hivemind-client test-identity` on the satellite to confirm it still reaches the hub.
- Remember that `hivemind-voice-sat`'s **default STT and TTS are remote services** hosted at `*.openvoiceos.org`, so the satellite needs internet access on first run. To go fully local, install and configure local STT/TTS plugins on the satellite — see [Voice Satellite](satellites/voice-sat.md).
- Confirm `ovos-core` and `ovos-messagebus` are still running on the hub (Step 0).

---

## Managing clients

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
- [Auto Discovery](concepts/discovery.md) — find the hub automatically over the local network
- [Docker Deployment](server/docker.md) — run the hub in a container
- [Nested Hives](concepts/mesh.md#nested-hives) — connect hubs to other hubs

---

**Next:** [Choosing a Satellite](satellites/index.md) to pick the right device for your hardware, or [Core Concepts](concepts/protocol.md) to understand how the protocol works.
