# Docker Deployment

Run HiveMind in Docker for easy deployment and isolation.

## Prerequisites

- Docker and Docker Compose installed
- A network accessible from satellite devices

## Quick Start with Docker Compose

### 1. Create a `docker-compose.yml`

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
    # Optional: Set a fixed hostname for discovery
    hostname: hivemind-hub

volumes:
  hivemind_data:
```

### 2. Start the container

```bash
docker-compose up -d
```

### 3. Add a client

```bash
docker-compose exec hivemind hivemind-core add-client --name "my-satellite"
```

The output will show the credentials needed for satellite setup.

### 4. View logs

```bash
docker-compose logs -f hivemind
```

---

## Advanced Configuration

### With SSL/TLS (Reverse Proxy)

For production, use a reverse proxy (nginx, Caddy) in front of HiveMind:

```yaml
version: '3.8'

services:
  hivemind:
    image: jarbashivemind/hivemind-core:latest
    ports:
      - "5678:5678"
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

### With OVOS Audio Processing

To use the `hivemind-audio-binary-protocol` for STT/TTS offloading:

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
      - ./config:/root/.config/mycroft
    depends_on:
      - audio_processor
    networks:
      - hivemind_network
    restart: unless-stopped

  audio_processor:
    image: jarbashivemind/ovos-core:latest
    environment:
      - OVOS_PLUGINS=hivemind-audio-binary-protocol
    volumes:
      - ./config:/root/.config/mycroft
    networks:
      - hivemind_network
    restart: unless-stopped

volumes:
  hivemind_data:

networks:
  hivemind_network:
```

### With Redis Backend

For multi-instance deployments:

```yaml
version: '3.8'

services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
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

---

## Management

### Add a Client

```bash
docker-compose exec hivemind hivemind-core add-client --name "living-room"
```

### List Clients

```bash
docker-compose exec hivemind hivemind-core list-clients
```

### Delete a Client

```bash
docker-compose exec hivemind hivemind-core delete-client --node-id 2
```

### Set Permissions

```bash
# Allow a message type
docker-compose exec hivemind hivemind-core allow-msg "speak" --node-id 2

# Blacklist a skill
docker-compose exec hivemind hivemind-core blacklist-skill "skill-homeassistant.openvoiceos" --node-id 2
```

See [Permissions](16_permissions.md) for the full CLI reference.

---

## Security Considerations

### Network Isolation

By default, the Docker Compose setup binds to `0.0.0.0` (all interfaces). For home use on a private network, this is acceptable. For production or internet-facing deployments:

1. **Use a reverse proxy** with SSL/TLS (see above).
2. **Bind to localhost only** and use port forwarding if needed.
3. **Implement firewall rules** to restrict access to trusted networks.

### Database Security

- **SQLite**: The default database file is stored in a Docker volume. Ensure the volume has appropriate permissions.
- **Redis**: If using Redis, configure authentication and restrict network access.

See [Database Backends](17_database.md) for detailed security guidance.

---

## Troubleshooting

### HiveMind not accessible from satellite

1. Check that the satellite can reach the host:
   ```bash
   ping <docker_host_ip>
   ```

2. Verify the port is open:
   ```bash
   telnet <docker_host_ip> 5678
   ```

3. Check logs:
   ```bash
   docker-compose logs -f hivemind
   ```

### Cannot connect to Redis

1. Verify Redis is running:
   ```bash
   docker-compose exec redis redis-cli ping
   ```

2. Check network connectivity between services:
   ```bash
   docker-compose exec hivemind ping redis
   ```

### High memory usage

1. Check running containers:
   ```bash
   docker stats
   ```

2. Review logs for errors:
   ```bash
   docker-compose logs hivemind
   ```

3. Consider using SQLite or Redis instead of JSON backend for better performance.

---

## Production Deployment

For production use:

1. **Use a managed database** (Redis Cloud, etc.) instead of containerized Redis.
2. **Set up SSL/TLS** with a reverse proxy like nginx or Caddy.
3. **Enable access controls**: Restrict network access to trusted clients only.
4. **Monitor logs and metrics**: Use a logging stack (ELK, Loki) to track issues.
5. **Regular backups**: Back up the database volumes regularly.
6. **Resource limits**: Set CPU and memory limits on containers:
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

See the [Quick Start Guide](01_quickstart.md) for more information on HiveMind setup.
