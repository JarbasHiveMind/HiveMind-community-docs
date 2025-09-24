# Agent Plugins for HiveMind

HiveMind is **just a transport protocol**, it moves **BUS messages** between devices and services.
On its own, HiveMind doesn’t decide what those messages *mean* or what to *do* with them.

That responsibility is delegated to **Agent Plugins**.

Agents are the abstraction layer that give purpose to a HiveMind node: they receive incoming messages and define how to handle them.

---

## What is an Agent Plugin?

An **Agent Plugin** tells HiveMind what to do once a message arrives.

Most agents handle **text-based input/output** (e.g., utterances, chat-like responses), but the system is flexible enough that agents can perform completely different tasks.

Each node in a HiveMind network loads exactly one agent plugin, which defines that node’s behavior.

---

## Available Agent Plugins

### 🔹 OVOS Bus Client Agent

* Provided by [`ovos-bus-client`](https://github.com/OpenVoiceOS/ovos-bus-client)
* Connects `hivemind-core` to a local `ovos-core` instance (must run on the same device).
* Effectively turns an existing OVOS installation into a **HiveMind Hub**.
* Satellites can forward utterances to the hub, and the hub returns responses via HiveMind.

---

### 🔹 Persona Agent

* Provided by [`ovos-persona`](https://github.com/OpenVoiceOS/ovos-persona)
* Loads a **persona** (often backed by an LLM or chatbot) instead of a full OVOS core.
* Allows lightweight HiveMind nodes to act as conversational agents without needing the full OVOS stack.
* Useful for [multi-agent experimentation](https://openvoiceos.github.io/ovos-technical-manual/150-advanced_solvers/#collaborative-agents-via-mos-mixture-of-solvers), or when you want a node that speaks with a distinct "personality."

Learn more about [OVOS personas](https://openvoiceos.github.io/ovos-technical-manual/150-personas/)

---

### 🔹 Media Player Agent

* Special-case agent focused on **media playback control**.
* Does **not** handle natural language queries.
* Instead, it reacts to OVOS bus messages related to media (play, pause, stop, next, etc.).
* Exposes HiveMind devices as **media players** to frameworks such as:
  * [Home Assistant](https://www.home-assistant.io/)
  * [Music Assistant](https://music-assistant.io/)
* Implemented as an agent plugin for technical consistency, but demonstrates HiveMind’s flexibility: not all agents need to be conversational.

---

## Why Agent Plugins Matter

* They **separate transport from behavior**

  * HiveMind moves messages around.
  * Agents decide what to do with them.

* They make HiveMind **extensible**

  * Want a chatbot? Use the Persona Agent.
  * Want a full assistant hub? Use the OVOS Bus Client Agent.
  * Want a network-controlled speaker? Use the Media Player Agent.

* They illustrate the design philosophy:
  **HiveMind doesn’t care what your node *is*, only that it can communicate.**

