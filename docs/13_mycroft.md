# OpenVoiceOS Messages

The OpenVoiceOS messagebus is considered an internal and private websocket for [minds](https://github.com/JarbasHiveMind/HiveMind-core/wiki/Terminology), clients do not connect directly to it.

A mind will inject its own context about the originating clients,  only responses to the client message will be forwarded, this provides client isolation. 

A mind will filter incoming and outgoing messages per client, the permissions model of the hivemind is extensive e.g. it might refuse utterances based on the intent

This info applies to `ovos-core`, Hivemind depends on this functionality but it is not part of the hivemind itself. HiveMind responsibility is only to deliver the BUS messages

From the POV of the Hivemind you can [replace ovos-core with anything](https://github.com/JarbasHiveMind/Fakecroft-DDG) as long as you respect the mechanisms below

  * [Message](#message)
  * [Targeting Theory](#targeting-theory)
  * [Sources](#sources)
  * [Destinations](#destinations)
  * [Skills](#skills)

## Message

A OpenVoiceOS message consists of a json payload, it contains a `type` , some `data` and a `context`.

The `context` is considered to be metadata and might be changed at any time in transit, the `context` can contain anything depending on where the message came from, and often is completely empty. 

You can think of the message `context` as a sort of session data for a individual interaction, in general messages down the chain keep the `context` from the original message, most listeners (eg, skills) will only care about `type` and `data`. 

## Targeting Theory

ovos-core uses the message `context` to add metadata about the messages themselves, where do they come from and what are they intended for.

the `Message` object provides the following methods:

- `message.forward` method, keeps previous context.
	- message continues going to same `destination`
- `message.reply` method swaps `"source"` with `"destination"`
	- message goes back to `source`

The context `destination` parameter in the original message can be set to a list with any number of intended targets:

```python
bus.emit(Message('recognizer_loop:utterance', data, 
				 context={'destination': ['audio', 'kde'],
						  'source': "remote_service"))
```

## Sources

ovos-core injects the context when it emits an utterance, this can be either typed or spoken via OVOS STT service

STT will identify itself as `audio`

mycroft.conf defines a list of `"native_sources"`, by default only `audio` is a native source

## Destinations

Output capable services are ovos-audio (TTS, music...)

TTS checks the message context if it's the intended target for the message and will only speak in the following conditions:

- Explicitly targeted i.e. the `destination` is native_source (default: `"audio")`

- `destination` is set to `None`

- `destination` is missing completely

The idea is that for example when the android app is used to access OpenVoiceOS the device at home shouldn't start to speak.

TTS will be executed when a native_source (eg, `audio`) is the `destination`

A missing `destination` or if the `destination` is set to `None` is interpreted as a multicast and should trigger all output capable processes (be it the ovos-audio process, a web-interface, the KDE plasmoid or maybe the android app)

## OVOS-Core

ovos-core is responsible for managing the routing context, skills do not usually need to worry about any of this

- intent service will `.reply` to the original utterance message

- all skill/intent service messages are `.forward` (from previous intent service `.reply`)

### Skills 

OpenVoiceOS skills can do anything, if you are developing/installing a mission critical skill carefully evaluate what it does and evaluate if it is hivemind friendly

If a skill emits it's own bus messages it needs to keep `message.context` around

**Common issues**:

- skills sending their own messages might not keep message.context or wrongly `.reply` to it 

- in the context of the Hivemind skills might not be [Session](https://github.com/OpenVoiceOS/ovos-bus-client/blob/dev/ovos_bus_client/session.py) aware and keep a shared state between clients, eg. a client may enable a voice game for everyone 