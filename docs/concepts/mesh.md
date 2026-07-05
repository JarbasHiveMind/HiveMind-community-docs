# Mesh Topology

Start with one server and a handful of devices and you have the whole picture most
people ever need. But nothing says a HiveMind stops there. A server can itself connect
*upward* to a bigger server, the way a room reports to a household and a household to a
whole building. Give every housemate their own hivemind-core, wire them all into a
shared one that owns the front-door lock and the living-room speakers, and you have a
tree — questions climb toward whoever can answer them, announcements roll back down to
whoever needs to hear. The pieces are the same as before; you've just stacked them.

!!! abstract "In a nutshell"
    - Every hivemind-core runs the brain and accepts satellites. Stack two of them and the lower one wears two hats at once: a client to its parent, a server to its own satellites.
    - Messages know which way to go: `ESCALATE` and `QUERY` climb toward the root, `BROADCAST` rolls down to the leaves, and `PROPAGATE`/`CASCADE` flood everywhere.
    - Trust doesn't leak between floors — every hop re-checks its own permission list, so a child hive can never hand out more than the root gave it.

## hivemind-core and satellites

Start with the simplest shape, the one nearly everyone runs. At the centre is a single
**hivemind-core instance** (the protocol also calls it a *master*); around it sit one or
more **satellites** (also called *clients* or *slaves*). hivemind-core holds the brain —
an OVOS skills server or a persona/LLM — and waits for connections. A satellite dials in,
sends what it heard, and gets an answer back. That's the whole loop:

```
[Satellite A] ──┐
[Satellite B] ───┤──→ [hivemind-core] ──→ OVOS / LLM
[Satellite C] ───┘
```

Notice what the diagram *doesn't* pin down: the arrows aren't any particular wire. The
protocol is transport-agnostic — WebSocket is the default, but HTTP, MQTT, and others plug
in underneath without changing the picture.

---

## site_id

One small tag makes a satellite location-aware. Every satellite carries a `site_id` — a
free-form label for where it physically is, like `"living-room"` or `"kitchen"`.
hivemind-core slips that tag into the OVOS message context, so a skill can answer
differently depending on the room, and so you can aim a `BROADCAST` at one place instead
of the whole house. Hold onto `site_id` — it's what makes "announce dinner *in the
kitchen*" possible a few sections from now.

---

## Nested hives

Here's where the single-server picture opens up. A hivemind-core instance doesn't only
accept satellites — it can also become a satellite of *another* hivemind-core. When it
does, it wears two hats at once: a client to its parent (sending `ESCALATE` up) and a
server to its own satellites (sending `BROADCAST` down). Stack a few of those and you get
a tree:

```
[Root hivemind-core]
    ├──→ [Child A]  ──→ [Satellite A1]
    │               └──→ [Satellite A2]
    └──→ [Child B]  ──→ [Satellite B1]
```

Once there's a tree, a message needs to know which way to travel through it — and that's
exactly what the routing verbs from the [Protocol](protocol.md) page are for. Each one is
a direction of travel. Watch them move:

### ESCALATE — upward routing

`ESCALATE` travels from a satellite up through intermediate hivemind-core instances toward the root. Use it when a node cannot handle a request locally and wants the parent to try.

![ESCALATE climbs toward the root](https://raw.githubusercontent.com/JarbasHiveMind/HiveMind-core/dev/resources/escalate.gif)

### BROADCAST — downward routing

`BROADCAST` travels from a hivemind-core instance down to all connected satellites. Supports a `target_site_id` field to reach only satellites in a specific location — for example, announcing dinner is ready in the kitchen.

![BROADCAST rolls down to every satellite](https://raw.githubusercontent.com/JarbasHiveMind/HiveMind-core/dev/resources/broadcast.gif)

### PROPAGATE — all-directions flood

`PROPAGATE` delivers a message to all connected nodes, both up and down the tree simultaneously. Useful for mesh-wide announcements and topology discovery (PING is always wrapped in PROPAGATE).

![PROPAGATE floods the whole mesh](https://raw.githubusercontent.com/JarbasHiveMind/HiveMind-core/dev/resources/propagate.gif)

### QUERY — routed request with response

`QUERY` travels upward like `ESCALATE`, but expects a response back. Each node in the chain attempts to answer from its local agent; the first node that can answer returns a response routed back downstream to the originator. If no node can answer, the root returns a `hive.query.timeout` error response.

### CASCADE — scatter/gather

`CASCADE` floods in all directions like `PROPAGATE`, and every node that can answer sends a response back. The originator collects responses via `CascadeAggregator` and applies a select callback to pick the best answer. See [Protocol — CASCADE](protocol.md).

> **Note:** `RENDEZVOUS` (rendezvous / wormhole relay) is a **reserved** message type. It is defined in the protocol but not wired into core: there is no routing handler for it, so it falls through to the unknown-message stub. Don't design around it.

---

## Relay nodes

A child in the tree usually answers for itself. But sometimes you want a middle node that
*doesn't* — one that just passes traffic through in both directions, like a repeater
extending the hive one hop further. That's a **relay node**: a hivemind-core instance
wired to a parent upstream and satellites downstream, forwarding between them.

You create one by calling `HiveMindListenerProtocol.bind_upstream(slave_protocol)` after
the slave is bound to a bus. From then on it fans traffic both ways:

- `BROADCAST` and `PROPAGATE` from the upstream master are fanned out to all downstream clients.
- `QUERY` and `CASCADE` from the upstream master are also fanned out downstream.
- Downstream `PROPAGATE`, `ESCALATE`, `QUERY`, and `CASCADE` are forwarded upstream.

This bidirectional relay is transparent to both the satellites below and the master above. The chain can be arbitrarily deep:

```
[Root hivemind-core]
    └──→ [Relay node]  ──→ [Satellite A]
                       └──→ [Satellite B]
```

A QUERY sent by Satellite A travels up through the relay node. If the relay node's local agent answers, the response returns directly. If not, the QUERY continues to the root. The response retraces the path back to Satellite A.

A CASCADE sent by Satellite A is forwarded to both Satellite B and up to the root (and its satellites). Responses flow back from all of them.

---

## Permissions across nested hives

Stacking servers raises a fair worry: if a guest connects to a child hive, could it reach
past that child and command the root? No — and the reason is worth internalising. Each hop
enforces its *own* permission chain. A child can hand its clients no more than the root
handed the child. So capability shrinks as you go down, never grows: a guest instance
inherits nothing just by plugging in.

??? note "Advanced: why the boundary actually holds"
    The boundary is not just a convention — it is enforced at **every hop**. A message arriving at a hivemind-core instance is checked against *that instance's* `allowed_types` ACL by `MessageTypeACLPolicy` (`HiveMind-core/hivemind_core/policy.py`) before it is forwarded onward. So when a child instance forwards a satellite's message upstream, it is itself a client of the root and is re-checked against the permissions the root granted *it*. A capability the root never granted the child instance can never be smuggled through by a downstream device.

    For concrete multi-hop routing tables and worked topology examples, see the [hivemind-test-harness docs](https://github.com/JarbasHiveMind/hivemind-test-harness/blob/HEAD/docs/index.md) (`07-message-routing.md` and `03-topologies.md` in particular).

---

## Use cases

That's a lot of machinery; here's what it's *for*. Three shapes people actually build,
each leaning on a different piece of what you just read.

**Multi-room home**: One root hivemind-core runs OVOS. Each room has a child instance that controls local devices. Satellites in each room connect to the child instance. Skills running on the root can BROADCAST to specific rooms.

**Shared household**: Two housemates each run their own OVOS instance (John and Jane). Both connect to a shared root hivemind-core (George) that controls shared smart-home devices. John and Jane route shared-home commands through George, but keep personal data (calendars, playlists) local to their own instances.

**Guest access**: A guest instance is created temporarily under the root. Guest satellites connect to it. The root grants the guest instance limited permissions; when the guest leaves, the entry is removed.

```
[George / Root hivemind-core]
    ├──→ [John's instance] ──→ phone, personal skills
    ├──→ [Jane's instance] ──→ phone, personal skills
    └──→ [Guest instance]  ──→ voice satellites in living room
```

---

## Local hives (no network)

One last shape, and it barely looks like a network at all. A "local hive" is a
hivemind-core instance and a satellite running on the *same machine* — no real hop between
them. Why bother? Because it lets two processes on one host talk over the HiveMind
protocol as if they were across the room: a skill can run as a satellite and hand certain
intents off to a second OVOS instance next door, all without a wire.

---

**Next:** [Protocol](protocol.md) for the message types in detail, or [Security](security.md) for how permission chains compose across hops.

---

## Source

Validated against the HiveMind source:

- [`hivemind_core/protocol.py`](https://github.com/JarbasHiveMind/HiveMind-core/blob/HEAD/hivemind_core/protocol.py) — `bind_upstream`, relay fan-out, and ESCALATE/BROADCAST/PROPAGATE/QUERY/CASCADE routing
- [`hivemind_core/policy.py`](https://github.com/JarbasHiveMind/HiveMind-core/blob/HEAD/hivemind_core/policy.py) — per-hop `allowed_types` re-check that enforces the permission boundary
- [hivemind-test-harness `docs/`](https://github.com/JarbasHiveMind/hivemind-test-harness/blob/HEAD/docs/index.md) — multi-hop routing tables and topology examples
