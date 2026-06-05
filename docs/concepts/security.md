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

The PBKDF2 handshake works as follows:

1. Each side generates a random IV and computes `HSUB = PBKDF2(password, IV, 100_000 iterations, SHA-256)`
2. Both sides exchange their `HSUB` and `IV`
3. Each side verifies the other's `HSUB` by recomputing it locally
4. A common salt is derived: `salt = IV_client XOR IV_server`
5. The session key is derived: `key = PBKDF2(password, salt, 100_000 iterations, SHA-256)` — 256 bits

After the handshake all messages are encrypted with **AES-256-GCM** using the derived session key. Each message carries a unique 12-byte nonce and a GCM authentication tag.

An alternative v0 path skips the handshake and uses a pre-shared `crypto_key` directly (the legacy `Encryption Key` printed by `add-client`). This path is kept for backward compatibility only.

### PGP identity (asymmetric)

Each node maintains a PGP key pair stored in `~/.config/hivemind/HiveMindComs.asc`. The public key is exchanged in the encrypted `HELLO` after the handshake. PGP keys are used for:

- **INTERCOM** — encrypting point-to-point messages so intermediate nodes cannot read them
- **Node authentication** — verifying sender identity on INTERCOM messages via signature

Reset your PGP keys at any time with `hivemind-client reset-pgp`.

## Identity file

The identity file at `~/.config/hivemind/_identity.json` stores all credentials a satellite needs to connect:

| Field | Description |
|---|---|
| `access_key` | Unique access credential assigned by the hub |
| `password` | Used for PBKDF2 key derivation during handshake |
| `default_master` | Hub host address |
| `default_port` | Hub port (default 5678 for WebSocket) |
| `site_id` | Physical location identifier injected into OVOS context |
| `public_key` | PGP public key (ASCII-armored) |
| `secret_key` | Path to PGP private key file |

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

### Managing permissions via CLI

```bash
# Allow a message type
hivemind-core allow-msg "speak" --node-id 2

# Remove an allowed message type
hivemind-core blacklist-msg "speak" --node-id 2

# Allow/deny ESCALATE from a client
hivemind-core allow-escalate --node-id 2
hivemind-core blacklist-escalate --node-id 2

# Allow/deny PROPAGATE from a client
hivemind-core allow-propagate --node-id 2
hivemind-core blacklist-propagate --node-id 2

# Blacklist a skill (OVOS-policy)
hivemind-core blacklist-skill "skill-homeassistant.openvoiceos" --node-id 2

# Un-blacklist a skill
hivemind-core allow-skill "skill-homeassistant.openvoiceos" --node-id 2

# Blacklist an intent (OVOS-policy)
hivemind-core blacklist-intent "HomeAssistant.DeviceControllerIntent" --node-id 2

# Un-blacklist an intent
hivemind-core allow-intent "HomeAssistant.DeviceControllerIntent" --node-id 2

# Grant admin flag
hivemind-core make-admin --node-id 2

# Revoke admin flag
hivemind-core revoke-admin --node-id 2

# Set arbitrary metadata (read by policy plugins)
hivemind-core set-metadata --node-id 2 --key role --value guest
```

### Per-client defaults

When a new client is added via `add-client`, it starts with a minimal default set of allowed message types. You must explicitly `allow-msg` any additional message types the client needs. This deny-by-default posture ensures that compromised credentials give an attacker minimal capability.

## Transport security (TLS)

HiveMind can use TLS for the WebSocket transport:

```bash
hivemind-core listen --ssl true --cert_dir /path/to/certs --cert_name mycert
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
- INTERCOM PGP authentication requires both nodes to have pre-exchanged public keys through a trusted channel.
