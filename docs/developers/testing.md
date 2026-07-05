# Testing Guide

Testing distributed software usually means standing up servers, opening sockets, and
praying the timing lines up. [`hivescope`](https://github.com/JarbasHiveMind/hivescope)
throws that whole ceremony out. You build a hive — a master, a couple of satellites,
maybe a relay — as plain Python objects wired directly together in memory. No real
sockets, no running processes, no network at all. Every `HiveMessage` that flows between
them is recorded, so you can assert exactly what was routed where, whether an ACL held,
and whether a session survived the trip. A whole multi-node topology becomes a fast,
ordinary unit test.

!!! abstract "In a nutshell"
    - Assemble a test topology from `MasterNode`, `SatelliteNode`, and `RelayNode` objects to validate message routing, session handling, ACL enforcement, and protocol compliance.
    - Every node carries a `MessageRecorder`; the master's `TestAgentProtocol` records every OVOS message injected onto its agent bus.
    - `hivescope.scenarios` ships pre-wired topology builders and `hivescope.assertions` ships ready-made checks; `pytest` fixtures wire the common ones up automatically.

`hivemind-test-harness` builds on top of hivescope for cross-repo stress and
multi-topology suites; to test a single client or skill, target hivescope directly.

---

## Install

```bash
pip install hivescope pytest
```

For OVOS skill-level tests backed by a live MiniCroft instance, install the
`ovos` extra:

```bash
pip install "hivescope[ovos]" pytest
```

---

## Overview

A test topology is assembled by a `TopologyBuilder` and made of nodes:

- **`MasterNode`** — wraps the `HiveMindListenerProtocol` (hivemind-core). Its
  `agent_protocol` is a `TestAgentProtocol` that records every OVOS message
  injected onto the agent bus.
- **`SatelliteNode`** — simulates a connected satellite client (slave protocol)
  with its own `internal_bus` (a `FakeBus`).
- **`RelayNode`** — a dual-role node (satellite upstream, master downstream).

Every node carries a **`MessageRecorder`** (`node.recorder`) that captures inbound
and outbound `HiveMessage`s as `RecordedMessage` entries.

`hivescope.scenarios` provides pre-wired builders (`single_satellite`,
`three_satellites`, `with_relay`, `star_topology`, `with_acl_enforcement`, …) and
`hivescope.assertions` provides ready-made checks. All scenario builders return a
**not-yet-started** builder — call `.start_all()` before testing and
`.stop_all()` in a `finally` block.

---

## Pattern A: Direct hivemind-core injection

Emit an OVOS message on the master's agent bus and assert it was seen there
(`emit_on_bus` simulates a skill response on hivemind-core).

```python
from hivescope import TopologyBuilder
from ovos_bus_client.message import Message

def test_direct_injection():
    b = TopologyBuilder()
    m = b.add_master("M0")
    b.start_all()
    try:
        m.emit_on_bus(Message("speak", {"utterance": "hello"}))

        m.agent_protocol.assert_injected("speak", count=1)
        last = m.agent_protocol.last_injected("speak")
        assert last.data["utterance"] == "hello"
    finally:
        b.stop_all()
```

---

## Pattern B: Satellite-to-hivemind-core round-trip

Send a `BUS` message from a satellite and assert the master injected the inner
OVOS utterance onto its agent bus. `single_satellite(allowed_types=[...])` grants
the satellite permission to inject that message type.

```python
from hivescope.scenarios import single_satellite
from ovos_bus_client.message import Message

def test_satellite_roundtrip():
    b = single_satellite(allowed_types=["recognizer_loop:utterance"])
    b.start_all()
    try:
        m = b.get_master("M0")
        s = b.get_satellite("S0")

        # SatelliteNode.send wraps a bare OVOS Message in a BUS HiveMessage
        s.send(Message("recognizer_loop:utterance",
                       {"utterances": ["what time is it?"]}))

        m.agent_protocol.assert_injected("recognizer_loop:utterance", count=1)
        injected = m.agent_protocol.last_injected("recognizer_loop:utterance")
        assert injected.data["utterances"] == ["what time is it?"]
    finally:
        b.stop_all()
```

---

## Pattern C: Multi-satellite topology

Use a multi-node scenario and assert traffic with `assertions.*` helpers. Here a
skill response routed to one satellite is delivered downstream as a `BUS`
message, which that satellite's recorder captures.

```python
from hivescope.scenarios import three_satellites
from hivescope.assertions import assert_message_received_by
from ovos_bus_client.message import Message

def test_downstream_routing():
    b = three_satellites()
    b.start_all()
    try:
        m = b.get_master("M0")
        s0 = b.get_satellite("S0")

        # Address a skill response at S0's peer; TestAgentProtocol reverse-routes
        # it back to that satellite via a BUS HiveMessage.
        m.emit_on_bus(Message(
            "speak",
            {"utterance": "hello S0"},
            context={"destination": [s0.peer]},
        ))

        assert_message_received_by(s0, "bus", count=1)
    finally:
        b.stop_all()
```

---

## Pattern D: Session and context verification

Attach an OVOS `Session` to the outgoing message and assert it survives injection
on hivemind-core.

```python
from hivescope.scenarios import single_satellite
from ovos_bus_client.message import Message
from ovos_bus_client.session import Session

def test_session_propagation():
    b = single_satellite(allowed_types=["recognizer_loop:utterance"])
    b.start_all()
    try:
        m = b.get_master("M0")
        s = b.get_satellite("S0")

        sess = Session("test-session-123")
        s.send(Message(
            "recognizer_loop:utterance",
            {"utterances": ["hello"]},
            context={"session": sess.serialize()},
        ))

        injected = m.agent_protocol.last_injected("recognizer_loop:utterance")
        assert injected.context["session"]["session_id"] == "test-session-123"
    finally:
        b.stop_all()
```

---

## Pytest fixtures

Enable hivescope's fixtures from your `tests/conftest.py`:

```python
pytest_plugins = ['hivescope.pytest_fixtures']
```

This provides `topology` (a started `TopologyBuilder`), `master_node`,
`satellite_node` (connected and handshook to `master_node`), `admin_satellite`,
and `restricted_satellite` — all auto-stopped on teardown. The
`restricted_satellite` is granted `allowed_types=["recognizer_loop:utterance"]`,
so its utterances reach the agent bus:

```python
from ovos_bus_client.message import Message

def test_with_fixtures(master_node, restricted_satellite):
    restricted_satellite.send(Message("recognizer_loop:utterance",
                                      {"utterances": ["hi"]}))
    master_node.agent_protocol.assert_injected(
        "recognizer_loop:utterance", count=1
    )
```

---

## Common assertions

```python
# OVOS messages injected onto the master's agent bus
m.agent_protocol.assert_injected("speak", count=1)
m.agent_protocol.assert_not_injected("recognizer_loop:utterance")
last = m.agent_protocol.last_injected("speak")
assert last.data["utterance"] == "hello world"

# HiveMessages routed through any node (recorder; msg_type is the lowercase
# HiveMessageType value, e.g. "bus", "propagate", "handshake")
node.recorder.assert_received("bus", count=1)
node.recorder.assert_not_received("propagate")

# Block until a message arrives (returns the RecordedMessage or None on timeout)
rec = master_node.wait_for("handshake", timeout=5)

# Ready-made helpers from hivescope.assertions
from hivescope.assertions import (
    assert_handshake_complete,
    assert_bus_message_routed,
    assert_message_received_by,
)
assert_bus_message_routed(m, count=1)
```

---

## Message flow in tests

```
SatelliteNode.send(HiveMessage | Message)
  ↓  (Message is wrapped in a BUS HiveMessage)
MasterNode.hm_protocol.handle_message(message, connection)
  ↓
HiveMindListenerProtocol routes by HiveMessageType:
  ├─ BUS:       inject onto the agent bus (TestAgentProtocol records it)
  ├─ PROPAGATE: relay to peers
  └─ INTERCOM:  satellite-to-satellite tunnel
  ↓
TestAgentProtocol.bus (FakeBus) — skill response emitted here
  (or simulate one with MasterNode.emit_on_bus(message))
  ↓
TestAgentProtocol reverse-routes by context["destination"]
  ↓  → BUS HiveMessage sent downstream
SatelliteNode.recorder records it; SatelliteNode.internal_bus fires
  ↓
Test assertion (recorder / agent_protocol.injected / assertions.*)
```

---

## CI integration

```yaml
# .github/workflows/tests.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v4
        with:
          python-version: "3.10"
      - run: pip install -e . hivescope pytest pytest-cov
      - run: pytest tests/ -v --cov=mypackage --cov-report=term-missing
```

---

## Running locally

```bash
# Install test dependencies
pip install hivescope pytest pytest-cov

# Run all tests
pytest tests/ -v

# Run with coverage
pytest tests/ -v --cov=mypackage --cov-report=term-missing

# Run specific test
pytest tests/test_routing.py -v
```

---

## See also

- [Client Library](client-library.md) — `HiveMessageBusClient` API
- [Writing Plugins](writing-plugins.md) — building HiveMind plugins
- [Protocol Concepts](../concepts/protocol.md) — message types and routing

---

## Source

Validated against the HiveMind source:

- [`hivescope/scenarios.py`](https://github.com/JarbasHiveMind/hivescope/blob/HEAD/hivescope/scenarios.py) — `single_satellite`, `three_satellites`, `with_relay`, `star_topology`, `with_acl_enforcement` and the other pre-wired builders
- [`hivescope/assertions.py`](https://github.com/JarbasHiveMind/hivescope/blob/HEAD/hivescope/assertions.py) — `assert_message_received_by`, `assert_bus_message_routed`, `assert_handshake_complete` and the rest of the ready-made checks
- [`hivemind-test-harness/docs/index.md`](https://github.com/JarbasHiveMind/hivemind-test-harness/blob/HEAD/docs/index.md) — cross-repo / multi-topology stress suites built on top of hivescope
