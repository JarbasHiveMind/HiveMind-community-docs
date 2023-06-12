# What is the HiveMind ?

The hivemind can be used to build many things, theres some jargon and lots of stuff potentially going on, let's explore what it can do, and by the end hopefully you will understand what it is

- [the hivemind is a OVOS add-on](#the-hivemind-is-a-ovos-add-on)
- [the hivemind connects devices](#the-hivemind-connects-devices)
- [the hivemind decentralizes ovos-core](#the-hivemind-decentralizes-ovos-core)
- [the hivemind encrypts the messagebus](#the-hivemind-encrypts-the-messagebus)
- [the hivemind authenticates the messagebus](#the-hivemind-authenticates-the-messagebus)
- [the hivemind safely exposes OVOS to the web](#the-hivemind-safely-exposes-ovos-to-the-web)
- [the hivemind isolates the messagebus](#the-hivemind-isolates-the-messagebus)
- [the hivemind is a protocol that transparently integrates with OVOS](#the-hivemind-is-a-protocol-that-transparently-integrates-with-ovos)
- [the hivemind provides skill isolation](#the-hivemind-provides-skill-isolation)
- [the hivemind can be used to integrate OVOS with any platform](#the-hivemind-can-be-used-to-integrate-ovos-with-any-platform)
- [the hivemind is a set of permissively licensed libraries](#the-hivemind-is-a-set-of-permissively-licensed-libraries)

_____

#### the hivemind is a OVOS add-on

Since the hivemind is focused on OVOS, let's start there

- step 1: install [ovos-core](https://github.com/MycroftAI/ovos-core) / own a [mycroft device](https://www.kickstarter.com/projects/aiforeveryone/mycroft-mark-ii-the-open-voice-assistant) / install a [project shipping mycroft](https://github.com/ChanceNCounter/awesome-mycroft-community#distributions-images-projects-shipping-mycroft)
- step 2: run [hivemind-core](https://github.com/JarbasHiveMind/HiveMind-core/blob/dev/examples/mycroft_master.py) in that device
- step 3: :tada:

This is the gist of it, you have a voice assistant and it can do things! It has a brain of sorts

I will refer to this as the *OVOS node* in the other examples

Ok, we installed it but still don't know anything about the hivemind, what does it do?

______________
#### the hivemind connects devices

If you are like me and live in the terminal, wouldn't it be nice to just ask a quick question?

- step 1: install the [cli terminal](https://github.com/OpenJarbas/HiveMind-cli) and [register it](https://github.com/JarbasHiveMind/HiveMind-core/blob/dev/examples/add_keys.py) (**see below**) in the OVOS node
- step 2: if auto discovery is enabled in the OVOS node it will auto connect, alternatively you can specify an ip address directly
- step 3 :tada:

isnt this the same thing as the mycroft debug cli but with extra work? 

No! the terminal is running in your laptop and OVOS is in a different room of the house
______________
#### the hivemind decentralizes ovos-core

With the hive mind we can create all kinds of thin clients that don't actually run OVOS

- step 1: install the [voice satellite](https://github.com/JarbasHiveMind/HiveMind-voice-sat) or the [push to talk node](https://github.com/JarbasHiveMind/HiveMind-PTT), eg, in a [raspberry pi 0](https://github.com/JarbasHiveMind/HiveMind-PTT/blob/respeaker2mic/respeaker.mp4)
     - NOTE: PTT is in the process of being merged into the voice satellite
- step 2: if auto discovery is enabled in the OVOS node it will auto connect, alternatively you can specify an ip address directly
- step 3 :tada:

Now you can access the *OVOS node* anywhere in your house, get a microphone in each room!

______________

#### the hivemind encrypts the messagebus

I have seen many people host OVOS in the cloud, but the messagebus in unencrypted and doesn't even [support ssl](https://github.com/MycroftAI/ovos-core/pull/1148), a common solution is to setup nginx and/or [smartgic docker image](https://github.com/OpenVoiceOS/ovos-docker)

hivemind supports ssl connections, and will even auto generate self signed certificates, however self signed certificates are [unsafe](https://en.wikipedia.org/wiki/Man-in-the-middle_attack)

managing ssl certificates at home can get messy really fast, as part of the [registration process](https://github.com/JarbasHiveMind/HiveMind-core/blob/dev/examples/add_keys.py) you can set an optional  `crypto_key` (must be 16 chars in lenght) and the hivemind will [AES encrypt](https://github.com/JarbasHiveMind/HiveMind-core/blob/dev/jarbas_hive_mind/utils/__init__.py#L137) every payload. This makes ssl essentially optional

- step 1: install the [local webchat](https://github.com/JarbasHiveMind/HiveMind-webchat)
- step 2: open `http://{ip_address}:9090` in your browser
- step 3: :tada:

NOTE: currently this `crypto_key` is assumed to be pre shared out of band to ensure a human in the loop, there will be [other options](https://github.com/JarbasHiveMind/poorman_handshake) coming soon

______________

#### the hivemind authenticates the messagebus

the message bus is [a privacy nightmare and should be closed from the outside world](https://github.com/Nhoya/MycroftAI-RCE), anyone can connect to it and capture every single thing going on in "OVOS's nervous system" or inject new commands

- step 1: install [ovos_utils](https://github.com/OpenVoiceOS/ovos_utils) on your *OVOS node* (or get it's ip address)
- step 2: run the [ovos_utils metrics](https://github.com/OpenVoiceOS/ovos_utils/blob/dev/examples/count_utterances.py) example with that ip address
- step 3: :question: (spyware)

By default the hivemind requires authentication, think of this like the token you would be given for any http api, unregistered clients can not connect to the hivemind

This is the registration step mentioned earlier

```python
from jarbas_hive_mind.database import ClientDatabase

name = "JarbasCliTerminal"
access_key = "something that uniquely identifies the client"
mail = "jarbasaai@mailfence.com"

with ClientDatabase() as db:
    db.add_client(name, mail, access_key)
```

There are [plans](https://github.com/JarbasHiveMind/HiveMind-core/issues/16) for [command](https://github.com/JarbasHiveMind/HiveMind-core/issues/17) line [utils](https://github.com/JarbasHiveMind/HiveMind-core/issues/18) for this registration process, currently you need to run python code or edit the [json_database](https://github.com/HelloChatterbox/json_database) manually

NOTE: username and email are placeholders and not currently used for anything, username can be used to provide a human readable string but it does not need to be unique and can change in every connection, in the future these may be used to better support keys shared across devices


______________

#### the hivemind safely exposes OVOS to the web

since there is a local chat node, can we get OVOS on a server and expose it to the public?

Yes! There is a catalan voice assistant built on top the hivemind, you can [test it](https://ona.assistent.cat/) and check [github](https://github.com/assistent-cat/ona)

- step 1: install the [flask chatroom template](https://github.com/JarbasHiveMind/HiveMind-flask-template)
- step 2: navigate to `http://{ip_address}:8081`
- step 3 :tada:


______________

#### the hivemind isolates the messagebus

I don't care about hackers and privacy, its all in my super safe network, Why not connect to the messagebus directly like the [android passthrough app](https://github.com/MycroftAI/Mycroft-Android) does? is the hivemind still needed?

- step 1: install the [android app](https://github.com/MycroftAI/Mycroft-Android)
- step 2: get the ip address of your *OVOS node* and set the url as `ws://{ip_address}:8181`
- step 3: :x: (disable the firewall in OVOS device)
- step 4: :question: 

When you use the android app you will notice that you get the answers from any question asked in the actual OVOS device in your phone, and that your device speaks everything you ask from the phone out loud.

this is solved by including proper metadata in the bus message.context, you can read more about how ovos-core routes messages internally in [the wiki](https://github.com/JarbasHiveMind/HiveMind-core/wiki/(ovos-core)-message-routing). each connected client only receives what it should, not everything



______________

#### the hivemind is a protocol that transparently integrates with OVOS

can we connect the android app using hivemind?

- step 1: install the [android app](https://github.com/MycroftAI/Mycroft-Android)
- step 2: get the ip address of your *OVOS node* and set the url as `wss://{ip_address}:5678?authorization={encoded}`
    - TODO this [url]() will generate the encoded url for you
    - decoding [#](https://github.com/JarbasHiveMind/HiveMind-core/blob/dev/jarbas_hive_mind/nodes/master.py#L40) encoding [#](https://github.com/JarbasHiveMind/hivemind_websocket_client/blob/master/hivemind_bus_client/client.py#L102) encoding javascript [#](https://github.com/JarbasHiveMind/HiveMind-js/blob/master/static/js/hivemind.js#L92)
    - TODO need a flag in hivemind-core to support this (assume all messages are payloads of [type "bus"](https://github.com/JarbasHiveMind/HiveMind-core/wiki/Messages#OVOS-messages)) per client (?)
- step 3 :x: **NOT YET**!

Does this mean we can turn any OVOS thingy in a hivemind node? we are getting there...


______________

#### the hivemind provides skill isolation


You have probably ran into bad warnings about installing skills from outside the marketplace, you should pay attention to them!

sometimes the skill is trustworthy, but dangerous

- step 1: install [monkey patcher skill]()
- step 2: OVOS got new features!
- step 3: :tada:
- step 4: :x: a skill changed ovos-core at runtime (?)

this can accidentally break other skills and over time get out of sync with ovos-core if not maintained, but some skills can be bad on purpose and cause a lot of damage

- step 1: install [bus bricker skill](https://github.com/EvilJarbas/BusBrickerSkill)
- step 2: as soon as OVOS loads this skill everything stops workings, [related issue](https://github.com/MycroftAI/ovos-core/issues/2905)
- step 3: :x: obviously EVIL

the [LocalHive](https://github.com/JarbasHiveMind/LocalHive) allows loading skills as if they were a hive node, they are completely isolated from each other and can run in their own container or .venv

NOTES:
- skills with conflicting requirements can coexist
- skills can be packaged by package managers
- :exclamation: **LocalHive replaces ovos-core**

______________

#### the hivemind can be used to integrate OVOS with any platform

How many times have you wanted to integrate OVOS with a chat platform? maybe just for a quick demo?

- step 1: install the [hackchat bridge](https://github.com/JarbasHiveMind/HiveMind-HackChatBridge)
- step 2: go the newly created channel and invite anyone to check it out
- step 3 :tada: **shared OVOS with friends**

What about something more useful?

- step 1: install the [mattermost bridge](https://github.com/JarbasHiveMind/HiveMind_mattermost_bridge)
- step 2: create skills and tell your coworkers how to use them in their workflow
- step 3 :tada: **OVOS is a productivity tool**

Even if not interested in exposing it to the public there are some private and secure ways to interact with your *OVOS node*

- step 1: install the [deltachat bridge](https://github.com/JarbasHiveMind/HiveMind-deltachat-bridge)
- step 2: install [ðeltachat](https://delta.chat/en) in your phone
- step 3 :tada: **control OVOS from your phone**

How can you use the hivemind to integrate with an existing service, maybe it's for costumer support on some platform

- step 1: install the [twitch bridge](https://github.com/JarbasHiveMind/HiveMind-twitch-bridge) 
- step 2: create skills to provide support to your users
     - TIP: add a greeting and help command
- step 3 :tada: **OVOS is a chatbot**


_____


#### the hivemind is a set of permissively licensed libraries

If you already have an application that interacts with the messagebus how hard is it to make it use the hivemind?

- step 1: replace all OVOS bus connections with [hivemind-bus-client](https://github.com/JarbasHiveMind/hivemind_websocket_client/) (python) or [HiveMindJS](https://github.com/JarbasHiveMind/HiveMind-js) (javascript)
- step 2: hope there arent any bugs
- step 3 :tada:

______________


