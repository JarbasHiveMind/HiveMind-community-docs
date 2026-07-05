# Developer Guide

**The Developer Guide is for writing code against HiveMind** — building a custom
satellite, embedding a client in an app, implementing a new transport or backend, or
testing a protocol implementation.

!!! tip "Not a developer? You can skip this whole section."
    Running hivemind-core and connecting off-the-shelf satellites needs no code — see
    [Quick Start](../quickstart.md) and [Satellites](../satellites/index.md). Come back
    here when you want to build something new.

---

## What's in this section

| Page | Use it when you want to… |
|---|---|
| [Client Library](client-library.md) | Connect to a hive from Python — send/receive messages, handle encryption, reconnect. |
| [Protocol Specification](protocol-spec.md) | Implement HiveMind in another language, or understand the exact wire format byte-for-byte. |
| [Writing Plugins](writing-plugins.md) | Add a new transport, agent backend, database, or permission policy. |
| [Testing](testing.md) | Write end-to-end tests for satellites, hivemind-core, or whole mesh topologies. |

---

## Which layer are you extending?

HiveMind is plugins all the way down. Match what you're building to its plugin family:

| You want to add… | Plugin family | Start at |
|---|---|---|
| A new way bytes travel (e.g. a custom socket) | Network protocol | [Writing Plugins](writing-plugins.md) |
| A new AI backend that answers requests | Agent protocol | [Writing Plugins](writing-plugins.md) · [Plugin Architecture](../concepts/plugins.md) |
| A new place to store client credentials | Database | [Writing Plugins](writing-plugins.md) |
| A new rule for who may do what | Policy | [Security & Permissions](../concepts/security.md) |
| A handler for raw audio/binary streams | Binary protocol | [Audio Binary Protocol](../server/audio-binary-protocol.md) |

For the concepts behind these families, read [Plugin Architecture](../concepts/plugins.md) first.

---

**Next:** [Client Library](client-library.md) — the fastest way to talk to a hive from code.
