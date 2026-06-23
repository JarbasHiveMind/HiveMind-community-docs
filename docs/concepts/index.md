# Core Concepts

This section explains **how HiveMind works** — the ideas behind the mesh, not the
commands to run it. You don't need to read it all before setting things up
([Quick Start](../quickstart.md) gets you running in ten minutes), but a skim here
makes every other page click into place.

!!! tip "New here? Read these two first"
    Start with **[Mesh & Hub Topology](mesh.md)** (what the pieces are) and
    **[Security & Permissions](security.md)** (how devices trust each other). The rest
    you can read on demand.

## What's in this section

| Page | What it answers | For |
|---|---|---|
| [Mesh & Hub Topology](mesh.md) | What are hubs, satellites, and relays? How do messages travel up and down the mesh? | Everyone |
| [Protocol & Message Types](protocol.md) | What does HiveMind actually send over the wire, and in which direction? | Curious users · developers |
| [Security & Permissions](security.md) | How are connections encrypted, and how do you control what each device may do? | Everyone running a hub |
| [Database Backends](databases.md) | Where are client credentials stored? SQLite, JSON, or Redis? | Operators |
| [Auto Discovery](discovery.md) | How do satellites find the hub automatically on a local network? | Everyone |
| [Plugin Architecture](plugins.md) | The five plugin families that make every layer swappable. | Developers · advanced operators |

## The one-paragraph version

A **hub** is the brain: it runs skills, an LLM, or another AI backend. **Satellites**
are the lightweight devices you talk to — a microphone, a browser tab, a terminal.
They connect to the hub over an **encrypted, permissioned protocol** that is agnostic
to both the transport (WebSocket, HTTP, MQTT…) and the payload (OVOS messages by
default). Hubs can connect to other hubs, forming a **mesh**: requests a hub can't
answer **escalate** upward, and announcements **broadcast** downward. Every other page
in this manual is a detail of that sentence.

---

**Next:** [Mesh & Hub Topology](mesh.md)
