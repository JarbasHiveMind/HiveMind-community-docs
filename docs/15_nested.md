### Nested Hives

Once you have a basic hive set up, you can add more Minds to it and connect them to each other.

Read [the protocol page](04_protocol.md) to learn how minds interact.

## Nested Hiveminds in action

Consider two housemates, Mom and Dad, who each have an AI assistant running on OpenVoiceOS, named John and Jane.

![imagem](https://github.com/JarbasHiveMind/HiveMind-community-docs/assets/33701864/486d97a1-484c-42e0-a556-193cf70fe6c6)

Mom and Dad share a house and most of their IoT devices. They want their assistants to control the smart home individually, without interfering with each other's commands. To do this, they create a hive for their house, named George, with at least one OpenVoiceOS instance acting as the brain.

Mom and Dad connect John and Jane as clients to the George hive. This setup lets John and Jane talk to George individually, not directly with each other. Their messages pass through George, which acts as an intermediary and keeps communication in order. John is connected to Dad's phone and calendar, and knows Dad's favorite songs. This keeps George free of personal data and gives Dad a personalized experience. The same holds for Jane and Mom: alarms and music playlists stay separate.

![imagem](https://github.com/JarbasHiveMind/HiveMind-community-docs/assets/33701864/1da8c4f5-243b-4b58-9465-e59612d5d74e)

As soon as a hive is decoupled, such as when Mom and Dad split their hives, each side becomes an independent master again.

When Dad tells his assistant to adjust the lights, the message goes through George. When Mom tells her assistant to set the temperature, the command routes through George too. George becomes the central point of control for shared devices, while John and Jane still work independently.

![imagem](https://github.com/JarbasHiveMind/HiveMind-community-docs/assets/33701864/e0634651-ab97-475a-bf7e-5cef68235c40)

If guests visit, Mom and Dad can grant them access to George directly, for example through the voice satellites around the house, or create a temporary guest hive under George.

This setup lets Mom and Dad connect and disconnect hives as needed.

![imagem](https://github.com/JarbasHiveMind/HiveMind-community-docs/assets/33701864/4b3e04ed-cc06-4405-a7e8-4e8b22dfb0cf)

Nested hiveminds give you a flexible way to manage AI systems and devices. Clusters nested within a master hive form a scalable, hierarchical structure.

## Permissions

Consider another scenario to explore nested hiveminds further. Suppose Mom and Dad have a guest, Bob, who also has his own AI assistant. To give Bob access to the shared smart home functions, they let Bob's assistant connect to the George hive as a client.

Mom and Dad still want to limit Bob's assistant's permissions within their ecosystem. They configure hivemind-core, acting as a firewall, to stop Bob's assistant from placing orders or reading sensitive information belonging to Mom and Dad. This fine-grained control keeps the guest AI within defined boundaries, so privacy and security hold for everyone involved.

![imagem](https://github.com/JarbasHiveMind/HiveMind-community-docs/assets/33701864/ae8530d6-a465-4ae6-b556-b3f50562d810)

Mom and Dad can also create a separate nested assistant for their children, with access limited to functions suitable for their age. This nested assistant has restricted permissions and tailored interactions, so kids get a safe AI experience while their privacy stays intact.

![imagem](https://github.com/JarbasHiveMind/HiveMind-community-docs/assets/33701864/217b4185-7e1b-46f0-af83-b3c097ff2b5f)

Nested hiveminds give you a versatile way to manage multiple AI assistants and tailor their capabilities to each person's needs. By configuring access permissions and firewalls, you can build an ecosystem that keeps privacy, security, and a personalized experience for each participant.

![img_15.png](img_15.png)

---
[← Terminology](02_terminology.md) · [Home](index.md) · [Plugins →](04_plugins.md)
