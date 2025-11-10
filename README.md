# Highly Available PostgreSQL Cluster in Docker Swarm

This repository provides a highly available PostgreSQL cluster setup using Docker Swarm, PostgreSQL with repmgr, and PgPool. The setup includes automatic failover, connection pooling, and supports both Traefik and HAProxy as load balancer options.

## Requirements

- Docker Engine 18.06.0+ (with Swarm mode enabled)
- Docker Compose 1.24.0+

## Deployment

1. Clone this repository:

```bash
git clone https://github.com/Romaxa55/postgres-docker-swarm-ha.git
```

2. Update the `docker-compose.yml` file with the appropriate environment variables, such as PostgreSQL credentials and etcd cluster configuration.

3. Create the Docker Swarm overlay networks:

```bash
docker network create --driver=overlay --attachable --subnet=192.168.0.1/24 postgres-network
```


4. Deploy the PostgreSQL cluster using Docker Stack:

```bash
docker stack deploy --compose-file docker-compose.yml etcd-stack

docker stack deploy -c docker-compose.yml postgres-ha
```


## Components

- **PostgreSQL with repmgr**: Primary and replica instances with automatic replication and failover.
- **PgPool**: Connection pooling and load balancing for PostgreSQL connections.
- **etcd**: Distributed key-value store for cluster coordination and configuration.
- **Traefik** (default) or **HAProxy**: Reverse proxy and load balancer options.
- **GlusterFS**: Distributed file system for shared storage across nodes.

## Load Balancer Options

This project supports two load balancer options:

1. **Traefik** (Default) - Modern, cloud-native edge router with automatic service discovery
   - Supports Traefik v2.x and v3.x
   - Automatic SSL with Let's Encrypt
   - HTTP/3 support (v3.0+)
   - Configuration via Docker labels
   
2. **HAProxy** - High-performance traditional load balancer
   - Superior performance and efficiency
   - Manual configuration
   - Fine-grained control

**To use Traefik** (default):
```bash
docker stack deploy -c docker-compose.yml ghost-cms
```

**To use HAProxy** (alternative):
```bash
docker stack deploy -c docker-compose.haproxy.yml ghost-cms
```

See [documents/LOAD_BALANCER_OPTIONS.md](documents/LOAD_BALANCER_OPTIONS.md) for detailed comparison and configuration.

## Documentation

Comprehensive documentation is available in the `documents/` folder:

- **[ARCHITECTURE.md](documents/ARCHITECTURE.md)** - System architecture and design
- **[DEPLOYMENT_GUIDE.md](documents/DEPLOYMENT_GUIDE.md)** - Detailed deployment instructions
- **[LOAD_BALANCER_OPTIONS.md](documents/LOAD_BALANCER_OPTIONS.md)** - Traefik vs HAProxy comparison
- **[CONTRIBUTING.md](documents/CONTRIBUTING.md)** - Contribution guidelines
- **[COPILOT_INSTRUCTIONS.md](documents/COPILOT_INSTRUCTIONS.md)** - GitHub Copilot assistance guide

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
