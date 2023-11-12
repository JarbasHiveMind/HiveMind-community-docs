# Message types

The HiveMind can be seen as a global OpenVoiceOS bus shared across devices, but not everyone gets every message!

Messages consist of 2 parts, a `message_type` and a `payload`.

The `message_type` defines how the message is routed. Each node may ignore specific `message_type`s (either globally or per client) depending on its configuration.

The payload can be another `HiveMessage`, a OpenVoiceOS `Message`, or even binary, depending on the `message_type`.

- [Message types](#message-types)
  * [Payload Messages](#payload-messages)
      - [Bus](#bus)
      - [Shared Bus](#shared-bus)
  * [Transport Messages](#transport-messages)
      - [Broadcast](#broadcast)
      - [Escalate](#escalate)
      - [Propagate](#propagate)

## Payload Messages

the payload of these messages is a OVOS `Message` object

#### Bus

`BUS` is single hop and flows from `slave -> master` and `master -> slave`

when master receives a `BUS` message, it will check it's global `whitelist/blacklist` and slave permissions, if the slave is authorized to inject that message in the `ovos-core` messagebus then it will be injected. Any messages from master's `ovos-core` that are a direct response to the injected message will be forwarded back to the slave

Permissions can be defined as a combination of `msg_type`, `intent_type`, `skill_id`, `accessKey` and `ip_address` rules

you can find the dedicated page on how these messages are handled by OVOS [here](https://github.com/JarbasHiveMind/HiveMind-core/wiki/Mycroft-Messages)

![](https://raw.githubusercontent.com/JarbasHiveMind/HiveMind-core/dev/resources/bus.gif)

The most common example will be injecting a user utterance and receiving the responses, here is the output from `master -> slave` to the injected utterance "tell me a joke"

```
{'msg_type': 'bus', 'payload': {'type': 'skill.converse.request', 'data': {'skill_id': 'mycroft-joke.mycroftai', 'utterances': ['tell me a joke'], 'lang': 'en-us'}, 'context': {'source': 'HiveMind', 'destination': 'tcp4:127.0.0.1:52772', 'platform': 'HiveMessageBusClientV0.0.1', 'peer': 'tcp4:127.0.0.1:52772', 'client_name': 'HiveMindV0.7'}}, 'route': [], 'node': None, 'source_peer': 'tcp4:0.0.0.0:5678'}
{'msg_type': 'bus', 'payload': {'type': 'mycroft-joke.mycroftai:JokingIntent', 'data': {'intent_type': 'mycroft-joke.mycroftai:JokingIntent', 'mycroft_joke_mycroftaiJoke': 'joke', 'target': None, 'confidence': 0.3333333333333333, '__tags__': [{'match': 'joke', 'key': 'joke', 'start_token': 3, 'entities': [{'key': 'joke', 'match': 'joke', 'data': [['joke', 'mycroft_joke_mycroftaiJoke']], 'confidence': 1.0}], 'end_token': 3, 'from_context': False}], 'utterance': 'tell me a joke'}, 'context': {'source': 'HiveMind', 'destination': 'tcp4:127.0.0.1:52772', 'platform': 'HiveMessageBusClientV0.0.1', 'peer': 'tcp4:127.0.0.1:52772', 'client_name': 'HiveMindV0.7'}}, 'route': [], 'node': None, 'source_peer': 'tcp4:0.0.0.0:5678'}
{'msg_type': 'bus', 'payload': {'type': 'mycroft.skill.handler.start', 'data': {'name': 'JokingSkill.handle_general_joke'}, 'context': {'source': 'HiveMind', 'destination': 'tcp4:127.0.0.1:52772', 'platform': 'HiveMessageBusClientV0.0.1', 'peer': 'tcp4:127.0.0.1:52772', 'client_name': 'HiveMindV0.7'}}, 'route': [], 'node': None, 'source_peer': 'tcp4:0.0.0.0:5678'}
{'msg_type': 'bus', 'payload': {'type': 'speak', 'data': {'utterance': "When Chuck Norris breaks the build, you can't fix it, because there is not a single line of code left.", 'expect_response': False, 'meta': {'skill': 'JokingSkill'}}, 'context': {'source': 'HiveMind', 'destination': 'tcp4:127.0.0.1:52772', 'platform': 'HiveMessageBusClientV0.0.1', 'peer': 'tcp4:127.0.0.1:52772', 'client_name': 'HiveMindV0.7'}}, 'route': [], 'node': None, 'source_peer': 'tcp4:0.0.0.0:5678'}
{'msg_type': 'bus', 'payload': {'type': 'mycroft.skill.handler.complete', 'data': {'name': 'JokingSkill.handle_general_joke'}, 'context': {'source': 'HiveMind', 'destination': 'tcp4:127.0.0.1:52772', 'platform': 'HiveMessageBusClientV0.0.1', 'peer': 'tcp4:127.0.0.1:52772', 'client_name': 'HiveMindV0.7'}}, 'route': [], 'node': None, 'source_peer': 'tcp4:0.0.0.0:5678'}
```

#### Shared Bus

`SHARED_BUS` is single hop and flows from `slave -> master`

the master passively monitors everything going on the slave `ovos-core` instance's `messagebus`

`SHARED_BUS` needs to be explicitly enabled in the slave device configuration, by default the `ovos-core` bus is not shared

in the case of terminals, messages of the type `BUS` are injected in their master, these will also be forwarded as `SHARED_BUS` to the next master

![](https://raw.githubusercontent.com/JarbasHiveMind/HiveMind-core/dev/resources/shared_bus.gif)

## Transport Messages

the payload of these messages is another `HiveMessage` object

#### Broadcast

`BROADCAST` is multi-hop and flows from `master -> slave `

Send message to all of the recipient's slaves ("send the message down")

![](https://raw.githubusercontent.com/JarbasHiveMind/HiveMind-core/dev/resources/broadcast.gif)

#### Escalate

`ESCALATE` is multi-hop and flows from `slave -> master`

Send message up the authority chain, never to a slave ("send the message up")

![](https://raw.githubusercontent.com/JarbasHiveMind/HiveMind-core/dev/resources/escalate.gif)


#### Propagate

`PROPAGATE` is multi-hop and flows both from `master -> slave` and `slave -> master`

Send message to all your direct connections ("send the message everywhere")

![](https://raw.githubusercontent.com/JarbasHiveMind/HiveMind-core/dev/resources/propagate.gif)

