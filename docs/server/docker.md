# Docker Deployment

**`hivemind-core` runs in Docker for easy deployment and isolation.**

!!! abstract "In a nutshell"
    - There is no published all-in-one image; containers are built locally from the `hivemind-skills-server-docker` compose stack or based on the `smartgic/*` images.
    - `hivemind-core` reads no environment variables — all configuration is the `server.json` file mounted into the non-root `hivemind` user's config directory.
    - The stack uses `network_mode: host`; SQLite is the default database, with Redis for multi-instance deployments.

There is no published "all-in-one" hivemind-core image on the JarbasHiveMind
namespace. Containers are either:

- **built locally** from the
  [`hivemind-skills-server-docker`](https://github.com/JarbasHiveMind/hivemind-skills-server-docker)
  compose stack (`Dockerfile.hivemind` → local image `ovos/hivemind-server`,
  built `FROM debian:trixie-slim` with a non-root `hivemind` user), or
- **based on the `smartgic/*` images** published from
  [`hivemind-docker`](https://github.com/JarbasHiveMind/hivemind-docker)
  (default registry `docker.io/smartgic`, e.g. `smartgic/hivemind-base`,
  `smartgic/hivemind-listener`).

The examples below follow the `hivemind-skills-server-docker` stack, which builds
its hivemind image locally.

---

## How config works in the container

`hivemind-core` reads **no environment variables**. All configuration is the
`server.json` file described in [OVOS Skills Server](ovos-server.md#configuration). In a
container you supply it by mounting a host folder onto the config directory of the
non-root `hivemind` user:

```
/home/hivemind/.config/hivemind-core   # server.json lives here
/home/hivemind/.config/hivemind        # identity.json
/home/hivemind/.local/share/hivemind   # data dir (clients db, certs)
```

The client database, certificates, etc. live under
`/home/hivemind/.local/share/hivemind` — not `/root/...`, because the image runs as
the unprivileged `hivemind` user.

---

## Basic deployment

### docker-compose.yml

```yaml
services:
  hivemind_core:
    build:
      context: .
      dockerfile: Dockerfile.hivemind
    image: ovos/hivemind-server
    container_name: hivemind_core
    hostname: hivemind_core
    restart: always
    network_mode: host
    volumes:
      # mount your server.json / identity / data into the hivemind user's home
      - ${HIVEMIND_CONFIG_FOLDER}:/home/hivemind/.config/hivemind:z
      - ${HIVEMIND_SERVER_CONFIG_FOLDER}:/home/hivemind/.config/hivemind-core:z
      - ${HIVEMIND_SHARE_FOLDER}:/home/hivemind/.local/share/hivemind:z
    depends_on:
      hivemind_redis:
        condition: service_started
```

`HIVEMIND_SERVER_CONFIG_FOLDER` is a host directory holding your `server.json`;
edit that file to set hosts, ports, TLS, the agent protocol, and the database
backend.

!!! note "Where do the `${...}` values come from?"
    The compose file reads `${HIVEMIND_CONFIG_FOLDER}`, `${HIVEMIND_REDIS_PORT}`,
    and friends from a `.env` file you create next to `docker-compose.yml`. The
    stack ships an example `.env` directly (there is no separate `.env.example`) —
    copy it, fill in your host paths and ports, then `docker compose up`.

Start:

```bash
docker compose up -d --build
```

View logs:

```bash
docker compose logs -f hivemind_core
```

> The stack uses `network_mode: host`, so the WebSocket (5678) and HTTP (5679)
> listeners are reachable directly on the host; published `ports:` are cosmetic
> under host networking.

---

## Redis backend

Redis is a real, supported client-database backend
([hivemind-redis-database](https://github.com/JarbasHiveMind/hivemind-redis-database)).
It is **not** selected by an environment variable — set it in the `database` block of
`server.json`:

```json
{
  "database": {
    "module": "hivemind-redis-db-plugin",
    "hivemind-redis-db-plugin": {
      "host": "127.0.0.1",
      "port": 6379
    }
  }
}
```

and run a Redis container alongside hivemind-core:

```yaml
services:
  hivemind_redis:
    image: redis
    container_name: hivemind_redis
    restart: always
    ports:
      - "${HIVEMIND_REDIS_PORT}:6379"
    volumes:
      - ${HIVEMIND_SERVER_CONFIG_FOLDER}/redis.conf:/usr/local/etc/redis/redis.conf
      - ${DATA_BASE_DIR}/hivemind/redis:/data

  hivemind_core:
    build:
      context: .
      dockerfile: Dockerfile.hivemind
    image: ovos/hivemind-server
    container_name: hivemind_core
    restart: always
    network_mode: host
    volumes:
      - ${HIVEMIND_CONFIG_FOLDER}:/home/hivemind/.config/hivemind:z
      - ${HIVEMIND_SERVER_CONFIG_FOLDER}:/home/hivemind/.config/hivemind-core:z
      - ${HIVEMIND_SHARE_FOLDER}:/home/hivemind/.local/share/hivemind:z
    depends_on:
      hivemind_redis:
        condition: service_started
```

The default backend is SQLite (stdlib, transactional); switch to Redis only if you
need it for a multi-instance / shared-store deployment.

---

## Persona service

The same stack can run a [persona server](persona-server.md) container (`hivemind_persona`,
built from `Dockerfile.persona`, which is `FROM smartgic/hivemind-base`) that exposes
HiveMind to OpenAI/Ollama-compatible apps. Its `VOICE_SAT_*` env vars are *client
credentials* it uses to connect back to `hivemind_core`; they must first be provisioned
with `hivemind-core add-client`.

---

## With SSL via reverse proxy

For production you can terminate TLS at a reverse proxy (nginx, Caddy) in front of
`hivemind-core`, or enable TLS natively per network protocol in `server.json`
(`network_protocol.<plugin>.ssl = true`, plus `cert_dir`/`cert_name`).

When proxying, don't also expose 5678/5679 directly to the internet.

---

## Management commands

Run the CLI inside the container. Use the real service name and a **positional**
node ID:

```bash
# Add a client
docker compose exec hivemind_core hivemind-core add-client --name "living-room"

# List clients
docker compose exec hivemind_core hivemind-core list-clients

# Remove client with node ID 2
docker compose exec hivemind_core hivemind-core delete-client 2

# Allow a message type for node ID 2
docker compose exec hivemind_core hivemind-core allow-msg "speak" 2

# Blacklist a skill for node ID 2 (requires OVOSAgentPolicy in policy.chain)
docker compose exec hivemind_core hivemind-core blacklist-skill "skill-homeassistant.openvoiceos" 2
```

---

## Troubleshooting

**Satellite cannot reach `hivemind-core`**: with `network_mode: host`, check the host's port
5678 (and 5679) is reachable from the satellite network and not firewalled.

**Cannot connect to Redis**: verify Redis is running
(`docker compose exec hivemind_redis redis-cli ping`) and that the `database` block in
`server.json` points at the right host/port.

**Config changes ignored**: confirm your `server.json` is on the host folder mounted at
`/home/hivemind/.config/hivemind-core`, and restart the container — the persona file
and most config are read at startup only.

---

## Source

Validated against the HiveMind source:

- [`docker-compose.yml`](https://github.com/JarbasHiveMind/hivemind-skills-server-docker/blob/HEAD/docker-compose.yml) — the compose stack, mounts, `network_mode: host`, and Redis service
- [`Dockerfile.hivemind`](https://github.com/JarbasHiveMind/hivemind-skills-server-docker/blob/HEAD/Dockerfile.hivemind) — `FROM debian:trixie-slim`, non-root `hivemind` user, `ovos/hivemind-server` image
- [`Dockerfile.persona`](https://github.com/JarbasHiveMind/hivemind-skills-server-docker/blob/HEAD/Dockerfile.persona) — `FROM smartgic/hivemind-base`, the persona-service container
- [`docker-bake.hcl`](https://github.com/JarbasHiveMind/hivemind-docker/blob/HEAD/docker-bake.hcl) — the published `smartgic/*` image build targets
