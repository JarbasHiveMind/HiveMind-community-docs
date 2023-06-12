# HiveMind Community Documentation

Welcome to the HiveMind Community Docs!

![](https://github.com/JarbasHiveMind/HiveMind-assets/raw/master/logo/hivemind-512.png)

HiveMind is a community-developed superset or extension of [OpenVoiceOS](https://openvoiceos.github.io/community-docs) the open-source voice operating system.

With HiveMind, you can extend one (or more, but usually just one!) instance of OpenVoiceOS to as many devices as you want, including devices that can't ordinarily run OpenVoiceOS!

HiveMind's developers have successfully connected to OpenVoiceOS from a PinePhone, a 2009 MacBook, and a Raspberry Pi 0, among other devices. OpenVoiceOS itself usually runs on our desktop computers or our home servers, but you can use any Mycroft-branded device, or [OpenVoiceOS](https://github.com/OpenVoiceOS/), as your central unit.

Join [Hivemind Matrix chat](https://matrix.to/#/#jarbashivemind:matrix.org) for general news, support and chit chat

# Getting started

At this moment development is in early stages, but mostly stable. Functionality is limited to basic questions and answers, but only because we haven't implemented the bit that lets OpenVoiceOS re-activate the mic to continue a discussion.

Full tutorials will follow later. For now, you wanna do this:

1. Clone this repo onto the computer or device that's running OpenVoiceOS.
2. Note the scripts in the `examples` folder. These are enough to spin up your Hive and connect to Mycroft from another device.
3. Edit `examples/add_keys.py` to create names and keys for your devices. You can use HiveMind without encryption, but it's discouraged for security reasons. (NOTE: HiveMind uses AES, so keys need to be exactly 16 characters long. This is far more secure than it sounds. Your bank probably uses it. So does the United States government, for some purposes.)
4. Ensure OpenVoiceOS is running.
5. Run `examples/mycroft_master.py`. Your Hive is now running, and you're ready to connect. See [here](https://github.com/JarbasHiveMind/HiveMind-core/wiki/Nodes) to find a client or terminal (ideally the Voice Satellite). You will need to know the hostname or IP address of the computer or device running OpenVoiceOS, and you'll also need the device name and key you created earlier.


The main configuration can be found at

    '~/.cache/json_database/HivemindCore.json'
