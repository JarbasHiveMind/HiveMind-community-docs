# Persona (LLM) Hub

> ℹ️ The standalone `hivemind-persona` package is **superseded**. Its last release
> (0.0.2, December 2024) pins `hivemind-core>=1.0.0,<2.0.0`, while the current core is
> 4.x — it cannot be installed alongside a current hub. Persona is now an **agent
> plugin** loaded by `hivemind-core`.

## Install

```bash
pip install hivemind-core hivemind-persona-agent-plugin
```

## Configure

Point `agent_protocol` at it in `~/.config/hivemind-core/server.json`:

```json
{
  "agent_protocol": {
    "module": "hivemind-persona-agent-plugin",
    "hivemind-persona-agent-plugin": {"persona": "/path/to/chatgpt.json"}
  }
}
```

Then run `hivemind-core listen`. Satellites that stream binary audio still need a
binary protocol plugin on the hub — see [Sound Server](./06_sound_server.md).

---


this is a hivemind Master node, but it is running [ovos-persona](https://github.com/OpenVoiceOS/ovos-persona) instead of connecting to ovos-core

you can use this to expose chatbots and LLMs via hivemind, satellites made for `hivemind-core` should be compatible

> ⚠️ Satellites made specifically for `hivemind-listener` (Sound server) will not work with `hivemind-persona`!

![img_13.png](img_13.png)


## Install

```bash
pip install hivemind-core hivemind-persona-agent-plugin
```

## ChatGPT

Install the [OpenAI solver](https://github.com/OpenVoiceOS/ovos-solver-plugin-openai-persona/)

create a chatgpt.json
```json
{
"name": "ChatGPT",
"solvers": [
    "ovos-solver-openai-persona-plugin"
],
"ovos-solver-openai-persona-plugin": {
    "api_url": "<your_local_LocalAI_server_url>",
    "key": "<your_OpenAI_key>",
    "persona": "helpful, creative, clever, and very friendly."
}
}
```

launch hivemind-persona with the created file

`# (superseded — see the configuration below)
# hivemind-persona --persona chatgpt.json`
