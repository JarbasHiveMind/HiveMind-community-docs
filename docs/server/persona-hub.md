# Persona Hub

A "persona hub" is a regular `hivemind-core` server whose **agent protocol** is the
[ovos-persona](https://github.com/OpenVoiceOS/ovos-persona) plugin. Instead of routing
utterances into a full OVOS skills stack, the hub answers natural-language queries
directly from an LLM / chatbot / solver-chain persona — no `ovos-core` required.

Satellites built for `hivemind-core` (text-based ones like HiveMind-cli, voice-sat, or
bridges) work against a persona hub. Satellites that depend on the audio binary
protocol (`hivemind-audio-binary-protocol`) do **not** apply here — a persona hub
exposes a text/query agent, not server-side audio processing.

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

## Configure the persona

An `ovos-persona` persona is a JSON document listing the solver(s) and their config.
Save it somewhere (for example `~/.config/ovos_persona/persona.json`):

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

## Start the server

```bash
hivemind-core listen
```

Satellites connecting to this hub now receive answers streamed from the persona,
sentence by sentence.

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

See [OVOS Skills Hub](ovos-hub.md#managing-clients) for the full client workflow.

## Solver plugins

Any `ovos-solver-*` plugin can back the persona. Common ones:

- `ovos-solver-openai-plugin` — OpenAI or any OpenAI-compatible API (LocalAI, Ollama,
  vLLM, …), installed via `pip install ovos-openai-plugin`.
- `ovos-solver-failure-plugin` — always returns a fallback response (useful for
  testing).

## Example personas

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
