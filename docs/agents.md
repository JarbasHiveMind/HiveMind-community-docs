# Agent Plugins for HiveMind

HiveMind is a transport protocol. It moves BUS messages between devices and services. On its own, HiveMind does not decide what those messages mean or what to do with them.

That responsibility belongs to Agent Plugins.

Agents give a HiveMind node its purpose: they receive incoming messages and define how to handle them.

---

## What is an Agent Plugin?

An Agent Plugin tells HiveMind what to do once a message arrives.

Most agents handle text-based input and output, such as utterances and chat-like responses, but the system is flexible enough that agents can perform other tasks entirely.

Each node in a HiveMind network loads exactly one agent plugin, which defines that node's behavior.

---

## Available agent plugins

### OVOS Bus Client Agent

- Provided by [`ovos-bus-client`](https://github.com/OpenVoiceOS/ovos-bus-client).
- Connects `hivemind-core` to a local `ovos-core` instance, which must run on the same device.
- Turns an existing OVOS installation into a HiveMind hub.
- Satellites forward utterances to the hub, and the hub returns responses over HiveMind.

### Persona Agent

- Provided by [`ovos-persona`](https://github.com/OpenVoiceOS/ovos-persona).
- Loads a persona, often backed by an LLM or chatbot, instead of a full OVOS core.
- Lets lightweight HiveMind nodes act as conversational agents without the full OVOS stack.
- Useful for [multi-agent experimentation](https://openvoiceos.github.io/ovos-technical-manual/150-advanced_solvers/#collaborative-agents-via-mos-mixture-of-solvers), or when you want a node with a distinct personality.

Learn more about [OVOS personas](https://openvoiceos.github.io/ovos-technical-manual/150-personas/).

### Media Player Agent

- A special-case agent focused on media playback control.
- Does not handle natural language queries.
- Reacts to OVOS bus messages related to media, such as play, pause, stop, and next.
- Exposes HiveMind devices as media players to frameworks such as [Home Assistant](https://www.home-assistant.io/).
- Implemented as an agent plugin for technical consistency, though it shows that not every agent needs to be conversational.

---

## Why agent plugins matter

- They separate transport from behavior: HiveMind moves messages around, and agents decide what to do with them.
- They make HiveMind extensible: use the Persona Agent for a chatbot, the OVOS Bus Client Agent for a full assistant hub, or the Media Player Agent for a network-controlled speaker.
- They reflect the design: HiveMind does not care what your node is, only that it can communicate.

---
[← Configuration](config.md) · [Home](index.md) · [Binary →](binary_plugins.md)
