# Persona Hub

`hivemind-persona` is a hub that runs [ovos-persona](https://github.com/OpenVoiceOS/ovos-persona) as its AI back-end instead of a full OVOS instance. Use it to expose LLMs, chatbots, and solver chains to HiveMind satellites — without installing `ovos-core`.

Satellites built for `hivemind-core` (text-based ones like HiveMind-cli, voice-sat, or bridges) work with `hivemind-persona`. Satellites that depend on the audio binary protocol (`hivemind-audio-binary-protocol`) do **not** work with `hivemind-persona`.

## Install

```bash
pip install hivemind-persona
```

## Usage

Point it at a persona definition file:

```bash
hivemind-persona --persona /path/to/persona.json
```

A persona file is a JSON document that lists solver plugins and their configuration:

```json
{
  "name": "MyAssistant",
  "solvers": [
    "ovos-solver-openai-persona-plugin"
  ],
  "ovos-solver-openai-persona-plugin": {
    "api_url": "http://localhost:8080/v1",
    "key": "your-api-key",
    "persona": "helpful, creative, and concise."
  }
}
```

Client management works identically to `hivemind-core`:

```bash
hivemind-persona add-client --name "my-client"
hivemind-persona list-clients
```

## Solver plugins

Any `ovos-solver-*` plugin works as the back-end. Examples:

- `ovos-solver-openai-persona-plugin` — OpenAI or OpenAI-compatible API (LocalAI, Ollama, etc.)
- `ovos-solver-failure-plugin` — always returns a fallback response (useful for testing)

Install solvers as separate pip packages:

```bash
pip install ovos-solver-openai-persona-plugin
```

## Persona file examples

**Local LLM via LocalAI / Ollama:**

```json
{
  "name": "LocalAssistant",
  "solvers": ["ovos-solver-openai-persona-plugin"],
  "ovos-solver-openai-persona-plugin": {
    "api_url": "http://127.0.0.1:11434/v1",
    "key": "ollama",
    "persona": "You are a concise and accurate assistant."
  }
}
```

**OpenAI ChatGPT:**

```json
{
  "name": "ChatGPT",
  "solvers": ["ovos-solver-openai-persona-plugin"],
  "ovos-solver-openai-persona-plugin": {
    "api_url": "https://api.openai.com",
    "key": "<your_openai_key>",
    "persona": "helpful, creative, clever, and very friendly."
  }
}
```
