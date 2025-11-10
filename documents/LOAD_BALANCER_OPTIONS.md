# Load Balancer Options: Traefik vs HAProxy

This document explains the two load balancer options available for this PostgreSQL Docker Swarm HA setup.

## Overview

The project supports two reverse proxy and load balancer options:

1. **Traefik** (Default) - Modern, dynamic, cloud-native edge router
2. **HAProxy** - Traditional, high-performance load balancer

Both options can handle traffic routing, SSL termination, and load balancing. Choose based on your requirements and preferences.

## Comparison Table

| Feature | Traefik | HAProxy |
|---------|---------|---------|
| **Auto-discovery** | ✅ Yes (Docker Swarm labels) | ❌ No (static config) |
| **Configuration** | Labels in docker-compose | Config file |
| **SSL Automation** | ✅ Let's Encrypt built-in | Manual cert management |
| **API Gateway** | ✅ Yes | Limited |
| **Performance** | Very Good | Excellent |
| **Learning Curve** | Easy | Moderate |
| **Docker Integration** | Native | Requires manual config |
| **Version Support** | v2.x and v3.x | Latest stable |
| **HTTP/3 (QUIC)** | v3.0+ only | Requires HAProxy 2.6+ |
| **UI Dashboard** | ✅ Yes | Via stats page |
| **Middleware Support** | ✅ Rich (rate limit, auth, etc.) | Via config |
| **Configuration Reload** | Automatic | Requires reload |

## Option 1: Traefik (Default)

### Features

- **Automatic Service Discovery**: Discovers services via Docker Swarm labels
- **Let's Encrypt Integration**: Automatic SSL certificate issuance and renewal
- **Dynamic Configuration**: No restart needed for new services
- **Middleware**: Built-in support for authentication, rate limiting, compression, etc.
- **Multiple Protocols**: HTTP, HTTPS, TCP, UDP
- **Dashboard**: Web UI for monitoring

### Versions

#### Traefik v2.x
- Stable and widely used
- Good documentation and community support
- All features needed for this project

#### Traefik v3.x
- Latest version with modern features
- HTTP/3 (QUIC) support
- Improved performance
- WebAssembly plugin support
- Backward compatible with v2 configuration (mostly)

### Configuration

Traefik is configured via Docker labels in `docker-compose.yml`:

```yaml
traefik:
  image: traefik:v3.0  # or traefik:v2.10 or traefik:latest
  command:
    - --providers.docker=true
    - --providers.docker.swarmMode=true
    - --providers.docker.endpoint=unix:///var/run/docker.sock
    - --entrypoints.web.address=:80
    - --entrypoints.websecure.address=:443
    - --certificatesresolvers.myresolver.acme.dnschallenge=true
    - --certificatesresolvers.myresolver.acme.dnschallenge.provider=cloudflare
    - --certificatesresolvers.myresolver.acme.email=your-email@example.com
    - --certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json
  environment:
    - CF_DNS_API_TOKEN=${CF_API_KEY}
  ports:
    - 80:80
    - 443:443
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock:ro
    - /mnt/glusterfs/letsencrypt:/letsencrypt
  networks:
    - postgres-network
  deploy:
    labels:
      - traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https
    placement:
      constraints:
        - node.role == manager
```

### Service Configuration

Configure services to be discovered by Traefik:

```yaml
ghost:
  image: ghost:latest
  deploy:
    labels:
      - traefik.enable=true
      - traefik.http.routers.ghost.rule=Host(`yourdomain.com`)
      - traefik.http.routers.ghost.entrypoints=web
      - traefik.http.routers.ghost.middlewares=redirect-to-https
      - traefik.http.routers.ghost-secured.rule=Host(`yourdomain.com`)
      - traefik.http.routers.ghost-secured.entrypoints=websecure
      - traefik.http.routers.ghost-secured.tls.certresolver=myresolver
      - traefik.http.services.ghost.loadbalancer.server.port=2368
  networks:
    - postgres-network
```

### Traefik v3 Specific Configuration

For Traefik v3, you can enable additional features:

```yaml
traefik:
  image: traefik:v3.0
  command:
    # ... existing commands ...
    - --experimental.http3=true  # Enable HTTP/3
    - --entrypoints.websecure.http3=true
    - --metrics.prometheus=true  # Enable Prometheus metrics
    - --api.dashboard=true  # Enable dashboard
```

### Accessing the Dashboard

Enable the Traefik dashboard (development only):

```yaml
traefik:
  command:
    - --api.dashboard=true
    - --api.insecure=true  # Only for development!
  ports:
    - 8080:8080  # Dashboard port
```

Access at: http://your-server:8080/dashboard/

### Pros
- ✅ Easy configuration via labels
- ✅ Automatic SSL certificates
- ✅ No manual updates needed for new services
- ✅ Modern cloud-native approach
- ✅ Great for microservices

### Cons
- ❌ More resource usage than HAProxy
- ❌ Requires Docker socket access
- ❌ Learning curve for advanced features

## Option 2: HAProxy

### Features

- **High Performance**: Extremely efficient, minimal resource usage
- **Battle-Tested**: Used by many high-traffic websites
- **Flexible**: Supports complex routing and load balancing scenarios
- **TCP/HTTP Support**: Can proxy any TCP protocol
- **Stats Page**: Built-in statistics dashboard
- **Health Checks**: Advanced health checking capabilities

### Configuration

HAProxy requires a configuration file (`haproxy.cfg`):

```haproxy
global
    log /dev/log local0
    log /dev/log local1 notice
    maxconn 4096
    daemon

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000

frontend http_front
    bind *:80
    redirect scheme https code 301 if !{ ssl_fc }

frontend https_front
    bind *:443 ssl crt /etc/ssl/certs/yourdomain.pem
    
    # Route based on host header
    acl host_ghost hdr(host) -i yourdomain.com
    use_backend ghost_backend if host_ghost

backend ghost_backend
    balance roundrobin
    option httpchk GET /
    server ghost1 ghost:2368 check
    server ghost2 ghost:2368 check
    server ghost3 ghost:2368 check

listen stats
    bind *:8404
    stats enable
    stats uri /
    stats refresh 30s
```

### Docker Compose Configuration

Use the alternative configuration file `docker-compose.haproxy.yml`:

```yaml
version: '3.8'

services:
  # ... PostgreSQL, etcd, pgpool services (same as before) ...

  haproxy:
    image: haproxy:latest
    ports:
      - 80:80
      - 443:443
      - 8404:8404  # Stats page
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
      - /mnt/glusterfs/ssl:/etc/ssl/certs:ro
    networks:
      - postgres-network
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager

  ghost:
    image: ghost:latest
    # No Traefik labels needed
    networks:
      - postgres-network
```

### SSL Certificate Management

With HAProxy, you need to manage SSL certificates manually:

```bash
# Obtain certificate with certbot
certbot certonly --standalone -d yourdomain.com

# Combine cert and key for HAProxy
cat /etc/letsencrypt/live/yourdomain.com/fullchain.pem \
    /etc/letsencrypt/live/yourdomain.com/privkey.pem \
    > /mnt/glusterfs/ssl/yourdomain.pem

# Setup auto-renewal
echo "0 0 * * 0 certbot renew --post-hook 'cat /etc/letsencrypt/live/yourdomain.com/fullchain.pem /etc/letsencrypt/live/yourdomain.com/privkey.pem > /mnt/glusterfs/ssl/yourdomain.pem && docker service update --force ghost-cms_haproxy'" | crontab -
```

### Stats Dashboard

Access HAProxy stats at: http://your-server:8404/

### Pros
- ✅ Superior performance and efficiency
- ✅ Lower resource usage
- ✅ More predictable behavior
- ✅ Fine-grained control
- ✅ No Docker socket access needed

### Cons
- ❌ Manual configuration required
- ❌ Manual SSL certificate management
- ❌ Requires reload for configuration changes
- ❌ More complex initial setup

## Switching Between Options

### From Traefik to HAProxy

1. Create `haproxy.cfg` configuration file
2. Obtain and configure SSL certificates
3. Stop the current stack:
   ```bash
   docker stack rm ghost-cms
   ```
4. Deploy with HAProxy configuration:
   ```bash
   docker stack deploy -c docker-compose.haproxy.yml ghost-cms
   ```

### From HAProxy to Traefik

1. Stop the current stack:
   ```bash
   docker stack rm ghost-cms
   ```
2. Deploy with Traefik configuration (default):
   ```bash
   docker stack deploy -c docker-compose.yml ghost-cms
   ```
3. Update DNS to point to the Traefik node

## Performance Considerations

### Traefik
- Good for: Dynamic environments, microservices, automatic SSL
- Resource usage: ~100-200MB RAM
- Best when: You need automatic service discovery

### HAProxy
- Good for: High-traffic sites, static configurations, maximum performance
- Resource usage: ~20-50MB RAM
- Best when: You need maximum performance and control

## When to Choose Traefik

Choose Traefik if you:
- Want automatic service discovery
- Need Let's Encrypt automation
- Prefer configuration via labels
- Are building a microservices architecture
- Want a modern, cloud-native solution
- Need quick deployments without configuration files
- Want HTTP/3 support (v3.0+)

## When to Choose HAProxy

Choose HAProxy if you:
- Need maximum performance
- Have static service configurations
- Want fine-grained control over load balancing
- Already have SSL certificate infrastructure
- Need minimal resource usage
- Require specific TCP-level routing
- Have complex routing requirements

## Recommended Approach

For this project, we recommend:

### Development/Testing: Traefik
- Easier to set up and test
- Automatic SSL for testing domains
- Quick iteration

### Production: Choose Based on Requirements

**Use Traefik if:**
- Services change frequently
- You want operational simplicity
- Automatic SSL is important
- You need modern features like HTTP/3

**Use HAProxy if:**
- You need maximum performance
- You have dedicated DevOps team
- SSL certificates are managed separately
- You have complex routing requirements

## Hybrid Approach

You can also use both:
- **HAProxy** for primary traffic (public websites)
- **Traefik** for internal services and APIs

This requires separate networks and careful DNS configuration.

## Migration Path

1. **Start with Traefik** (default in this project)
2. **Monitor** performance and operational overhead
3. **Switch to HAProxy** if performance becomes critical
4. **Keep both configurations** in the repository for flexibility

## Files in This Project

- `docker-compose.yml` - Default configuration with Traefik
- `docker-compose.haproxy.yml` - Alternative configuration with HAProxy
- `haproxy.cfg` - Sample HAProxy configuration
- `documents/LOAD_BALANCER_OPTIONS.md` - This document

## Additional Resources

### Traefik
- Official docs: https://doc.traefik.io/traefik/
- v3 migration: https://doc.traefik.io/traefik/v3.0/migration/v2-to-v3/
- Docker Swarm guide: https://doc.traefik.io/traefik/providers/docker/

### HAProxy
- Official docs: https://www.haproxy.org/
- Configuration manual: https://cbonte.github.io/haproxy-dconv/
- Docker image: https://hub.docker.com/_/haproxy

## Conclusion

Both Traefik and HAProxy are excellent choices. The default Traefik configuration provides the best developer experience and is suitable for most use cases. Consider HAProxy if you need maximum performance or have specific requirements that Traefik doesn't meet.
