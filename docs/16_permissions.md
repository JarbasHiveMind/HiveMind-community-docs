# HiveMind Permission System

HiveMind's permission system gives fine-grained control over access to resources, such as bus messages, skills, and intents, on a per-client basis. Unlike traditional Role-Based Access Control (RBAC), HiveMind configures each client individually, rather than assigning it a predefined role. This allows dynamic, flexible access management.

### Key concepts

1. **Client-specific permissions**: HiveMind assigns permissions to individual clients, such as users, devices, or applications. Each client can have a unique set of permissions based on its needs or restrictions. Permissions control access to bus messages, skills, and intents, and can be configured more granularly than in a typical RBAC system.

2. One predefined role: admin. A client may be marked admin with `--admin true`, or with `make-admin` and `revoke-admin` later. Admin grants the reserved `default` session and the right to originate `BROADCAST`, and only while `can_broadcast` is not revoked. It does not exempt a client from `allowed_types`.

3. **Routing permissions**: `can_broadcast`, `can_escalate` and `can_propagate` gate a client's right to originate `BROADCAST`, [`QUERY`](04_protocol.md#query-message)/`ESCALATE`, and [`CASCADE`](04_protocol.md#cascade-message)/`PROPAGATE` respectively. They are set with the `allow-broadcast`, `allow-escalate` and `allow-propagate` commands (and their `blacklist-` counterparts). For `QUERY` and `CASCADE`, this check runs before a frame claiming to be a response is routed, so a client without the matching permission cannot forge a response and deliver arbitrary content to another peer. See [Response routing](04_protocol.md#query-message) for the limit of that guarantee.

4. **HTTP and WebSocket parity**: `can_broadcast`, `intent_blacklist` and `skill_blacklist` from a client's record apply the same way over the HTTP transport as over WebSocket. See [Network Plugins](network_plugins.md).

5. **Fine-grained access control**: Permissions are not just "allowed" or "denied." You can configure access at a fine-grained level, down to individual bus messages, skills, and intents.

6. **Emergent roles**: HiveMind has no formal roles, but roles can emerge from client-specific configuration. A client with broad access can function like an "admin," while another client with limited access can serve as a "guest." These roles are not predefined; they follow from each client's permission settings.

### Comparison to traditional RBAC

| **Feature**         | **Traditional RBAC**                                | **HiveMind Permission System**                                                                              |
|---------------------|-----------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| **Role Definition**  | Predefined roles (e.g., admin, user, guest)         | No predefined roles; permissions are assigned per client                                                    |
| **Permissions**      | Roles are granted permissions to access resources   | Permissions are configured on a per-client basis                                                            |
| **Granularity**      | Roles typically have broad access to resources      | Permissions are fine-grained, allowing access control over individual resources (messages, skills, intents) |
| **Flexibility**      | Less flexible, roles are static                     | Highly flexible, permissions can be adjusted per client at any time                                          |
| **Emergent Roles**   | Predefined roles based on job function or hierarchy | Roles emerge based on client-specific configuration                                                         |

### How it works

1. **Client configuration**: each client in the HiveMind ecosystem has a custom configuration that determines which actions it can perform. You can adjust this configuration at any time.

2. **Dynamic permission assignment**: HiveMind assigns permissions per client, so an administrator can specify which bus messages, skills, and intents each client can access or perform.

3. **Examples**:
    - A trusted client might get access to a wide range of skills and intents, including ones that need elevated privileges.
    - A restricted client can have specific actions or skills blacklisted, to keep it within a tightly controlled scope.

By configuring each client independently, HiveMind's permission system gives administrators a customizable, secure way to manage access across the ecosystem, tailored to each client's needs.

---
[← Pairing](03_pairing.md) · [Home](index.md) · [Handshake →](12_handshake.md)
