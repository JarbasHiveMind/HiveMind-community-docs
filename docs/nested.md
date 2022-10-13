# Nested Hives

HiveMind is a suite of programs and thin clients that distribute an instance of Mycroft. One AI, many devices.

A Hive is defined by providing at least one instance of HiveMind-core, which is most commonly run as companionware alongside Mycroft-core. It and nodes like it can serve as a Hive's "brain." In a tree diagram, this node is at the top.

A **node** is any device or software client connected to a Hive.

While a node is serving as a Hive's "brain," it is referred to as that Hive's **master node**.

Nodes can have clients of their own. A specific node and all its clients, including clients of clients, constitute a **cluster**. Clusters within clusters are called **subclusters**.

A **satellite** is a node with I/O capabilities, such as the HiveMind Voice Satellite, our reference implementation for a voice client.

A **slave** is a node with limited functionality, which cannot issue instructions. Examples include sensors, displays, buttons, or, in software terms, echo clients and connection testers.

---

Say we share a house. We each have an AI, with a primary instance of Mycroft. My AI's name is John, and yours is Jane.

As housemates, we share most of our IoT devices. We want John and Jane each to be able to control our smart home, but I don't want you to be able to control John, and you don't want me to control Jane.

No problem! We give the house a Hive of its own, with at least one instance of Mycroft. Call it George. Hives can be nested, so we have John and Jane connect to George as clients. They can talk *to* George, but they can't talk to each other, except *through* George (and only when George deigns to pass the message along.)

Now, when I tell John to mess with the lights, John will go through George. When you tell Jane to set the temperature, Jane will go through George. When you have guests, you might allow them to use George directly, or, while they're visiting, nest their personal Hive under George!

Because Hives can be "cut" as immediately as they can be nested, it makes sense to think of Jane and John as separate Hives of their own, even while they're also part of George's cluster. From George's perspective, John and Jane are *subclusters*; from John's and Jane's perspectives, while they are nested, George is *master*. However, as soon as they split, they are once again their own masters. You might even have some devices isolated from George. Those devices wouldn't see themselves as part of a subcluster. They'd see themselves as part of Jane's cluster, with Jane as master, full stop.

Auto-discovery (configurable) is used to facilitate joining, leaving, and nesting Hives. Once you've configured your devices, you can nest and decouple Hives as often as you like!

**Note:** "Parallel" nesting, such as having Jane nested beneath both George and a second master *at the same time*, is roadmapped, but not yet supported. This would, for example, enable Jane to remain in ordinary communication with George while you're at work, *and* let Jane communicate with the Hive controlling your workplace. No ETA, as the developers have many higher priorities.

