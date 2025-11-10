# GitHub Copilot Instructions for PostgreSQL Docker Swarm HA Project

## Project Overview

This project provides a highly available PostgreSQL cluster setup using Docker Swarm with the following components:
- **PostgreSQL with repmgr**: For replication and failover management
- **PgPool**: Connection pooling and load balancing
- **etcd**: Distributed configuration store for cluster coordination
- **Traefik**: Reverse proxy and load balancer (default option)
- **HAProxy**: Alternative reverse proxy option
- **GlusterFS**: Distributed file system for shared storage
- **Ghost CMS**: Example application using the HA PostgreSQL cluster

## Architecture

The cluster consists of:
- 3 PostgreSQL nodes with automatic replication (pg-1, pg-2, pg-3)
- 3 etcd nodes for distributed consensus
- PgPool for connection pooling with 2 replicas
- Traefik or HAProxy for ingress traffic management
- GlusterFS for shared persistent storage

## Key Technologies

- **Docker Swarm**: Container orchestration
- **Bitnami PostgreSQL with repmgr**: PostgreSQL clustering solution
- **Ansible**: Infrastructure provisioning (playbook.yml)
- **Docker Compose v3.8**: Service definitions

## Code Patterns

### Docker Compose YAML Anchors
The project uses YAML anchors to reduce duplication:
```yaml
pg-1: &pg
  image: 'bitnami/postgresql-repmgr:latest'
  # ... shared configuration

pg-2:
  <<: *pg  # Inherits from &pg anchor
```

### Deployment Constraints
Services are pinned to specific nodes using placement constraints:
```yaml
deploy:
  placement:
    constraints:
      - node.hostname == swarm-01
```

### Traefik Labels
Traefik routing is configured via Docker labels:
```yaml
deploy:
  labels:
    - traefik.enable=true
    - traefik.http.routers.ghost.rule=Host(`example.com`)
```

## Development Guidelines

### Adding New Services
1. Add service definition in docker-compose.yml
2. Use appropriate network: `postgres-network`
3. Add deployment constraints if needed
4. Configure Traefik labels for external access
5. Update environment variables in .env_postgres if needed

### Modifying PostgreSQL Configuration
- Environment variables are in .env_postgres
- Node names and network names must match
- Update PGPOOL_BACKEND_NODES when adding/removing nodes

### Traefik Configuration
- Uses Docker Swarm provider
- Automatic SSL with Let's Encrypt
- CloudFlare DNS challenge for certificates
- Middleware for HTTPS redirect

## Important Files

- **docker-compose.yml**: Main service definitions
- **.env_postgres**: Database credentials and configuration
- **playbook.yml**: Ansible provisioning playbook
- **hosts.yaml**: Ansible inventory
- **start.sh**: Deployment script

## Testing Approach

This project is infrastructure-focused with no unit tests. Testing is done through:
1. Docker Compose validation: `docker-compose config`
2. Deployment testing in actual Docker Swarm
3. Database connectivity testing
4. Failover scenario testing

## Common Tasks

### Deploying the Stack
```bash
docker stack deploy --compose-file docker-compose.yml ghost-cms
```

### Checking Service Status
```bash
docker service ls
docker service ps <service-name>
```

### Viewing Logs
```bash
docker service logs <service-name>
```

### Scaling Services
```bash
docker service scale ghost-cms_pgpool=3
```

## Security Considerations

- Credentials are in .env_postgres (should be secured)
- SSH port is custom (15122) as defined in playbook.yml
- Traefik handles SSL/TLS termination
- etcd should be secured in production

## Limitations

- Node hostnames must match constraints in docker-compose.yml
- Requires exactly 3 nodes for current configuration
- GlusterFS mount path hardcoded to /mnt/glusterfs
- CloudFlare DNS provider hardcoded for Let's Encrypt

## Tips for AI Assistants

- Always maintain YAML anchor pattern when modifying services
- Preserve deployment constraints and network configurations
- When adding Traefik alternatives, maintain backward compatibility
- Consider both Traefik v2 and v3+ syntax differences
- Maintain consistency in service naming conventions (pg-1, pg-2, etc.)
- Environment variable changes must be reflected in both docker-compose.yml and .env_postgres
