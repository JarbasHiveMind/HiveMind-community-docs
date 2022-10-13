## Node Repositories

- [Client Libraries](#client-libraries)
  * [HiveMind-websocket-client](https://github.com/JarbasHiveMind/hivemind_websocket_client)
  * [HiveMindJs](https://github.com/JarbasHiveMind/HiveMind-js)
- [Terminals](#terminals)
  * [Remote Cli](https://github.com/OpenJarbas/HiveMind-cli)
  *  [Voice Satellite](https://github.com/OpenJarbas/HiveMind-voice-sat)
  * [Flask Chatroom](https://github.com/JarbasHiveMind/HiveMind-flask-template)
  * [Webchat](https://github.com/OpenJarbas/HiveMind-webchat)
- [Bridges](#bridges)
  * [Mattermost Bridge](https://github.com/OpenJarbas/HiveMind_mattermost_bridge)
  * [HackChat Bridge](https://github.com/OpenJarbas/HiveMind-HackChatBridge)
  * [Twitch Bridge](https://github.com/OpenJarbas/HiveMind-twitch-bridge)
  *  [DeltaChat Bridge](https://github.com/JarbasHiveMind/HiveMind-deltachat-bridge)
- [Minds](#minds)
  * [LocalHive](https://github.com/JarbasHiveMind/LocalHive)
  * [NodeRed](https://github.com/OpenJarbas/HiveMind-NodeRed)
- [FakeCrofts](#fakecrofts)
  * [DuckDuckGo](https://github.com/JarbasHiveMind/Fakecroft-DDG)

WIP - different existing or planned hivemind nodes, most of this does not exist yet!!!

## LocalHive

The LocalHive is a hardened mycroft skills service, the mycroft messagebus is replaced with a hivemind connection

https://github.com/JarbasHiveMind/LocalHive

_"security as a requirement, not a feature"_

- the LocalHive is HTTP only
- the LocalHive uses no crypto
- the LocalHive does not require accessKey, instead it only accepts connections coming from 0.0.0.0
- the LocalHive rejects all connections not coming from 0.0.0.0
- the LocalHive runs in port 6989
- skills can not listen to each other's traffic
- by default skills only register and trigger intents, nothing else
- each skill can run in it's own .venv with it's own requirements

LocalHive is built on top of HolmesV and **can not coexist with mycroft-core, it replaces it**, be sure to use a dedicated .venv

The default terminals can connect to LocalHive as long as they are running on same device, the full mycroft stack can be replaced with the equivalent terminal, this is the recommended way to run mycroft in a dedicated device

## wormhole node

- its actually 2 nodes
- node 1 drops messages in place X
- node 2 retrieves messages from place X
- messages are literal hive protocol messages
- X is any transport layer, literally anything
- nodes might not know each other at all as long as they know how to retrieve stuff
- the objective is hiding location
- any master that sees node 2 just thinks it is node 1!

implementations:
- usenet anon message boards (read [#](https://github.com/JarbasAl/remailers/blob/master/examples/anon_message_retriever.py) post [#](https://github.com/JarbasAl/remailers/blob/master/examples/anon_post.py))


### storage node

- any node can leave a payload in a storage node + associated proof
   - optionally encrypted (recommended)
- a proof is a text string + same string encrypted with "receiver pubkey"
- any node can request any (encrypted) file
- the storage node will send the encrypted proof
- the node sends the decrypted proof
- if both match the node proves it is the receiver
- the storage node sends the file


note: connections to these nodes should be ephemeral, ie, nodes disconnect once the deed is done
note2: these can be public and hosted by random people, if you trust PGP

### rendevouz node

a variation of the above, imagine a scenario with a very very large hive, maybe some nodes are even public or half way across the world

- node fires a "query" hive message
- message contains the address of a storage node the mind should drop the answer in 
    - the response doesnt need to travel all the way back
    - optionally include node pubkey (may have been shared out of band)
- node checks the pre defined storage node every timestep until it receives an answer
    - depending on relationship with storage node it might be possible to use events instead
    - a storage node can be a http api (see [http bridge]() TODO)
- where did the answer come from?
