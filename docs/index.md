
# HiveMind

![HiveMind Logo](https://github.com/JarbasHiveMind/HiveMind-assets/raw/master/logo/hivemind-512.png)

!!! warning "Pre-release software"
    HiveMind is under active development. Expect bugs and breaking changes between releases.

Run one voice assistant — or one chatbot — on a computer you already own, and let
every other device in your home talk to it. The Raspberry Pi in the kitchen, a
five-dollar microcontroller on the nightstand, an old phone, a browser tab on your
laptop: each becomes a **satellite** of the same assistant. All the thinking happens
on your server. No account, no cloud, nothing leaving your network — unplug the
internet and it still answers.

That server is **hivemind-core**. The small devices are **satellites**. HiveMind is
the language they speak to each other.

!!! abstract "In a nutshell"
    - **One brain, many bodies.** The assistant runs once, on hivemind-core; cheap devices connect as satellites and borrow it.
    - **It's yours.** Self-hosted, encrypted end to end, and fully functional offline — the network never has to leave the building.
    - **Anything can be a satellite.** A browser, a microcontroller, a phone, a Pi, a terminal — one protocol, over whatever transport you have.

## Picture it

You set up hivemind-core once, on a mini PC in a closet. Then you scatter satellites
around the house:

```
   "what's the weather?"                          "set a 10 minute timer"
        kitchen Pi  ───────┐                 ┌─────── bedroom microcontroller
                           │                 │
      laptop browser ──────┤   hivemind-core ├────── your phone
                           │  (the assistant │
        garage CLI  ───────┘   lives here)   └─────── a friend's house, over the internet
```

Every satellite reaches the *same* assistant — same skills, same memory, same voice.
Add a device by handing it a key; it joins in seconds. The microcontroller doesn't
run speech recognition or an LLM — it can't. It just captures audio and lets
hivemind-core do the heavy lifting, then plays back the reply. That's why a satellite
can cost five dollars.

And because hivemind-core runs on hardware you control, the conversation stays with
you. There is no service to sign up for and no server on the other side of the world
holding your recordings. Take the whole thing to a cabin with no internet — the
satellites still talk to hivemind-core over the local network, and the assistant
still works.

## What can it run?

hivemind-core is the mesh; the *intelligence* is a back end you choose and plug in:

- an [OpenVoiceOS](https://openvoiceos.github.io/community-docs) skills server — a full,
  private voice assistant (weather, timers, music, home control, hundreds of skills);
- a **persona** — an LLM with a personality, for open-ended conversation;
- or your own back end, through the agent-protocol plugins.

Swap the brain without touching a single satellite. The devices never know the
difference.

## Where to start

=== "I just want to use it"

    You don't need to write any code.

    | I want to… | Go to |
    |---|---|
    | Understand HiveMind in plain terms | [About](about.md) |
    | Set up hivemind-core and connect my first satellite | [Quick Start](quickstart.md) |
    | Run it all from a friendly web UI | [Admin Panel](server/admin-panel.md) |
    | Pick the right satellite for my hardware | [Choosing a Satellite](satellites/index.md) |
    | Run it in Docker | [Docker Deployment](server/docker.md) |
    | Just chat with an LLM | [Persona Server](server/persona-server.md) |
    | Connect Home Assistant | [Home Assistant](integrations/home-assistant.md) |
    | Get unstuck | [FAQ](faq.md) · [Glossary](reference/glossary.md) |

=== "I want to build on it"

    Connect from code, port the protocol elsewhere, or extend hivemind-core.

    | I want to… | Go to |
    |---|---|
    | Connect from Python | [Client Library](developers/client-library.md) |
    | Implement HiveMind in another language | [Protocol Specification](developers/protocol-spec.md) |
    | Add a transport, back end, database, or policy | [Writing Plugins](developers/writing-plugins.md) |
    | Test a satellite, server, or whole mesh | [Testing](developers/testing.md) |
    | Look up a flag, config key, or message type | [Reference](reference/index.md) |

    Under the hood: every satellite authenticates with its own key and password,
    the session is encrypted with a modern Noise handshake (no TLS required, though
    you can add it), and hivemind-core instances chain into a mesh — messages
    `ESCALATE` up to a bigger brain or `BROADCAST` down to a room full of devices.
    Start with the [Protocol Specification](developers/protocol-spec.md).

!!! tip "Totally new to this?"
    Read [About](about.md) for the plain-language tour, keep the
    [Glossary](reference/glossary.md) open in a tab, then do the
    [Quick Start](quickstart.md). Running HiveMind does not require being a developer.

## Community

- [Matrix chat](https://matrix.to/#/#jarbashivemind:matrix.org) — news, support, and discussion
- [GitHub](https://github.com/JarbasHiveMind) — source and issues
- [YouTube](https://www.youtube.com/channel/UCYoV5kxp2zrH6pnoqVZpKSA/) — guides and demos

**Next:** [Quick Start](quickstart.md) — set up hivemind-core and connect your first satellite.
