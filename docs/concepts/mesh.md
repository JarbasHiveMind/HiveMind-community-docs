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
