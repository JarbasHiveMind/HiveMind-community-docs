# Nested Hives

Two people share a house. Each wants their own assistant — their own music, their
own calendar, their own alarms — but they also share the lights, the thermostat, and
the front door. Neither wants their private data leaking into the other's assistant,
and neither wants to manage two separate copies of the smart home.

A single assistant can't do this. Nested hives can: you let assistants connect to
*other* assistants, and hand each connection exactly the permissions it should have.

!!! abstract "In a nutshell"
    - A **hive** is one hivemind-core and everything connected under it. A hive can connect to another hive as if it were an ordinary satellite — that's **nesting**.
    - Messages between nested hives pass *through* the parent, so a child can reach the shared world without reaching its siblings directly.
    - Each connection carries its own permissions, so the parent acts as a firewall — a guest or a child sees only what you allow.
    - Unplug a nested hive and it's a fully independent assistant again.

## A house with two minds

Meet Mom and Dad. Dad's assistant is **John**; Mom's is **Jane**. Each runs its own
hivemind-core, tied to personal things — John knows Dad's phone, his calendar, his
favourite songs; Jane keeps Mom's playlists and alarms. Kept apart, they never mix.

But the *house* has shared devices, so they stand up one more hivemind-core for the
home and name it **George**. George owns the lights, the thermostat, the shared
speakers. John and Jane each connect to George the same way a microphone satellite
would — as clients.

![Mom and Dad each with their own assistant](https://github.com/JarbasHiveMind/HiveMind-community-docs/assets/33701864/486d97a1-484c-42e0-a556-193cf70fe6c6)

Now the shape matters. John and Jane talk **to** George, not to each other. When Dad
says "turn on the lights," John forwards the request up to George, who owns the
lights and acts. When Mom sets the temperature, the same thing happens through
George. George is the shared point of control; John and Jane stay private islands
hanging off it.

![John and Jane both connect to George](https://github.com/JarbasHiveMind/HiveMind-community-docs/assets/33701864/1da8c4f5-243b-4b58-9465-e59612d5d74e)

Personal data never crosses over. Dad's calendar lives on John; Mom's alarms live on
Jane; only the messages meant for the shared house ever reach George. And the moment
Mom and Dad decouple their hives — say they move apart — John and Jane are simply two
independent assistants again, no untangling required.

![Shared control flows through George](https://github.com/JarbasHiveMind/HiveMind-community-docs/assets/33701864/e0634651-ab97-475a-bf7e-5cef68235c40)

## Guests, on your terms

A friend, **Bob**, comes to stay, and he brings his own assistant. You want him to
work the lights like everyone else — but not to place orders on your accounts or read
anything personal.

So you connect Bob's assistant to George as a client, and give *that connection* a
narrow permission set. hivemind-core is the firewall: it lets Bob's messages through
for the shared devices and drops everything else, per the [permissions](security.md)
you set. Bob gets a real, useful assistant in your home; your private world stays
sealed.

![A guest assistant with limited permissions](https://github.com/JarbasHiveMind/HiveMind-community-docs/assets/33701864/ae8530d6-a465-4ae6-b556-b3f50562d810)

The same trick makes a safe assistant for kids: a nested hive with a tailored,
limited permission set — the skills and content that suit them, and nothing else.
Every participant gets an experience shaped to them, and the boundaries are enforced
at the connection, not left to good behaviour.

![A tailored nested assistant for children](https://github.com/JarbasHiveMind/HiveMind-community-docs/assets/33701864/217b4185-7e1b-46f0-af83-b3c097ff2b5f)

## Why it scales

Nesting turns HiveMind from a star into a tree. A hivemind-core can be a parent to
some hives and a child to another, so you compose small, private clusters into larger
shared ones without any of them losing independence:

- A message that a hive can't answer locally can **escalate** up to a bigger parent.
- A message meant for many devices can **broadcast** down to the children.
- Each edge in the tree is authenticated and permissioned on its own.

That's how one household grows into a street, or one department into a company: not by
building a bigger single brain, but by connecting many small ones and letting the
parents decide what flows where. The routing that carries these messages up and down
the tree is the [mesh](mesh.md); the boundaries that keep each hive private are its
[permissions](security.md).

**Next:** [Security & Permissions](security.md) — the access keys and per-connection allowlists that make the firewall real.
