# Persona Server

Sometimes you don't want timers and weather skills — you just want to *talk to
something*. A persona server is the simplest hive there is: swap the skills brain for an
LLM with a personality, and every satellite becomes a way to chat with it. No
`ovos-core`, no messagebus, no skill stack to stand up — just hivemind-core, a persona,
and whatever language model you point it at (OpenAI, a local Ollama, anything
OpenAI-shaped). Answers stream back sentence by sentence, so the satellite can start
speaking before the model has finished thinking.

!!! abstract "In a nutshell"
    - Swaps the `agent_protocol` for `hivemind-persona-agent-plugin`, so `hivemind-core` answers from an LLM / solver chain instead of skills.
    - A persona is just a JSON document naming the solver plugins (e.g. `ovos-solver-openai-plugin`) and their config.
    - Text satellites and bridges work against it; audio-binary-protocol satellites don't apply — a persona serves words, not server-side audio.

Satellites built for `hivemind-core` (text-based ones like HiveMind-cli, voice-sat, or
bridges) work against a persona server. Satellites that depend on the audio binary
protocol (`hivemind-audio-binary-protocol`) do **not** apply here — a persona server
exposes a text/query agent, not server-side audio processing.

---

## Install

```bash
pip install hivemind-core hivemind-persona-agent-plugin ovos-persona
```

You also need at least one solver plugin for the persona to call. For an
OpenAI-compatible backend (OpenAI, LocalAI, Ollama, vLLM, …):

```bash
pip install ovos-openai-plugin
```

This provides the `ovos-solver-openai-plugin` solver.

---

## Configure the persona

A persona is just a small JSON file — a name and a list of "solvers," where a solver is
whatever actually produces the answer (usually an LLM). Here's the whole thing for an
OpenAI-compatible backend; save it somewhere like `~/.config/ovos_persona/persona.json`:

```json
{
  "name": "MyAssistant",
  "solvers": [
    {
      "module": "ovos-solver-openai-plugin",
      "ovos-solver-openai-plugin": {
        "api_url": "http://localhost:8000/v1",
        "key": "your-api-key",
        "model": "local-model",
        "system_prompt": "You are a helpful, concise assistant."
      }
    }
  ]
}
```

The OpenAI solver keys are `api_url`, `key`, `model`, and `system_prompt` — there is no
`persona` key inside the solver block; the persona is the surrounding document.

---

## Configure hivemind-core

Point the `agent_protocol` block in `~/.config/hivemind-core/server.json` at the
persona plugin, and give it the persona to run:

```json
{
  "agent_protocol": {
    "module": "hivemind-persona-agent-plugin",
    "hivemind-persona-agent-plugin": {
      "persona": "~/.config/ovos_persona/persona.json"
    }
  }
}
```

`persona` may be either a path to a persona JSON file (`~` is expanded) or an inline
persona config dict. The file is read once at startup; restart `hivemind-core` to pick
up edits.

---

## Start the server

```bash
hivemind-core listen
```

Satellites connecting to this server now receive answers streamed from the persona,
sentence by sentence.

---

## Managing clients

Client management is the normal `hivemind-core` CLI — there is no separate persona
command. Node IDs are **positional**:

```bash
# Add a new client
hivemind-core add-client --name "my-client"

# List clients
hivemind-core list-clients

# Remove client with node ID 2
hivemind-core delete-client 2
```

See [OVOS Skills Server](ovos-server.md#managing-clients) for the full client workflow.

---

## Solver plugins

Any `ovos-solver-*` plugin can back the persona. Common ones:

- `ovos-solver-openai-plugin` — OpenAI or any OpenAI-compatible API (LocalAI, Ollama,
  vLLM, …), installed via `pip install ovos-openai-plugin`.
- `ovos-solver-failure-plugin` — always returns a fallback response (useful for
  testing).

---

## Example personas

The persona file is where you decide *who* your assistant talks to. Swapping from a cloud
model to one running on your own machine is a one-line change — the `api_url`. Two ends of
that spectrum:

**Local LLM via LocalAI / Ollama** (same OpenAI solver, pointed at a local `api_url`):

```json
{
  "name": "LocalAssistant",
  "solvers": [
    {
      "module": "ovos-solver-openai-plugin",
      "ovos-solver-openai-plugin": {
        "api_url": "http://127.0.0.1:11434/v1",
        "key": "ollama",
        "model": "llama3",
        "system_prompt": "You are a concise and accurate assistant."
      }
    }
  ]
}
```

**OpenAI / ChatGPT:**

```json
{
  "name": "ChatGPT",
  "solvers": [
    {
      "module": "ovos-solver-openai-plugin",
      "ovos-solver-openai-plugin": {
        "api_url": "https://api.openai.com/v1",
        "key": "<your_openai_key>",
        "model": "gpt-4o-mini",
        "system_prompt": "You are helpful, creative, clever, and very friendly."
      }
    }
  ]
}
```

---

## Source

Validated against the HiveMind source:

- [`hivemind_persona_agent_plugin/__init__.py`](https://github.com/JarbasHiveMind/hivemind-persona-agent-plugin/blob/HEAD/hivemind_persona_agent_plugin/__init__.py) — the `persona` config key (path or inline dict, read once at startup) and sentence-by-sentence streaming
- [`ovos-persona`](https://github.com/OpenVoiceOS/ovos-persona) — the persona document format and the solver chain it runs
