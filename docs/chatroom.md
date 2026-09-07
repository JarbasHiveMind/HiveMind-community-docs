# Chatroom and Universal Translator

HiveMind carries encrypted node-to-node messages with [INTERCOM](04_protocol.md#intercom-message). A small application on top of that is a chatroom: members send lines of text to each other, and each member renders an incoming line the way it wants — print it, speak it, or translate it into its own language.

The [hivemind-chatroom](https://github.com/JarbasHiveMind/hivemind-chatroom) project is a working example with a tutorial.

## How it works

Each member is a satellite. A line of text travels as an INTERCOM: it is encrypted to each recipient's public key, signed by the sender, and routed to the recipient across the hive. A direct message names one recipient. A group message goes to every member of the roster.

The hub does not read the message. INTERCOM is node-to-node, so the hub only relays it. The hub needs no skills and no language model for this — the agent behind the hub does not take part. The same hive that runs a voice assistant also carries the chatroom, and neither affects the other.

The transform runs on the receiver. A member translates an incoming line into its own language after it arrives, not before it is sent. There is no shared chatroom language and no central translator. A member that wants speech runs the text through text-to-speech; a member that wants text does nothing. One sender reaches many receivers, and each renders the message its own way.

## Universal translator

Turn on the translation transform and the room becomes a universal translator: everyone writes or speaks in their own language, and everyone reads or hears every message in theirs. Each member translates on receipt, so members can use different languages at the same time with no coordination.

Translation can run on each member (the member calls a translate service, or translates locally) or on a shared service the members call. The first keeps the hive a plain router and lets each member choose its language on its own.

## Voice, or not

Voice is a receiver-side choice. A member that adds text-to-speech becomes a voice endpoint that hears the room in its language. A member that does not stays a text client. Both share one room.

## Addressing and trust

A member needs two things to take part: the public keys of the peers it addresses, and those same keys in its trust store, because INTERCOM verifies the sender's signature before it delivers anything ([Encryption](19_crypto.md)). A message from a key the receiver does not hold is dropped as an unverified signature.

A member learns this roster from the hub, which knows every member's key from [pairing](03_pairing.md) and presence. This is the one part to get right: a member that never receives anything is usually missing a peer's key from its trust store.

---
[← Home Assistant](07_homeassistant.md) · [Home](index.md) · [Transport →](04_protocol.md)
