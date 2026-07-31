# OpenVoiceOS Messages

The OpenVoiceOS messagebus is an internal, private websocket for [minds](02_terminology.md). Clients do not connect to it directly.

A mind injects its own context about the originating client, and forwards only responses to that client's message. This gives client isolation.

A mind filters incoming and outgoing messages per client. The HiveMind permissions model is extensive; for example, it can refuse an utterance based on the intent it maps to.

This applies to `ovos-core`. HiveMind depends on this behavior, but it is not part of HiveMind itself. HiveMind is only responsible for delivering BUS messages.

From HiveMind's point of view, you can [replace ovos-core with anything](https://github.com/JarbasHiveMind/Fakecroft-DDG), as long as it respects the mechanisms below.

  * [Message](#message)
  * [Targeting Theory](#targeting-theory)
  * [Sources](#sources)
  * [Destinations](#destinations)
  * [Skills](#skills)

## Message

An OpenVoiceOS message is a JSON payload. It contains a `type`, some `data`, and a `context`.

The `context` is metadata and can change at any point in transit. It can contain anything, depending on where the message came from, and is often empty. Think of the message `context` as session data for an individual interaction: messages down the chain generally keep the `context` from the original message, and most listeners, such as skills, only care about `type` and `data`.

## Targeting theory

ovos-core uses the message `context` to add metadata about the messages themselves, including where they come from and what they are meant for.

The `Message` object provides these methods:

- `message.forward` keeps the previous context; the message continues to the same `destination`.
- `message.reply` swaps `"source"` with `"destination"`; the message goes back to `source`.

You can set the `destination` parameter in the original message's context to a list with any number of intended targets:

```python
bus.emit(Message('recognizer_loop:utterance', data, 
				 context={'destination': ['audio', 'kde'],
						  'source': "remote_service"))
```

## Sources

ovos-core injects the context when it emits an utterance, whether typed or spoken through the OVOS STT service.

STT identifies itself as `audio`.

`mycroft.conf` defines a list of `"native_sources"`. By default, only `audio` is a native source.

## Destinations

Output-capable services include ovos-audio (TTS, music, and so on).

TTS checks the message context to see if it is the intended target, and speaks only when:

- the `destination` is explicitly a native source, by default `"audio"`.
- the `destination` is set to `None`.
- the `destination` is missing.

For example, when the Android app accesses OpenVoiceOS, the device at home should not start speaking.

TTS runs when a native source, such as `audio`, is the `destination`.

A missing `destination`, or a `destination` set to `None`, is treated as a multicast and should trigger all output-capable processes: the ovos-audio process, a web interface, the KDE plasmoid, or the Android app.

## OVOS-Core

ovos-core manages the routing context. Skills usually do not need to worry about this.

- The intent service `.reply`s to the original utterance message.
- All skill and intent service messages `.forward` from the previous intent service `.reply`.

### Skills

OpenVoiceOS skills can do anything. If you develop or install a mission-critical skill, check carefully what it does and whether it works well with HiveMind.

If a skill emits its own bus messages, it needs to keep `message.context` intact.

Common issues:

- A skill sending its own messages might drop `message.context`, or `.reply` to it incorrectly.
- In a HiveMind context, a skill might not be [Session](https://github.com/OpenVoiceOS/ovos-bus-client/blob/dev/ovos_bus_client/session.py)-aware, and might keep shared state between clients. For example, a client could enable a voice game for everyone.

---
[Home](index.md)
