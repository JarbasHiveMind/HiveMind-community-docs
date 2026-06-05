# Mesh Topology

## Hubs and satellites

Every HiveMind deployment has at least one **hub** (also called a master) and one or more **satellites** (also called clients or slaves). The hub runs the AI back-end — either an OVOS skills server or a persona/LLM server — and accepts incoming connections from satellites. A satellite connects to the hub, sends utterances or messages, and receives responses.

```
[Satellite A] ──┐
[Satellite B] ───┤──→ [Hub] ──→ OVOS / LLM
[Satellite C] ───┘
```

The protocol itself is transport-agnostic: WebSocket is the default, but HTTP, MQTT, and other transports are supported via network protocol plugins.

## site_id

Every satellite carries a `site_id` — a free-form string identifying its physical location (e.g. `"living-room"`, `"kitchen"`). The hub injects `site_id` into OVOS message context, allowing skills to return location-aware responses. It also lets you target `BROADCAST` messages at a specific room.

## Nested hives

HiveMind hubs can connect to other hubs. A sub-hub acts as a client to its parent (sending `ESCALATE` up the chain) while acting as a server to its own satellites (sending `BROADCAST` down the chain). This creates a hierarchy:

```
[Root Hub]
    ├──→ [Sub-hub A]  ──→ [Satellite A1]
    │                 └──→ [Satellite A2]
    └──→ [Sub-hub B]  ──→ [Satellite B1]
```

### ESCALATE — upward routing

`ESCALATE` travels from a satellite up through sub-hubs toward the root. Use it when a node cannot handle a request locally and wants the parent to try.

### BROADCAST — downward routing

`BROADCAST` travels from a master down to all connected satellites. Supports a `target_site_id` field to reach only satellites in a specific location — for example, announcing dinner is ready in the kitchen.

### PROPAGATE — all-directions flood

`PROPAGATE` delivers a message to all connected nodes, both up and down the tree simultaneously. Useful for mesh-wide announcements and topology discovery (PING is always wrapped in PROPAGATE).

### QUERY — routed request with response

`QUERY` travels upward like `ESCALATE`, but expects a response back. Each node in the chain attempts to answer from its local agent; the first node that can answer returns a response routed back downstream to the originator. If no node can answer, the root returns a `hive.query.timeout` error response.

### CASCADE — scatter/gather

`CASCADE` floods in all directions like `PROPAGATE`, and every node that can answer sends a response back. The originator collects responses via `CascadeAggregator` and applies a select callback to pick the best answer. See [Protocol — CASCADE](protocol.md).

## Relay nodes

A **relay node** is a hub that is simultaneously connected upstream to a parent master. It serves its own downstream satellites while forwarding traffic in both directions.

A relay is created by calling `HiveMindListenerProtocol.bind_upstream(slave_protocol)` after the slave is bound to a bus. Once bound:

- `BROADCAST` and `PROPAGATE` from the upstream master are fanned out to all downstream clients.
- `QUERY` and `CASCADE` from the upstream master are also fanned out downstream.
- Downstream `PROPAGATE`, `ESCALATE`, `QUERY`, and `CASCADE` are forwarded upstream.

This bidirectional relay is transparent to both the satellites below and the master above. The chain can be arbitrarily deep:

```
[Root Hub]
    └──→ [Relay Hub]  ──→ [Satellite A]
                      └──→ [Satellite B]
```

A QUERY sent by Satellite A travels up through the Relay Hub. If the Relay Hub's local agent answers, the response returns directly. If not, the QUERY continues to the Root Hub. The response retraces the path back to Satellite A.

A CASCADE sent by Satellite A is forwarded to both Satellite B and up to the Root Hub (and its satellites). Responses flow back from all of them.

## Permissions across nested hives

Each hop enforces its own permission chain. A sub-hub can grant clients no more access than the root has granted the sub-hub. This creates a natural permission boundary: a guest sub-hub never inherits root capabilities simply by connecting.

## Use cases

**Multi-room home**: One root hub runs OVOS. Each room has a sub-hub that controls local devices. Satellites in each room connect to the sub-hub. Skills running on the root can BROADCAST to specific rooms.

**Shared household**: Two housemates each run their own OVOS instance (John and Jane). Both connect to a shared root hub (George) that controls shared smart-home devices. John and Jane route shared-home commands through George, but keep personal data (calendars, playlists) local to their own hubs.

**Guest access**: A guest sub-hub is created temporarily under the root. Guest satellites connect to the sub-hub. The root grants the sub-hub limited permissions; when the guest leaves, the sub-hub entry is removed.

```
[George / Root Hub]
    ├──→ [John's Hub] ──→ phone, personal skills
    ├──→ [Jane's Hub] ──→ phone, personal skills
    └──→ [Guest Hub]  ──→ voice satellites in living room
```

## Local hives (no network)

A "local hive" is a hub and satellite running on the same machine. This is useful when you want separate processes communicating via the HiveMind protocol without a real network hop — for example, a skill running as a satellite that delegates some intents to another OVOS instance on the same host.
