# Protocol

The ability to exchange information and commands seamlessly is made possible through the Hivemind Protocol.

Now, let's explore the different message types introduced by the Hivemind Protocol.

## Payload Messages

Payload messages contain a OpenVoiceOS `Message` object, serving as a container for information or commands.

### Bus Message

The `BUS` message facilitates single-hop communication, flowing **between slaves and masters**. When a master receives a `BUS` message, it examines its global whitelist/blacklist and slave permissions. If the slave is authorized to inject the message into the ovos-core message bus, the message is injected accordingly. Subsequently, any direct responses from the master's ovos-core, triggered by the injected message, are forwarded back to the originating slave. The permissions can be defined based on criteria such as message type, intent type, skill ID, access key, and IP address rules.

![](https://raw.githubusercontent.com/JarbasHiveMind/HiveMind-core/dev/resources/bus.gif)

Terminal applications such as the voice-sat usually inject natural language queries from users, others applications such as the Home Assistant integration may inject other messages if authorized to do so

you can authorize message_types via the [hivemind-core](https://github.com/JarbasHiveMind/HiveMind-core/) package

Reference BUS payloads for OVOS can be found [here](https://github.com/OpenVoiceOS/message_spec)

```bash
$hivemind-core allow-msg "speak"
```

the [hivemind-websocket-client](https://github.com/JarbasHiveMind/hivemind_websocket_client) package provides an utility to connect to a Mind and send a bus message

```bash
$ hivemind-client send-mycroft --help
Usage: hivemind-client send-mycroft [OPTIONS]

  send a single mycroft message

Options:
  --key TEXT       HiveMind access key (default read from identity file)
  --password TEXT  HiveMind password (default read from identity file)
  --host TEXT      HiveMind host (default read from identity file)
  --port INTEGER   HiveMind port number (default: 5678)
  --siteid TEXT    location identifier for message.context  (default read from
                   identity file)
  --msg TEXT       ovos message type to inject
  --payload TEXT   ovos message.data json
  --help           Show this message and exit.

```

### Shared Bus Message

The `SHARED_BUS` message is designed for single-hop communication, exclusively flowing **from slave to master**. In this scenario, the master passively monitors the ovos-core message bus on the slave device. It is important to note that the `SHARED_BUS` functionality needs to be explicitly enabled in the slave device configuration, as the default behavior does not involve sharing the ovos-core bus. In the case of terminals, messages of type `BUS` are injected into their respective masters and are simultaneously forwarded as `SHARED_BUS` messages to the subsequent masters in the hierarchy.

> NOTE: this is usually done by [HiveMind Skill](https://github.com/JarbasHiveMind/ovos-skill-fallback-hivemind) if enabled in settings

![](https://raw.githubusercontent.com/JarbasHiveMind/HiveMind-core/dev/resources/shared_bus.gif)


## Transport Messages

Transport messages contain another `HiveMessage` object as their payload.

> NOTE: these message types only matter for [Nested Hives](https://jarbashivemind.github.io/HiveMind-community-docs/15_nested/)

### Broadcast Message

The `BROADCAST` message plays a crucial role in multi-hop communication, flowing **from a master to its associated slaves**. It serves as a means to disseminate a message to all slaves connected to a particular master, effectively "sending the message down" the authority chain. Through the use of broadcast messages, information or commands can be efficiently transmitted throughout the hive, facilitating widespread collaboration.

A Mind may decide to broadcast a message at any time

![](https://raw.githubusercontent.com/JarbasHiveMind/HiveMind-core/dev/resources/broadcast.gif)

A broadcast message may also contain a `"target_site_id"`

When a slave receives a broadcast message it will check if it's own `"site_id"` matches the `"target_site_id"`, and if it does, the BUS message is injected into the slave internal bus

This allows for example a Mind to make all the satellites in `site_id: "kitchen"` speak a message out loud

### Escalate Message

The `ESCALATE` message is an essential multi-hop message that travels **from a slave to its master**. It enables the slave to send a message up the authority chain, ensuring that the message reaches higher-level minds within the hierarchy. By employing escalate messages, valuable information or commands can be elevated, allowing for higher-level decision-making within the hive.

![](https://raw.githubusercontent.com/JarbasHiveMind/HiveMind-core/dev/resources/escalate.gif)

the [hivemind-websocket-client](https://github.com/JarbasHiveMind/hivemind_websocket_client) package provides an utility to connect to a Mind and escalate a message

```bash
$ hivemind-client escalate --help
Usage: hivemind-client escalate [OPTIONS]

  escalate a single mycroft message

Options:
  --key TEXT       HiveMind access key (default read from identity file)
  --password TEXT  HiveMind password (default read from identity file)
  --host TEXT      HiveMind host (default read from identity file)
  --port INTEGER   HiveMind port number (default: 5678)
  --siteid TEXT    location identifier for message.context  (default read from
                   identity file)
  --msg TEXT       ovos message type to inject
  --payload TEXT   ovos message.data json
  --help           Show this message and exit.

```

### Propagate Message

The `PROPAGATE` message is a versatile multi-hop message that flows **both from a master to its slaves and vice versa**. This message type enables a master to send a message to all its direct connections, ensuring that the message reaches every relevant node within the hive. Propagate messages are instrumental in disseminating critical information or commands throughout the network, enabling comprehensive collaboration and coordination.

![](https://raw.githubusercontent.com/JarbasHiveMind/HiveMind-core/dev/resources/propagate.gif)

the [hivemind-websocket-client](https://github.com/JarbasHiveMind/hivemind_websocket_client) package provides an utility to connect to a Mind and propagate a message

```bash
$ hivemind-client propagate --help
Usage: hivemind-client propagate [OPTIONS]

  propagate a single mycroft message

Options:
  --key TEXT       HiveMind access key (default read from identity file)
  --password TEXT  HiveMind password (default read from identity file)
  --host TEXT      HiveMind host (default read from identity file)
  --port INTEGER   HiveMind port number (default: 5678)
  --siteid TEXT    location identifier for message.context  (default read from
                   identity file)
  --msg TEXT       ovos message type to inject
  --payload TEXT   ovos message.data json
  --help           Show this message and exit.

```


## Protocol Versions

| Protocol Version     | 0   | 1   |
|----------------------|-----|-----|
| json serialization   | yes | yes |
| binary serialization | no  | yes |
| pre-shared AES key   | yes | yes |
| password handshake   | no  | yes |
| PGP handshake        | no  | yes |
| zlib compression     | no  | yes |


some clients such as HiveMind-Js do not yet support protocol V1
