# Core Concepts

You can run HiveMind without ever reading this section — the [Quick Start](../quickstart.md)
gets you talking to a satellite in ten minutes flat. But at some point you'll wonder
*why* a new device can't do anything until you allow it, or *how* a question finds its
way to the right server, or *what* actually crosses the wire. This is where those
answers live: the ideas behind the mesh, not the commands to run it. Skim it once and
every other page clicks into place.

!!! tip "New here? Read these two first"
    Start with **[Mesh Topology](mesh.md)** (what the pieces are) and
    **[Security & Permissions](security.md)** (how devices trust each other). The rest
    you can read on demand.

## What's in this section

| Page | What it answers | For |
|---|---|---|
| [Mesh Topology](mesh.md) | What are hivemind-core instances, satellites, and relays? How do messages travel up and down the mesh? | Everyone |
| [Protocol & Message Types](protocol.md) | What does HiveMind actually send over the wire, and in which direction? | Curious users · developers |
| [Security & Permissions](security.md) | How are connections encrypted, and how do you control what each device may do? | Everyone running hivemind-core |
| [Database Backends](databases.md) | Where are client credentials stored? SQLite, JSON, or Redis? | Operators |
| [Auto Discovery](discovery.md) | How do satellites find hivemind-core automatically on a local network? | Everyone |
| [Plugin Architecture](plugins.md) | The five plugin families that make every layer swappable. | Developers · advanced operators |

---

## The one-paragraph version

**hivemind-core** is the brain: it runs skills, an LLM, or another AI backend. **Satellites**
are the lightweight devices you talk to — a microphone, a browser tab, a terminal.
They connect to hivemind-core over an **encrypted, permissioned protocol** that is agnostic
to both the transport (WebSocket, HTTP, MQTT…) and the payload (OVOS messages by
default). hivemind-core instances can connect to other instances, forming a **mesh**: requests one instance can't
answer **escalate** upward, and announcements **broadcast** downward. Every other page
in this manual is a detail of that sentence.

---

**Next:** [Mesh Topology](mesh.md)
