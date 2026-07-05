# What is HiveMind?

Voice assistants like Alexa or Google Home are two things wearing one shell: the
*ears and mouth* (a microphone and a speaker) and the *brain* (the part that
understands you and answers). In those products the brain lives in someone else's
data center, and every word you say takes a trip to the cloud.

HiveMind pulls those two apart. You run the brain once, on a computer in your own
home, and the ears and mouth become small, cheap, interchangeable devices —
**satellites** — that connect to it over your network. The brain is
**hivemind-core**. Everything else plugs into it.

!!! abstract "In a nutshell"
    - Satellites handle *hearing and speaking*; hivemind-core handles *understanding and answering* — and serves many satellites at once, each with its own identity and permissions.
    - Give hivemind-core an OVOS skills back end for a full voice assistant, or a persona back end to just chat with an LLM.
    - Satellites range from a one-line CLI to a full offline voice stack — the thinner the device, the more hivemind-core does for it.
    - It's a private gateway around your own assistant, running entirely on hardware you own.

## Why split them apart?

Because the brain is expensive and the ears are cheap.

Speech recognition, intent parsing, and an LLM need real memory and CPU. A doorway
sensor, a desk button, or a ten-dollar board has none of that — but it has a
microphone, and that's all a satellite needs. Run the brain *once*, on a machine
that can handle it, and every underpowered gadget in the house can offer the full
assistant just by forwarding audio to hivemind-core and playing back the reply.

Three things fall out of that split, for free:

- **Privacy.** The brain is in your closet, not a cloud. Your voice never leaves the
  building unless *you* send it somewhere.
- **Reach.** Anything that can open a network connection can be a satellite — a
  browser tab, a microcontroller, a phone, a Raspberry Pi, a chat room.
- **Resilience.** No internet? No problem. Satellites talk to hivemind-core over the
  local network, so the assistant keeps working in a blackout or a cabin off the grid.

```
[Mic Satellite]  ──┐
[Voice Relay]    ──┤──→  hivemind-core  ──→  skills · LLM · intents
[Browser Tab]    ──┤        (the brain)
[CLI Client]     ──┘        └──→  the reply goes back to whichever satellite asked
```

## What's the brain, exactly?

hivemind-core is the meeting point — it authenticates satellites, routes messages,
and enforces permissions. It is *not* the intelligence itself. You choose that and
plug it in:

| Back end | You get | Good for |
|---|---|---|
| **OVOS skills server** | a full private voice assistant — weather, timers, music, home control, hundreds of skills | replacing a smart speaker |
| **Persona (LLM)** | an LLM with a personality, no skills, simplest setup | open-ended conversation |
| **Your own** | anything, via the agent-protocol plugins | custom back ends |

Swap the back end and the satellites never notice — they only ever talk to
hivemind-core. HiveMind is a *gateway* around an assistant like OVOS, not a
replacement for it.

## What's a satellite, exactly?

Satellites form a spectrum, and the only question that places a device on it is
**where the audio is processed**. At the thin end, the device forwards raw audio and
hivemind-core does everything. At the thick end, the device runs wake word, speech
recognition, and speech synthesis itself, and sends hivemind-core only the final text.

| Satellite | Runs on the device | Runs on hivemind-core |
|---|---|---|
| [CLI client](satellites/cli.md) | nothing (you type) | everything |
| [WebSpeech](satellites/webspeech.md) (browser) | mic capture, VAD | speech · intents · skills |
| [Mic satellite](satellites/mic-satellite.md) | mic, VAD | speech · intents · skills |
| [Voice relay](satellites/voice-relay.md) | mic, VAD, wake word | speech · intents · skills |
| [Voice satellite](satellites/voice-sat.md) | the whole voice stack | intents · skills |

A thin satellite is cheaper and simpler but leans on the network; a thick one keeps
even the audio private and rides out a flaky connection. Most people start thin and
thicken the devices that need it. See [Choosing a Satellite](satellites/index.md).

!!! tip "Keep this open"
    The [Glossary](reference/glossary.md) defines every term used across the manual on
    a single page — handy on your first read.

**Next:** [Quick Start](quickstart.md) — stand up hivemind-core and connect your first satellite.
