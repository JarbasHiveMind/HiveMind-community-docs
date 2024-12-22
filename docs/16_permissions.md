# HiveMind Permission System

HiveMind's permission system provides fine-grained control over access to resources, such as bus messages, skills, and intents, on a **per-client** basis. Unlike traditional Role-Based Access Control (RBAC), HiveMind emphasizes **client-specific configurations** rather than predefined roles, allowing for dynamic and flexible access management.

### Key Concepts

1. **Client-Specific Permissions**:
    - Permissions in HiveMind are assigned to **individual clients**, such as users, devices, or applications. This means that each client can have a unique set of permissions based on its specific needs or restrictions.
    - Permissions control access to bus messages, skills, and intents, enabling **dynamic configuration** that is more granular and flexible compared to traditional RBAC systems.

2. **No Predefined Roles**:
    - HiveMind does not rely on **predefined roles** like “admin” or “user.” Instead, each client is configured independently with a tailored set of permissions.
    - For instance, a “basic client” might have access to general voice commands, while a “restricted client” could have specific skills or intents blocked.

3. **Fine-Grained Access Control**:
    - Permissions are not just binary (e.g., “allowed” or “denied”). They can be configured at a **fine-grained level**, allowing administrators to control access to specific resources, such as individual bus messages, skills, and intents.
    - This allows for maximum flexibility in defining which clients have access to what, down to the level of individual interactions.

4. **Emergent Roles**:
    - While there are no formal roles in HiveMind, **roles can emerge** through client-specific configurations. For example, a client with broad access might function like an "admin," while another client with limited access could serve as a "guest."
    - These roles are not predefined but are dynamically created based on the client’s permission settings.

### Comparison to Traditional RBAC

| **Feature**         | **Traditional RBAC**                                | **HiveMind Permission System**                                                                              |
|---------------------|-----------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| **Role Definition**  | Predefined roles (e.g., admin, user, guest)         | No predefined roles; permissions are assigned per client                                                    |
| **Permissions**      | Roles are granted permissions to access resources   | Permissions are configured on a per-client basis                                                            |
| **Granularity**      | Roles typically have broad access to resources      | Permissions are fine-grained, allowing access control over individual resources (messages, skills, intents) |
| **Flexibility**      | Less flexible, roles are static                     | Highly flexible, permissions can be dynamically adjusted per client                                         |
| **Emergent Roles**   | Predefined roles based on job function or hierarchy | Roles emerge based on client-specific configuration                                                         |

### How It Works

1. **Client Configuration**:
    - Each client in the HiveMind ecosystem has a **custom configuration** that determines the actions it is allowed to perform. This configuration can be adjusted dynamically as needed.

2. **Dynamic Permission Assignment**:
    - Permissions are assigned on a **per-client basis**, providing administrators with the ability to specify which bus messages, skills, and intents each client can access or perform.

3. **Examples**:
    - A **trusted client** might be granted access to a wide range of skills and intents, including those requiring elevated privileges.
    - A **restricted client** could have specific actions or skills blacklisted to ensure it operates within a tightly controlled scope.

By leveraging client-specific configurations, HiveMind's permission system offers a highly customizable and secure approach to managing access across the ecosystem, allowing administrators to tailor the experience for each client based on their individual needs.