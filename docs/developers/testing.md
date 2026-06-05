# Testing Guide

`hivemind-test-harness` provides an in-process test topology for validating message routing, session management, and protocol compliance — without physical hardware or network connectivity.

## Install

```bash
pip install hivemind-test-harness pytest
```

## Overview

The harness runs a complete HiveMind topology in-process:

- **`TestTopology`** — manages the in-process hub (protocol handlers + OVOS core)
- **`TestSatellite`** — simulates a connected satellite client
- **`CaptureSession`** — records messages for assertion
- **`AgentProtocol`** — wraps OVOS for message injection and capture

## Pattern A: Direct hub injection

Inject a message directly into the hub's OVOS bus and assert responses.

```python
from hivemind_test_harness import TestTopology
from ovos_bus_client.message import Message

def test_direct_injection():
    with TestTopology() as harness:
        master = harness.master

        msg = Message(
            "recognizer_loop:utterance",
            {"utterances": ["hello"]}
        )
        master.internal_bus.emit(msg)

        speak_msgs = master.agent_protocol.captured("speak")
        assert len(speak_msgs) > 0
```

## Pattern B: Satellite-to-hub round-trip

Test the full request-response flow: satellite → hub → skill → satellite.

```python
from hivemind_test_harness import TestTopology, TestSatellite
from ovos_bus_client.message import Message
from hivemind_bus_client.message import HiveMessage, HiveMessageType

def test_satellite_roundtrip():
    with TestTopology() as harness:
        sat = TestSatellite(harness.master)

        msg = HiveMessage(
            HiveMessageType.BUS,
            Message("recognizer_loop:utterance",
                    {"utterances": ["what time is it?"]})
        )
        sat.send(msg)

        injected = harness.master.agent_protocol.last_injected(
            "recognizer_loop:utterance"
        )
        assert injected.context["peer"] == sat._connection.peer
        assert injected.context["session"]["session_id"] == "test-session"
```

## Pattern C: Multi-satellite topology

Test INTERCOM messaging between two satellites.

```python
def test_intercom():
    with TestTopology() as harness:
        sat1 = TestSatellite(harness.master)
        sat2 = TestSatellite(harness.master)

        msg = HiveMessage(
            HiveMessageType.INTERCOM,
            {"from": sat1.peer, "to": sat2.peer, "data": "hello"}
        )
        sat1.send(msg)

        intercom_msgs = []
        sat2.internal_bus.on("hive:intercom", intercom_msgs.append)

        assert len(intercom_msgs) > 0
```

## Pattern D: Session and context verification

Verify that session IDs and context keys are set correctly.

```python
from ovos_bus_client.session import Session

def test_session_propagation():
    with TestTopology() as harness:
        sat = TestSatellite(harness.master)
        sess = Session("test-session-123")

        msg = HiveMessage(
            HiveMessageType.BUS,
            Message(
                "recognizer_loop:utterance",
                {"utterances": ["hello"]},
                context={"session": sess.serialize()}
            )
        )
        sat.send(msg)

        injected = harness.master.agent_protocol.last_injected(
            "recognizer_loop:utterance"
        )
        assert injected.context["session"]["session_id"] == "test-session-123"
        assert injected.context["peer"] == sat._connection.peer
        assert injected.context["source"] == sat._connection.peer
        assert injected.context["destination"] == "skills"
```

## Common assertions

```python
# All captured messages of a type
captured = master.agent_protocol.captured("speak")
assert len(captured) > 0

# Last message of a type
last_speak = master.agent_protocol.last_captured("speak")
assert last_speak.data["utterance"] == "hello world"

# Session state
msg = master.agent_protocol.last_injected("recognizer_loop:utterance")
assert msg.context["session"]["session_id"] is not None
assert msg.context["peer"] == expected_peer

# Blacklist enforcement
msg = master.agent_protocol.last_injected("recognizer_loop:utterance")
assert "bad.skill" not in msg.context["session"]["blacklisted_skills"]
```

## Message flow in tests

```
TestSatellite.send(HiveMessage)
  ↓
TestTopology.master.websocket_handler.on_message()
  ↓
HiveMindListenerProtocol.handle_message()
  ↓
  ├─ BUS: inject to OVOS bus
  ├─ PROPAGATE: relay to peers
  └─ INTERCOM: end-to-end encrypted tunnel
  ↓
OVOS IntentService.handle_utterance()
  ↓
Skill match → emit speak / intent.handled
  ↓
HiveMindListenerProtocol.handle_internal_mycroft()
  ↓
TestSatellite.receive()
  ↓
Test assertion
```

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
      - run: pip install -e . hivemind-test-harness pytest pytest-cov
      - run: pytest tests/ -v --cov=mypackage --cov-report=term-missing
```

## Running locally

```bash
# Install test dependencies
pip install hivemind-test-harness pytest pytest-cov

# Run all tests
pytest tests/ -v

# Run with coverage
pytest tests/ -v --cov=mypackage --cov-report=term-missing

# Run specific test
pytest tests/test_routing.py -v
```

## See also

- [Protocol Concepts](../concepts/protocol.md) — message types and routing
- [Client Library](client-library.md) — `HiveMessageBusClient` API
