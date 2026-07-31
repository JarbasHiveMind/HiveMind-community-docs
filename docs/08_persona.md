# Persona

This is a HiveMind master node that runs [ovos-persona](https://github.com/OpenVoiceOS/ovos-persona) instead of connecting to ovos-core.

Use it to expose chatbots and LLMs over HiveMind. Satellites made for `hivemind-core` should be compatible.

> Satellites made specifically for `hivemind-listener` (the sound server) do not work with `hivemind-persona`.

![img_13.png](img_13.png)

## Install

```bash
pip install hivemind-persona
```

## ChatGPT

Install the [OpenAI solver](https://github.com/OpenVoiceOS/ovos-solver-plugin-openai-persona/).

Create a `chatgpt.json`:
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

Launch hivemind-persona with the file:

`hivemind-persona --persona chatgpt.json`

---
[Home](index.md)
