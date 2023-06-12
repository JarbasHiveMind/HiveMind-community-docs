# Protocol

In the fascinating world of the Hivemind, a network of interconnected intelligent agents, the ability to exchange information and commands seamlessly is made possible through the Hivemind Protocol. This blog post delves into the intricacies of the protocol, particularly focusing on the concept of nested Hiveminds. By understanding the inner workings of the protocol and the possibilities it unlocks, we gain insight into the potential of distributed AI systems.

## Terminology

Before we delve into the depths of the Hivemind Protocol, let's familiarize ourselves with some key terms used within the ecosystem:

- **Node**: A device or software client connected to the Hivemind network.
- **Mind**: A node that actively listens for connections and provides the functionality of Mycroft-core.
- **Fakecroft**: A mind that imitates Mycroft-core without actually running it, potentially located within another mind in the chain or not using Mycroft-core at all.
- **Slave**: A mind that connects to another mind and accepts commands from it.
- **Terminal**: A user-facing node that connects to a mind but doesn't accept connections itself.
- **Bridge**: A node that links an external service to a mind.
- **Hive**: A collection of interconnected nodes forming a collaborative network.
- **Master Mind**: The highest-level node in a hive that is not connected to any other nodes but receives connections from them.

Now, let's explore the different message types introduced by the Hivemind Protocol.

## Payload Messages

Payload messages contain a Mycroft `Message` object, serving as a container for information or commands.

### Bus Message

The `BUS` message facilitates single-hop communication, flowing between slaves and masters. When a master receives a `BUS` message, it examines its global whitelist/blacklist and slave permissions. If the slave is authorized to inject the message into the Mycroft-core message bus, the message is injected accordingly. Subsequently, any direct responses from the master's Mycroft-core, triggered by the injected message, are forwarded back to the originating slave. The permissions can be defined based on criteria such as message type, intent type, skill ID, access key, and IP address rules.

### Shared Bus Message

The `SHARED_BUS` message is designed for single-hop communication, exclusively flowing from a slave to a master. In this scenario, the master passively monitors the Mycroft-core message bus on the slave device. It is important to note that the `SHARED_BUS` functionality needs to be explicitly enabled in the slave device configuration, as the default behavior does not involve sharing the Mycroft-core bus. In the case of terminals, messages of type `BUS` are injected into their respective masters and are simultaneously forwarded as `SHARED_BUS` messages to the subsequent masters in the hierarchy.

## Transport Messages

Transport messages contain another `HiveMessage` object as their payload.

### Broadcast Message

The `BROADCAST` message plays a crucial role in multi-hop communication, flowing from a master to its associated slaves. It serves as a means to disseminate a message to all slaves connected to a particular master, effectively "sending the message down" the authority chain. Through the use of broadcast messages, information or commands can be efficiently transmitted throughout the hive, facilitating widespread collaboration.

### Escalate Message

The `ESCALATE` message is an essential multi-hop message that travels from a slave to its master. It enables the slave to send a message up the authority chain, ensuring that the message reaches higher-level minds within the hierarchy. By employing escalate messages, valuable information or commands can be elevated, allowing for higher-level decision-making within the hive.

### Propagate Message

The \`PROPAGATE

\` message is a versatile multi-hop message that flows both from a master to its slaves and vice versa. This message type enables a master to send a message to all its direct connections, ensuring that the message reaches every relevant node within the hive. Propagate messages are instrumental in disseminating critical information or commands throughout the network, enabling comprehensive collaboration and coordination.

## Conclusion

The Hivemind Protocol, with its diverse range of message types, serves as the backbone of communication within nested Hiveminds. By understanding and harnessing the power of these messages, nodes within the network can collaborate seamlessly, exchanging information, commands, and responses. As the field of distributed AI continues to evolve, the Hivemind Protocol messages will undoubtedly play a pivotal role in shaping the future of intelligent systems. With each advancement, the potential for innovative applications and the realization of a truly interconnected AI ecosystem becomes increasingly tangible. The nested Hivemind represents a compelling paradigm for distributed AI, unlocking new possibilities and paving the way for exciting developments in the field.