# HiveMind registry

The HiveMind protocol is currently under heavy development, and design is ongoing. At the moment, we are looking toward a framework where most inter-device and inter-application messages will be defined by formal protocols, each corresponding to an *Action* that a HiveMind device can take.

These are not as intimidating as they sound. They're basically just another data structure. If you can work with dictionaries, you can work with these. The biggest differences will be that order matters, and data is strictly typed.

## What is an Action?

An *Action* is an instruction or request which can be issued from one HiveMind device to another, if permissions allow. It has a standard name and a 128-bit GUID. Implementations can use either one to refer to an Action, and other implementations should be able to consume an appropriately-constructed message.

If you and I write programs that need to send or consume the same kind of message, we might as well use the same message. I need a message to place a phone call, you need a message to place a phone call, let's write a message to place a phone call.

The reasoning is simple: not only won't people duplicate work, we won't even have to know about each other to write interoperable code!

## The HiveMind Action Registry

### The scenario

Imagine a video doorbell. I make the doorbell, which speaks HiveMind as a native language. You write a Mycroft Skill that detects when doorbells ring, and tells Mycroft to speak. A third person writes a program that detects when *video* doorbells ring, locates you within your home, and opens a video link on the nearest screen.

Why should any us need to know about each other's existence, let alone each other's code? I should focus on programming a microcontroller, and you should focus on your really cool Skill. You shouldn't need to worry about how your Skill will know about my doorbell, nor write a compatibility layer for 10 different kinds of doorbell.

### Enter the Registry

We will each go to the HiveMind Action Registry, and search for terms like 'doorbell' and 'ring doorbell'. There, we will find a standardized message `RING_DOORBELL`, defined both in JSON and binary terms.

That message might carry metadata, such as whether the doorbell is video-equipped, or maybe it provides for the doorbell to have more than one button. All of that information would be present on the Action Registry. Depending on your project's needs, you can hardcode all of it (the doorbell) or expect HiveMind-core to provide it at runtime (the Mycroft Skill, probably.)

Thus empowered, everyone can write all the doorbell-related code they like. If, for some reason, the message defined on the Registry is insufficient, they can submit an RFC and add to that message's protocol. If their RFC is rejected, that's not the end of things; the HiveMind protocol leaves space for a few "free" instructions, so that your device can circumvent the standard message in a safe, sane way.

## Code

see https://github.com/JarbasHiveMind/hivemind_websocket_client/pull/4


