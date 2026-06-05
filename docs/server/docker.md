# Docker Deployment

Run `hivemind-core` in Docker for easy deployment and isolation.

## Prerequisites

- Docker and Docker Compose installed
- The hub network accessible from satellite devices

## Basic deployment

### docker-compose.yml

```yaml
version: '3.8'

services:
  hivemind:
    image: jarbashivemind/hivemind-core:latest
    ports:
      - "5678:5678"
    environment:
      - HIVEMIND_HOST=0.0.0.0
      - HIVEMIND_PORT=5678
      - HIVEMIND_DB_BACKEND=sqlite
    volumes:
      - hivemind_data:/root/.local/share/hivemind-core
    restart: unless-stopped
    hostname: hivemind-hub

volumes:
  hivemind_data:
```

Start:

```bash
docker-compose up -d
```

Add a client:

```bash
docker-compose exec hivemind hivemind-core add-client --name "my-satellite"
```

View logs:

```bash
docker-compose logs -f hivemind
```

## With SSL via reverse proxy

For production deployments, run a reverse proxy (nginx, Caddy) in front of HiveMind:

```yaml
version: '3.8'

services:
  hivemind:
    image: jarbashivemind/hivemind-core:latest
    environment:
      - HIVEMIND_HOST=localhost
      - HIVEMIND_PORT=5678
      - HIVEMIND_DB_BACKEND=sqlite
    volumes:
      - hivemind_data:/root/.local/share/hivemind-core
    networks:
      - hivemind_network
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "443:443"
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certs:/etc/nginx/certs:ro
    depends_on:
      - hivemind
    networks:
      - hivemind_network
    restart: unless-stopped

volumes:
  hivemind_data:

networks:
  hivemind_network:
```

Do not expose port 5678 directly to the internet when using a reverse proxy.

## With Redis backend

For multi-instance or high-availability deployments:

```yaml
version: '3.8'

services:
  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    networks:
      - hivemind_network
    restart: unless-stopped

  hivemind:
    image: jarbashivemind/hivemind-core:latest
    ports:
      - "5678:5678"
    environment:
      - HIVEMIND_HOST=0.0.0.0
      - HIVEMIND_PORT=5678
      - HIVEMIND_DB_BACKEND=redis
      - REDIS_HOST=redis
      - REDIS_PORT=6379
    depends_on:
      - redis
    networks:
      - hivemind_network
    restart: unless-stopped

volumes:
  redis_data:

networks:
  hivemind_network:
```

## Management commands

```bash
# Add a client
docker-compose exec hivemind hivemind-core add-client --name "living-room"

# List clients
docker-compose exec hivemind hivemind-core list-clients

# Remove a client
docker-compose exec hivemind hivemind-core delete-client --node-id 2

# Allow a message type
docker-compose exec hivemind hivemind-core allow-msg "speak" --node-id 2

# Blacklist a skill
docker-compose exec hivemind hivemind-core blacklist-skill "skill-homeassistant.openvoiceos" --node-id 2
```

## Resource limits

For production containers:

```yaml
services:
  hivemind:
    ...
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 2G
```

## Troubleshooting

**Satellite cannot reach the hub**: check that the Docker host's port 5678 is reachable from the satellite network and that no firewall is blocking it.

**Cannot connect to Redis**: verify Redis is running (`docker-compose exec redis redis-cli ping`) and that both containers share the same network.

**High memory usage**: check `docker stats`. Consider switching from JSON to SQLite or Redis for better performance under load.
