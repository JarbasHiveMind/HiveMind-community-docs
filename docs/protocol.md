# Protocol

The ability to exchange information and commands seamlessly is made possible through the Hivemind Protocol.

Now, let's explore the different message types introduced by the Hivemind Protocol.

## Payload Messages

Payload messages contain a OpenVoiceOS `Message` object, serving as a container for information or commands.

### Bus Message

The `BUS` message facilitates single-hop communication, flowing between slaves and masters. When a master receives a `BUS` message, it examines its global whitelist/blacklist and slave permissions. If the slave is authorized to inject the message into the ovos-core message bus, the message is injected accordingly. Subsequently, any direct responses from the master's ovos-core, triggered by the injected message, are forwarded back to the originating slave. The permissions can be defined based on criteria such as message type, intent type, skill ID, access key, and IP address rules.

![](https://raw.githubusercontent.com/JarbasHiveMind/HiveMind-core/dev/resources/bus.gif)


### Shared Bus Message

The `SHARED_BUS` message is designed for single-hop communication, exclusively flowing from a slave to a master. In this scenario, the master passively monitors the ovos-core message bus on the slave device. It is important to note that the `SHARED_BUS` functionality needs to be explicitly enabled in the slave device configuration, as the default behavior does not involve sharing the ovos-core bus. In the case of terminals, messages of type `BUS` are injected into their respective masters and are simultaneously forwarded as `SHARED_BUS` messages to the subsequent masters in the hierarchy.

![](https://raw.githubusercontent.com/JarbasHiveMind/HiveMind-core/dev/resources/shared_bus.gif)


## Transport Messages

Transport messages contain another `HiveMessage` object as their payload.

### Broadcast Message

The `BROADCAST` message plays a crucial role in multi-hop communication, flowing from a master to its associated slaves. It serves as a means to disseminate a message to all slaves connected to a particular master, effectively "sending the message down" the authority chain. Through the use of broadcast messages, information or commands can be efficiently transmitted throughout the hive, facilitating widespread collaboration.

![](https://raw.githubusercontent.com/JarbasHiveMind/HiveMind-core/dev/resources/broadcast.gif)

A Mind may decide to broadcast a message at any time

### Escalate Message

The `ESCALATE` message is an essential multi-hop message that travels from a slave to its master. It enables the slave to send a message up the authority chain, ensuring that the message reaches higher-level minds within the hierarchy. By employing escalate messages, valuable information or commands can be elevated, allowing for higher-level decision-making within the hive.

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

The `PROPAGATE` message is a versatile multi-hop message that flows both from a master to its slaves and vice versa. This message type enables a master to send a message to all its direct connections, ensuring that the message reaches every relevant node within the hive. Propagate messages are instrumental in disseminating critical information or commands throughout the network, enabling comprehensive collaboration and coordination.

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