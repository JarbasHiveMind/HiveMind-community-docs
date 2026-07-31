# HiveMind Permission System

HiveMind's permission system gives fine-grained control over access to resources, such as bus messages, skills, and intents, on a per-client basis. Unlike traditional Role-Based Access Control (RBAC), HiveMind configures each client individually, rather than assigning it a predefined role. This allows dynamic, flexible access management.

### Key concepts

1. **Client-specific permissions**: HiveMind assigns permissions to individual clients, such as users, devices, or applications. Each client can have a unique set of permissions based on its needs or restrictions. Permissions control access to bus messages, skills, and intents, and can be configured more granularly than in a typical RBAC system.

2. **No predefined roles**: HiveMind does not rely on predefined roles like "admin" or "user." Instead, you configure each client independently with a tailored set of permissions. For example, a "basic client" might get access to general voice commands, while a "restricted client" could have specific skills or intents blocked.

3. **Fine-grained access control**: Permissions are not just "allowed" or "denied." You can configure access at a fine-grained level, down to individual bus messages, skills, and intents.

4. **Emergent roles**: HiveMind has no formal roles, but roles can emerge from client-specific configuration. A client with broad access can function like an "admin," while another client with limited access can serve as a "guest." These roles are not predefined; they follow from each client's permission settings.

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
