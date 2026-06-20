# DeltaChat Bridge

[HiveMind-deltachat-bridge](https://github.com/JarbasHiveMind/HiveMind-deltachat-bridge) connects a [DeltaChat](https://delta.chat/) email-based chat account to a HiveMind hub. Messages sent to the bot email address are forwarded to the hub as utterances; responses come back as email replies.

DeltaChat is an end-to-end encrypted messenger that uses email as its transport. No separate server is required — any email account works.

## Install

```bash
pip install HiveMind-deltachat-bridge
```

!!! warning "Console entry point is broken upstream"
    The packaged `hm-deltachat-bridge` console command does not work: `setup.py`
    points it at a `deltachat_bridge` module while the package is actually named
    `hm_deltachat_bridge`. Until this is fixed upstream, run the bridge from a
    checkout (`python -m hm_deltachat_bridge ...`).

## Usage

See the [repository documentation](https://github.com/JarbasHiveMind/HiveMind-deltachat-bridge) for setup instructions. You will need:

- A DeltaChat-compatible email account for the bot
- HiveMind credentials (access key and password)

## How it works

The bridge runs as a HiveMind client. Incoming DeltaChat messages are forwarded to the hub as `recognizer_loop:utterance` OVOS messages. The hub's `speak` responses are sent back as DeltaChat messages to the originating conversation.
