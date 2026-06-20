# Security

## Handshake and encryption

Every HiveMind connection begins with a handshake that establishes a shared session key without transmitting that key over the wire. The standard handshake (protocol v1) uses PBKDF2-SHA256:

```
Server → Client  HELLO        (node_id, server public key — plaintext)
Server → Client  HANDSHAKE    (capabilities: binarize, crypto_required, etc. — plaintext)
Client → Server  HANDSHAKE    (binarize flag + PBKDF2 envelope — plaintext)
Server → Client  HANDSHAKE    (PBKDF2 envelope — plaintext)
Client → Server  HELLO        (session data, site_id, client public key — ENCRYPTED)
```

The handshake works as follows:

1. Each side generates a random IV and computes a salted-hash subject `HSUB = IV ‖ SHA256(IV + password)` (the IV concatenated with the hash, not a PBKDF2 value)
2. Both sides exchange their `HSUB` and `IV`
3. Each side verifies the other's `HSUB` by recomputing it locally
4. A common salt is derived: `salt = IV_client XOR IV_server`
5. The session key is derived with PBKDF2: `key = PBKDF2-HMAC-SHA256(password, salt, 100_000 iterations)` — 256 bits

After the handshake all messages are encrypted with the negotiated cipher using the derived session key. Both **ChaCha20-Poly1305** and **AES-GCM** are supported (config `allowed_ciphers: ["CHACHA20-POLY1305", "AES-GCM"]`; the runtime default is AES-GCM). Each message carries a unique nonce and an authentication tag.

An alternative v0 path skips the handshake and uses a pre-shared `crypto_key` directly (the legacy `Encryption Key` printed by `add-client`). This path is kept for backward compatibility only.

### RSA identity (asymmetric)

Each node maintains an **RSA 2048-bit** key pair, PEM-encoded and stored at `~/.config/hivemind/HiveMindComs.pem`. Encryption uses PKCS#1 OAEP; signatures use PSS with SHA-256. The public key is exchanged in the encrypted `HELLO` after the handshake. The RSA keys are used for:

- **INTERCOM** — encrypting point-to-point messages so intermediate nodes cannot read them
- **Node authentication** — verifying sender identity on INTERCOM messages via signature

Reset the RSA key at any time with `hivemind-client reset-pgp` (the command name is retained; it recreates the RSA key pair).

#### Bootstrapping satellite-to-satellite trust

For INTERCOM (and the signed trust checks on PROPAGATE / CASCADE), a node only accepts messages from peers whose public key it has been told to trust. That trust is configured out of band: each node maintains a `trusted_keys` mapping (alias → public-key string) in its identity file. Add a peer's key with `NodeIdentity.add_trusted_key(alias, pubkey)` (and `trusted_keys` / `remove_trusted_key` to read or revoke). After PING discovery has mapped the network, `HiveMapper.mark_trusted_nodes(trusted_keys)` flips each discovered node's `trusted` flag based on that mapping, so subsequent source-trust checks resolve quickly. Keys must be exchanged through a channel you already trust — there is no automatic key acceptance.

## Identity file

The identity file at `~/.config/hivemind/_identity.json` stores all credentials a satellite needs to connect:

| Field | Description |
|---|---|
| `access_key` | Unique access credential assigned by the hub |
| `password` | Used for PBKDF2 key derivation during handshake |
| `default_master` | Hub host address |
| `default_port` | Hub port (default 5678 for WebSocket) |
| `site_id` | Physical location identifier injected into OVOS context |
| `public_key` | RSA public key string |
| `secret_key` | Path to the RSA private key (PEM) file |

Write the identity file:

```bash
hivemind-client set-identity \
  --key <access_key> \
  --password <password> \
  --host <hub_host> \
  --port 5678 \
  --siteid living-room
```

## Permissions

Permissions in HiveMind are configured per client — there are no roles. Each client has its own `allowed_types` whitelist and per-client skill/intent blacklists.

### How the policy chain works

For every message received from a satellite, the hub runs a policy admission chain:

1. **`allowed_types` check** — the client's per-client whitelist is checked first. If the OVOS message type is not in the allowed list, the message is dropped immediately. This is fail-closed: a new client with an empty whitelist can send nothing.

2. **Policy plugins** — configured policies run in order. The default policy is `OVOSAgentPolicy`, which reads `Client.metadata` to build per-client session blacklists (skills and intents).

Admin flag (`make-admin`) does **not** bypass the `allowed_types` check. It signals to policy plugins that extra-privileged operations are permitted, but the hard ACL is always enforced first.

### Writing a custom policy

Admission control beyond the `allowed_types` ACL is an extension point. A custom policy is a plugin that subclasses `PolicyPlugin` (from `hivemind_plugin_manager.policy`) and is registered under the `hivemind.policy` entry-point group. The hub loads it into an ordered **policy chain** and calls it for every inbound message.

**The contract.** Override any of three hooks:

- `review(message, client)` — inspect a Mycroft `Message` before it is emitted onto the agent bus. Return a `Verdict`.
- `review_binary(payload, client)` — same, for binary payloads (e.g. raw audio). The default allows everything, so override it only if you care about binaries.
- `observe(message, client)` — fire-and-forget hook called *after* a message was successfully emitted. Use for counters, audit logs, telemetry. Must not raise.

A `Verdict` is either an allow or a deny:

- `Verdict.allow(*mutations)` — let the message proceed. Optionally carries one or more `Mutation` objects describing changes to apply before the next policy runs. (Concrete mutation types are agent-specific and ship with the agent plugin, e.g. the OVOS bridge's skill/intent blacklist mutations — they are not part of the base primitives.)
- `Verdict.deny(code, reason="", **data)` — drop the message and short-circuit the chain. `code` is a stable machine-readable string; the `DenyCodes` enum provides the built-in codes (`POLICY_ERROR`, `POLICY_CHAIN_UNAVAILABLE`, `ACL_DISALLOWED_TYPE`, `SESSION_ID_DEFAULT_FORBIDDEN`), but custom policies may emit their own string codes.

The chain is **fail-closed**: an exception raised inside `review` / `review_binary` (or a mutation's `apply`) is converted to `Verdict.deny("policy_error", ...)`. There is no operator knob to make it lenient. `Client.is_admin` is informational only — the chain runner never skips a policy based on it; a policy that wants to treat admins specially checks `client.is_admin` itself.

When a policy denies a message, the originating client is notified on the bus with a `hive.policy.denied` message carrying the `denied_type`, `code`, `reason`, and `data` from the verdict.

**The chain.** Operators configure the chain as an ordered list under `policy.chain` in `~/.config/hivemind-core/server.json`:

```json
{
  "policy": {
    "chain": [
      {"module": "hivemind-intent-quota-policy", "config": {"limit": 100}},
      {"module": "my-custom-policy", "config": {}, "optional": true}
    ]
  }
}
```

Each entry names a `module` (the entry-point name), an optional `config` dict passed to the plugin, and an optional `optional` flag. An `optional: true` policy that raises is logged and skipped (the chain continues); a mandatory policy that raises fails the chain closed. The built-in `MessageTypeACLPolicy` (the `allowed_types` ACL) is **always force-prepended** to the chain and is mandatory — it cannot be removed, made optional, or reordered, even if you list it explicitly. If the chain fails to build at startup, the hub installs a `DenyAllPolicy` fallback that rejects everything until the config is fixed.

**Minimal skeleton.** A policy plugin and its entry point:

```python
# my_policy/__init__.py
from hivemind_plugin_manager.policy import PolicyPlugin, Verdict


class MyPolicy(PolicyPlugin):
    def review(self, message, client):
        if message.msg_type == "some.forbidden.type":
            return Verdict.deny("forbidden_type",
                                "this type is never allowed here")
        return Verdict.allow()
```

```toml
# pyproject.toml
[project.entry-points."hivemind.policy"]
"my-custom-policy" = "my_policy:MyPolicy"
```

See [Writing Plugins — Policy plugins](../developers/writing-plugins.md#5-policy) for the full plugin-authoring walkthrough.

### Managing permissions via CLI

The node ID is a **positional** argument (not a `--node-id` option). If omitted, the command prompts you to pick a client.

```bash
# Allow a message type
hivemind-core allow-msg "speak" 2

# Remove an allowed message type
hivemind-core blacklist-msg "speak" 2

# Allow/deny ESCALATE from a client
hivemind-core allow-escalate 2
hivemind-core blacklist-escalate 2

# Allow/deny PROPAGATE from a client
hivemind-core allow-propagate 2
hivemind-core blacklist-propagate 2

# Blacklist a skill (OVOS-policy)
hivemind-core blacklist-skill "skill-homeassistant.openvoiceos" 2

# Un-blacklist a skill
hivemind-core allow-skill "skill-homeassistant.openvoiceos" 2

# Blacklist an intent (OVOS-policy)
hivemind-core blacklist-intent "HomeAssistant.DeviceControllerIntent" 2

# Un-blacklist an intent
hivemind-core allow-intent "HomeAssistant.DeviceControllerIntent" 2

# Grant admin flag
hivemind-core make-admin 2

# Revoke admin flag
hivemind-core revoke-admin 2

# Set arbitrary metadata (read by policy plugins)
hivemind-core set-metadata 2 --key role --value guest
```

### Per-client defaults

When a non-admin client is added via `add-client`, its `allowed_types` whitelist is **empty** — it is denied on every message until you explicitly `allow-msg` each message type it needs. (Admin clients bypass the whitelist.) This deny-all-by-default posture ensures that compromised credentials give an attacker no capability until access is granted.

## Transport security (TLS)

HiveMind can use TLS for the WebSocket transport. `hivemind-core listen` takes no flags — TLS is configured in `~/.config/hivemind-core/server.json` under `network_protocol.hivemind-websocket-plugin`:

```json
{
  "network_protocol": {
    "hivemind-websocket-plugin": {
      "host": "0.0.0.0",
      "port": 5678,
      "ssl": true,
      "cert_dir": "/path/to/certs",
      "cert_name": "mycert"
    }
  }
}
```

**For local networks**: a self-signed certificate is sufficient. Pass `--selfsigned` on satellite commands to accept it.

**For internet-facing deployments**: use a reverse proxy (nginx, Caddy, Traefik) with valid certificates from Let's Encrypt. Keep HiveMind on an internal port and expose only the proxy externally. Do not expose port 5678 directly to the internet.

## Security checklist

**Local/private networks:**

- [ ] Use a strong, randomly generated password (12+ chars, mixed case and symbols)
- [ ] Store the password in an environment variable, not in plain config files
- [ ] Firewall port 5678 to trusted subnets only
- [ ] Monitor logs for repeated failed handshakes

**Internet-facing deployments:**

- [ ] Deploy a reverse proxy (nginx proxy manager, Caddy, Traefik)
- [ ] Obtain valid TLS certificates (Let's Encrypt)
- [ ] Enable automatic certificate renewal
- [ ] Keep port 5678 unexposed to the public internet (behind the proxy only)
- [ ] Apply rate limiting and firewall rules at the proxy

## Limitations

- The password handshake provides confidentiality and mutual authentication but security is proportional to password entropy. Weak passwords are the primary attack surface.
- Without TLS, a network observer can see the encrypted ciphertext but not its content. TLS adds defence-in-depth.
- INTERCOM RSA authentication requires both nodes to have pre-exchanged public keys through a trusted channel.

---

**Next:** [Mesh Topology](mesh.md) for how permissions compose across nested hives, or [Writing Plugins](../developers/writing-plugins.md#5-policy) to ship a custom admission policy.
