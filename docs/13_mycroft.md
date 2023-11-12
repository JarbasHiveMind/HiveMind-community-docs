# OpenVoiceOS Messages

The OpenVoiceOS messagebus is considered an internal and private websocket for [minds](https://github.com/JarbasHiveMind/HiveMind-core/wiki/Terminology), clients do not connect directly to it.

A mind will inject its own context about the originating clients,  only responses to the client message will be forwarded, this provides client isolation. 

A mind will filter incoming and outgoing messages per client, the permissions model of the hivemind is extensive (or will be, it's WIP!) eg, it might refuse utterances based on the intent, ovos-core will never even see them

This info applies to ovos-core, Hivemind depends on it and originated the PR, but this functionality is not part of the hivemind itself

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

original PR https://github.com/MycroftAI/mycroft-core/pull/2461

## Sources

ovos-core injects the context when it emits an utterance, this can be either typed in the `ovos-cli-client` or spoken via OVOS STT service

STT will identify itself as `audio`

ovos-cli-client will identify itself as `debug_cli`


## Destinations

Output capable services are the cli and the TTS

The command line is a debug tool, it will ignore the `destination`

TTS checks the message context if it's the intended target for the message and will only speak in the following conditions:

- Explicitly targeted i.e. the `destination` is `"audio"`
- `destination` is set to `None`
- `destination` is missing completely

The idea is that for example when the android app is used to access OpenVoiceOS the device at home shouldn't start to speak.

TTS will be executed when `"audio"` or `"debug_cli"` are the `destination`

A missing `destination` or if the `destination` is set to `None` is interpreted as a multicast and should trigger all output capable processes (be it the ovos-audio process, a web-interface, the KDE plasmoid or maybe the android app)

## Skills

- intent service will `.reply` to the original utterance message
- all skill/intent service messages are `.forward` (from previous intent service `.reply`)
- skills sending their own messages might not respect this :warning: 
- `converse`/`get_response` support is limited, the context is lost :warning: [ovos-core is working on it](https://github.com/OpenVoiceOS/ovos-bus-client/blob/dev/ovos_bus_client/session.py)
- in the context of the Hivemind skills might keep a shared state between clients, eg. a client may enable [parrot mode](https://github.com/JarbasSkills/skill-parrot) for everyone  :warning: :skull: 
- scheduled events support is limited, the context is lost :warning:
