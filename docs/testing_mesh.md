# A multi-node test mesh

A single hub exercises client-to-hub traffic, but not routing between nodes: escalation up a tree, propagation across subtrees, node-to-node INTERCOM, and what happens when a relay drops. The [hivetree](https://github.com/JarbasHiveMind/hivetree) project builds a real tree of hubs with one `docker compose up`, so this traffic can be exercised end to end.

## The shape

    m0   root master
   /  \
  r1    r2   relays, each connected upstream to m0

Each node runs one container: a local message bus and `hivemind-core`. The two relays connect upstream to the root, which turns them into relays for the leaves beneath them ([Nested Hives](15_nested.md)). Leaf clients connect to a relay and send traffic that crosses real hops.

The mesh runs the latest published HiveMind alphas. To test an unreleased build, drop its wheel into the image and rebuild — the rest of the tree is unchanged.

## What it exercises

The included probes drive the routing message types across the tree and check the result:

- **Escalate** — a leaf sends an utterance up to the root.
- **Propagate** — a message from a leaf under one relay reaches a leaf under the other, across the root.
- **INTERCOM** — a leaf sends an encrypted, signed message to one other leaf, which only that leaf can read.
- **Relay loss** — one relay is stopped, and the other subtree keeps working.

The tree is also where a routing or delivery finding is reproduced before it is believed, and where a fix is re-checked after it lands. It complements the unit and end-to-end tests in `hivemind-core`, which check one hub against itself and can not reach cross-node routing.

## Setup notes

Two settings trip up a first run:

A node's network and agent plugins are named by their entry-point names in `server.json`, not their package names. The websocket network plugin is `hivemind-websocket-plugin`.

`allowed_types` is deny-by-default and also gates the inner message type of a `BUS`, `QUERY`, or `ESCALATE` payload. A client that sends an utterance needs `recognizer_loop:utterance` allowed, not only `bus` ([Permissions](16_permissions.md)).

---
[← Libraries](11_devs.md) · [Home](index.md) · [ELI5 →](gpt_eli5.md)
