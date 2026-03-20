
# HiveMind-community-docs — FAQ

## Where is the official documentation hosted?
It is hosted on GitHub Pages at https://jarbashivemind.github.io/HiveMind-community-docs/.

## How can I contribute to the documentation?
You can contribute by submitting pull requests to this repository. All documentation is written in Markdown and managed by MkDocs.

## How do I build the documentation locally?
You need to install `mkdocs` and the `mkdocs-material` theme (though this repo currently uses `readthedocs` theme). Run `mkdocs serve` from the root of the repository to preview changes.

## Are there diagrams of the protocol?
Yes, there are several diagrams (e.g., `HANDSHAKE_V1.png`) and detailed protocol documentation in the `docs/` folder.

## Do I need OVOS to use HiveMind?
No. HiveMind supports two hub types: `hivemind-core` (OVOS-based) and `hivemind-persona` (standalone LLM/chatbot). See [Quick Start](docs/01_quickstart.md) and [Persona Server](docs/08_persona.md).

## What's the difference between hivemind-core and hivemind-persona?
`hivemind-core` runs OVOS for skills-based AI with plugins. `hivemind-persona` runs LLMs or chatbots without OVOS. Both use the same HiveMind protocol.
