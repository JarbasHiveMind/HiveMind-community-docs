# Testing Guide

HiveMind provides a comprehensive test harness for validating integration behavior, message routing, and protocol compliance in an in-process topology without requiring external hardware.

**Source**: `hivemind-test-harness` package

## Overview

The test harness simulates a complete HiveMind topology in-process:

- **Master Node** — Runs the hub (HiveMind protocol handlers + OVOS core)
- **Satellite Nodes** — Lightweight clients simulating real satellites
- **Test Observer** — Captures messages for assertion

This enables rapid development and testing without physical devices.

## Test Patterns

### Pattern A: Direct Hub Message Injection

The simplest pattern — inject a message directly into the hub's OVOS bus and assert responses.

```python
from hivemind_test_harness import TestTopology

with TestTopology() as harness:
    master = harness.master

    # Inject an utterance into the hub's OVOS bus
    msg = Message(
        "recognizer_loop:utterance",
        {"utterances": ["hello"]}
    )
    master.internal_bus.emit(msg)

    # Assert that a skill was activated
    speak_msgs = master.agent_protocol.captured("speak")
    assert len(speak_msgs) > 0
    assert "hello" in speak_msgs[0].data.get("utterance", "").lower()
```

### Pattern B: Satellite-to-Hub Communication

Test full request-response flow from a satellite through the hub to a skill and back.

```python
from hivemind_test_harness import TestTopology, TestSatellite
from ovos_bus_client.message import Message
from hivemind_bus_client.message import HiveMessage, HiveMessageType

with TestTopology() as harness:
    sat = TestSatellite(harness.master)

    # Satellite sends an utterance
    msg = HiveMessage(
        HiveMessageType.BUS,
        Message("recognizer_loop:utterance", {"utterances": ["what time is it?"]})
    )
    sat.send(msg)

    # Assert hub processes it
    injected = harness.master.agent_protocol.last_injected("recognizer_loop:utterance")
    assert injected.context["peer"] == sat._connection.peer
    assert injected.context["session"]["session_id"] == "test-session"

    # Assert satellite receives the response
    responses = []
    sat.internal_bus.once("speak", responses.append)
    # (skill processing happens automatically)

    assert len(responses) > 0
    assert "time" in responses[0].data.get("utterance", "").lower()
```

### Pattern C: Multi-Satellite Topology

Test inter-satellite communication and message propagation.

```python
with TestTopology() as harness:
    sat1 = TestSatellite(harness.master)
    sat2 = TestSatellite(harness.master)

    # Satellite 1 sends an INTERCOM message to Satellite 2
    msg = HiveMessage(
        HiveMessageType.INTERCOM,
        {"from": sat1.peer, "to": sat2.peer, "data": "hello"}
    )
    sat1.send(msg)

    # Assert Satellite 2 receives it
    intercom_msgs = []
    sat2.internal_bus.on("hive:intercom", intercom_msgs.append)
    # trigger processing...

    assert len(intercom_msgs) > 0
```

### Pattern D: Message Routing Verification

Validate session ID and context key propagation throughout the message chain.

```python
from ovos_bus_client.session import Session

with TestTopology() as harness:
    sat = TestSatellite(harness.master)
    sess = Session("test-session-123")

    # Send with explicit session
    msg = HiveMessage(
        HiveMessageType.BUS,
        Message(
            "recognizer_loop:utterance",
            {"utterances": ["hello"]},
            context={"session": sess.serialize()}
        )
    )
    sat.send(msg)

    # Verify session ID is preserved at hub
    injected = harness.master.agent_protocol.last_injected("recognizer_loop:utterance")
    assert injected.context["session"]["session_id"] == "test-session-123"

    # Verify peer and destination are set correctly
    assert injected.context["peer"] == sat._connection.peer
    assert injected.context["source"] == sat._connection.peer
    assert injected.context["destination"] == "skills"
```

## CI Integration

The test harness integrates with GitHub Actions for automated testing:

```yaml
# .github/workflows/tests.yml
name: Unit Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-python@v2
        with:
          python-version: "3.10"
      - run: pip install -e . pytest
      - run: pytest test/ -v --cov=hivemind_bus_client
```

## Key Assertions

Common test assertions for HiveMind behavior:

```python
# Message capture
captured = master.agent_protocol.captured("speak")
assert len(captured) > 0

# Last message of a type
last_speak = master.agent_protocol.last_captured("speak")
assert last_speak.data["utterance"] == "hello world"

# Session verification
msg = master.agent_protocol.last_injected("recognizer_loop:utterance")
assert msg.context["session"]["session_id"] is not None
assert msg.context["peer"] == expected_peer
assert msg.context["source"] == expected_peer

# Blacklist enforcement
msg = master.agent_protocol.last_injected("recognizer_loop:utterance")
assert "bad.skill" not in msg.context["session"]["blacklisted_skills"]
```

## Running Tests Locally

```bash
# Install test dependencies
pip install hivemind-test-harness pytest pytest-cov

# Run all tests
pytest test/ -v

# Run specific test file
pytest test/test_routing.py -v

# Run with coverage
pytest test/ -v --cov=mypackage --cov-report=term-missing
```

## Test Harness Architecture

**Source**: `hivemind-test-harness/docs/`

### Core Components

1. **`TestTopology`** — Manages the in-process hub, protocol handlers, and cleanup
2. **`TestSatellite`** — Simulates a connected satellite client
3. **`CaptureSession`** — Records all messages on the internal bus for assertion
4. **`AgentProtocol`** — Wrapper around OVOS protocol for message injection/capture

### Message Flow in Tests

```
Test Case
  ↓
TestSatellite.send(HiveMessage)
  ↓
TestTopology.master.websocket_handler.on_message()
  ↓
HiveMindListenerProtocol.handle_message()
  ↓
  ├─ BUS: inject to OVOS bus
  ├─ PROPAGATE: relay to peers
  └─ INTERCOM: end-to-end encrypted message
  ↓
OVOS IntentService.handle_utterance()
  ↓
Skill match → emit speak / intent.handled
  ↓
HiveMindListenerProtocol.handle_internal_mycroft()
  ↓
TestSatellite.receive() + internal_bus.emit()
  ↓
Test assertion on captured messages
```

## See Also

- [Protocol: Message Routing & Context Keys](/HiveMind-community-docs/docs/04_protocol.md#session-management--context-keys)
- [Client Libraries: HiveMessageBusClient](/HiveMind-community-docs/docs/11_devs.md)
- [Developers: OVOS Integration](/HiveMind-community-docs/docs/ovos_integration.md)
