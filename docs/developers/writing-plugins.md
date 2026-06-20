# Writing Plugins

HiveMind Core is assembled entirely from plugins discovered at runtime by the
**HiveMind Plugin Manager** (HPM) — see [Plugin Architecture](../concepts/plugins.md)
for the operator-facing view. This page is the developer-facing companion: how to
author your own plugin for each of the five families.

## The plugin-manager model

HPM is built on Python entry points. Each plugin family has a dedicated
**entry-point group** and a matching **factory** in the `hivemind_plugin_manager`
package. The groups are the values of the `HiveMindPluginTypes` enum
(`hivemind_plugin_manager/__init__.py`):

| Family | Entry-point group | Factory | Base class |
|---|---|---|---|
| Network protocol | `hivemind.network.protocol` | `NetworkProtocolFactory` | `NetworkProtocol` |
| Agent protocol | `hivemind.agent.protocol` | `AgentProtocolFactory` | `AgentProtocol` |
| Binary data handler | `hivemind.binary.protocol` | `BinaryDataHandlerProtocolFactory` | `BinaryDataHandlerProtocol` |
| Database | `hivemind.database` | `DatabaseFactory` | `AbstractDB` / `AbstractRemoteDB` |
| Policy | `hivemind.policy` | `PolicyPluginFactory` | `PolicyPlugin` |

Discovery is uniform: `find_plugins(plug_type)` scans the entry-point group and
returns a `dict` mapping each entry-point **name** (the string in your
`[project.entry-points]` declaration) to the loaded class. The factory's
`get_class(name)` looks the class up, and `create(...)` instantiates it with the
runtime keyword arguments (`config`, `hm_protocol`, and family-specific extras).

```python
from hivemind_plugin_manager import find_plugins, HiveMindPluginTypes

# {'hivemind-websocket-plugin': <class '...HiveMindWebsocketProtocol'>}
print(find_plugins(HiveMindPluginTypes.NETWORK_PROTOCOL))
```

So authoring a plugin is two steps: (1) subclass the right base class and implement
its contract, (2) register the class under the right entry-point group so
`find_plugins` can see it. Each section below gives both.

Every protocol base derives from a shared `_SubProtocol` dataclass
(`hivemind_plugin_manager/protocols.py`) and so receives, for free:

- `self.config` — the plugin's config dict
- `self.hm_protocol` — the `HiveMindListenerProtocol` (the hub) once wired up
- `self.identity` — the node `NodeIdentity`
- `self.database` — the active client `AbstractDB`, via the hub
- `self.clients` — the connected `HiveMindClientConnection` map, via the hub
- `self.callbacks` — `ClientCallbacks` (`on_connect` / `on_disconnect` /
  `on_invalid_key` / `on_invalid_protocol`)

---

## 1. Network protocol

A network protocol is the transport: it accepts satellite connections and feeds
their `HiveMessage` traffic into the hub.

- **Entry-point group:** `hivemind.network.protocol`
- **Base class:** `NetworkProtocol` (`hivemind_plugin_manager/protocols.py`)
- **Contract:** implement the single abstract method:

  ```python
  @abc.abstractmethod
  def run(self):
      ...
  ```

  `run()` starts the transport (bind a socket / start a server loop) and keeps it
  running, accepting client connections and dispatching their messages to the hub.
  It is invoked by `NetworkProtocolFactory.create(...)` consumers in
  `hivemind-core`. In addition to the shared `_SubProtocol` members, a network
  protocol exposes `self.agent_protocol` (the active `AgentProtocol`, via the hub).

Real implementation: `hivemind-websocket-protocol`
(`HiveMindWebsocketProtocol`, a Tornado WebSocket server).

### Minimal skeleton

```python
# my_network_protocol/__init__.py
from hivemind_plugin_manager.protocols import NetworkProtocol


class MyNetworkProtocol(NetworkProtocol):
    def run(self):
        host = self.config.get("host", "0.0.0.0")
        port = self.config.get("port", 5678)
        # bind your transport here, accept client connections,
        # and feed received HiveMessages into self.hm_protocol
        ...
```

```toml
# pyproject.toml
[project.entry-points."hivemind.network.protocol"]
"my-network-plugin" = "my_network_protocol:MyNetworkProtocol"
```

---

## 2. Agent protocol

An agent protocol is the AI back-end: it answers natural-language queries arriving
from the mesh.

- **Entry-point group:** `hivemind.agent.protocol`
- **Base class:** `AgentProtocol` (`hivemind_plugin_manager/protocols.py`)
- **Contract:** implement the single abstract method:

  ```python
  @abc.abstractmethod
  def natural_language_query(self, utterance: str,
                             lang: str) -> Iterator[Optional[str]]:
      ...
  ```

  This is a **generator**. `yield` each answer chunk (the text of one `speak`) as
  it is produced, then `yield None` once to signal end-of-query. Yielding `None`
  immediately with no chunks means the agent has no answer — the node then
  escalates the query upstream instead of stalling. Streaming lets a satellite
  start speaking the first sentence while the rest is still being generated. This
  is the mandatory seam that `hivemind-core`'s QUERY/CASCADE handlers consume. An
  `AgentProtocol` also carries `self.bus` (a `FakeBus`/`MessageBusClient`) in
  addition to the shared `_SubProtocol` members.

Real implementations:

- `hivemind-ovos-agent-plugin` — `OVOSAgentProtocol` (bridges to an OpenVoiceOS
  message bus; yields each `speak` and `None` when the utterance is handled).
- `hivemind-persona-agent-plugin` — `PersonaAgentProtocol` (an `ovos-persona`
  LLM/solver agent; yields sentences as the model produces them).
- `hivemind-a2a-agent-plugin` — `A2AAgentProtocol` (bridges the hive to Google
  A2A agents).

### Minimal skeleton

```python
# my_agent_plugin/__init__.py
from typing import Iterator, Optional
from hivemind_plugin_manager.protocols import AgentProtocol


class MyAgentProtocol(AgentProtocol):
    def natural_language_query(self, utterance: str,
                               lang: str) -> Iterator[Optional[str]]:
        # produce one or more text chunks...
        yield f"You said: {utterance}"
        # ...then terminate the stream
        yield None
```

```toml
# pyproject.toml
[project.entry-points."hivemind.agent.protocol"]
"my-agent-plugin" = "my_agent_plugin:MyAgentProtocol"
```

---

## 3. Binary data handler

A binary data handler processes binary `HiveMessage` payloads — raw audio, images,
files — on the hub (server-side wakeword/STT/VAD/TTS, camera frames, etc.).

- **Entry-point group:** `hivemind.binary.protocol`
- **Base class:** `BinaryDataHandlerProtocol` (`hivemind_plugin_manager/protocols.py`)
- **Contract:** unlike the others, the handler methods are **not abstract** — each
  has a default that logs a warning and ignores the payload. Override only the ones
  you handle. The overridable methods are:

  ```python
  def handle_microphone_input(self, bin_data: bytes, sample_rate: int,
                              sample_width: int, client): ...

  def handle_stt_transcribe_request(self, bin_data: bytes, sample_rate: int,
                                    sample_width: int, lang: str, client): ...

  def handle_stt_handle_request(self, bin_data: bytes, sample_rate: int,
                                sample_width: int, lang: str, client): ...

  def handle_numpy_image(self, bin_data: bytes, camera_id: str, client): ...

  def handle_receive_tts(self, bin_data: bytes, utterance: str, lang: str,
                         file_name: str, client): ...

  def handle_receive_file(self, bin_data: bytes, file_name: str, client): ...
  ```

  `client` is the originating `HiveMindClientConnection`. A binary handler also
  carries `self.agent_protocol` (so it can hand a transcription off to the agent)
  in addition to the shared `_SubProtocol` members.

Real implementation: `hivemind-audio-binary-protocol`
(`AudioBinaryProtocol`, server-side wakeword / STT / VAD / TTS).

### Minimal skeleton

```python
# my_binary_protocol/protocol.py
from hivemind_plugin_manager.protocols import BinaryDataHandlerProtocol


class MyBinaryProtocol(BinaryDataHandlerProtocol):
    def handle_microphone_input(self, bin_data, sample_rate,
                                sample_width, client):
        # e.g. run STT, then forward to self.agent_protocol
        ...
```

Note `hivemind-audio-binary-protocol` registers its entry point via `setup.py`
rather than `pyproject.toml`; both forms work. The `setup.py` form:

```python
# setup.py
setup(
    ...
    entry_points={
        'hivemind.binary.protocol':
            'my-binary-plugin=my_binary_protocol.protocol:MyBinaryProtocol'
    }
)
```

The equivalent `pyproject.toml` form:

```toml
[project.entry-points."hivemind.binary.protocol"]
"my-binary-plugin" = "my_binary_protocol.protocol:MyBinaryProtocol"
```

---

## 4. Database

A database plugin stores `Client` credential records (the client whitelist).

- **Entry-point group:** `hivemind.database`
- **Base class:** `AbstractDB` — or `AbstractRemoteDB` if your backend is a network
  service that needs `host` / `port` (`hivemind_plugin_manager/database.py`).
- **Contract:** implement these four abstract methods:

  ```python
  @abc.abstractmethod
  def add_item(self, client: Client) -> bool: ...

  @abc.abstractmethod
  def search_by_value(self, key: str,
                      val: Union[str, bool, int, float]) -> List[Client]: ...

  @abc.abstractmethod
  def __len__(self) -> int: ...

  @abc.abstractmethod
  def __iter__(self) -> Iterable['Client']: ...
  ```

  `add_item` persists a `Client` and returns success. `search_by_value` returns all
  `Client` rows whose `key` equals `val`. `__len__` returns the record count and
  `__iter__` iterates all `Client` rows. The base class derives `delete_item`,
  `update_item`, `replace_item`, and `get_client_by_id` from these, so you do not
  override them unless you have a faster path. `AbstractRemoteDB` re-declares the
  same four abstract methods and adds `host` (default `"127.0.0.1"`) and `port`
  fields. The factory passes `host` / `port` only to `AbstractRemoteDB` subclasses.

Real implementation: `hivemind-sqlite-database` (`SQLiteDB`, subclasses
`AbstractDB`). Others: `hivemind-json-db-plugin`, `hivemind-redis-db-plugin`.

### Minimal skeleton

```python
# my_database/__init__.py
from typing import Iterable, List, Union
from hivemind_plugin_manager.database import AbstractDB, Client


class MyDB(AbstractDB):
    def add_item(self, client: Client) -> bool:
        ...
        return True

    def search_by_value(self, key: str,
                        val: Union[str, bool, int, float]) -> List[Client]:
        return [c for c in self if c[key] == val]

    def __len__(self) -> int:
        ...

    def __iter__(self) -> Iterable['Client']:
        ...
```

```toml
# pyproject.toml
[project.entry-points."hivemind.database"]
"my-db-plugin" = "my_database:MyDB"
```

---

## 5. Policy

A policy plugin is HiveMind's admission-control point: it sees every Mycroft
`Message` (and every binary payload) about to be forwarded to the agent bus, and
can allow, deny, or mutate it.

- **Entry-point group:** `hivemind.policy`
- **Base class:** `PolicyPlugin` (`hivemind_plugin_manager/policy.py`)
- **Contract:** override one or more of these hooks (all synchronous; none is
  abstract — the defaults allow everything):

  ```python
  def review(self, message, client) -> Verdict: ...
  def review_binary(self, payload: bytes, client) -> Verdict: ...
  def observe(self, message, client) -> None: ...
  ```

  - `review` — inspect a Mycroft `Message` before `bus.emit()`; return a `Verdict`.
  - `review_binary` — same for a binary payload; default returns `Verdict.allow()`.
  - `observe` — called after a message was successfully emitted; for counters /
    audit logs / telemetry. Must not raise.

  A `Verdict` (`hivemind_plugin_manager/policy.py`) is constructed via the two
  classmethods:

  - `Verdict.allow(*mutations)` — allow, optionally carrying `Mutation` objects to
    be applied before the message proceeds.
  - `Verdict.deny(code, reason="", **data)` — deny, where `code` is a stable
    machine-readable string (use the `DenyCodes` enum, e.g.
    `DenyCodes.ACL_DISALLOWED_TYPE`, or any string).

  A `Mutation` is an abstract base; concrete mutation kinds are agent-specific and
  ship with the consuming agent plugin (the OVOS plugin ships `AddBlacklistedSkill`,
  `RewriteUtterance`, etc.). The chain runner in `hivemind-core` treats any unhandled
  exception in `review` / `review_binary` as a fail-closed
  `Verdict.deny("policy_error", ...)`; `observe` exceptions are logged and swallowed.

  Policies run in the order set by the operator's `policy.chain` config, but
  `MessageTypeACLPolicy` is **always force-prepended** to the chain and cannot be
  removed by configuration — it enforces each client's `allowed_types` whitelist
  before any configured plugin runs.

Real implementation: `hivemind-ovos-agent-plugin` ships a policy alongside its agent,
registering `OVOSAgentPolicy` under `hivemind.policy` (see its `pyproject.toml`).

### Minimal skeleton

```python
# my_policy/__init__.py
from hivemind_plugin_manager.policy import PolicyPlugin, Verdict, DenyCodes


class MyPolicy(PolicyPlugin):
    def review(self, message, client) -> Verdict:
        if message.msg_type == "some.forbidden.type":
            return Verdict.deny(DenyCodes.ACL_DISALLOWED_TYPE,
                                reason="not allowed here")
        return Verdict.allow()
```

```toml
# pyproject.toml — note: a single package may register entry points in
# multiple groups (the OVOS plugin registers both an agent and a policy)
[project.entry-points."hivemind.policy"]
"my-policy" = "my_policy:MyPolicy"
```

---

## Next

- [Plugin Architecture](../concepts/plugins.md) — how operators select and configure
  installed plugins in `server.json`.
