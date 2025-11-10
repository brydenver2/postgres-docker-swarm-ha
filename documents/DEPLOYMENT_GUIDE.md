# Deployment Guide

This guide provides detailed instructions for deploying the highly available PostgreSQL cluster on Docker Swarm.

## Prerequisites

### System Requirements

- **Minimum**: 3 nodes (physical or virtual machines)
- **OS**: Debian/Ubuntu Linux (tested on Debian 10+)
- **RAM**: 2GB minimum per node (4GB+ recommended)
- **Disk**: 20GB minimum per node
- **Network**: All nodes must be able to communicate with each other

### Software Requirements

- Docker Engine 18.06.0+
- Docker Compose 1.24.0+
- Ansible 2.9+ (for automated provisioning)
- SSH access to all nodes
- Python 3.6+ on control machine

## Deployment Options

This project supports two deployment approaches:

1. **Automated Deployment with Ansible** (Recommended)
2. **Manual Deployment**

## Option 1: Automated Deployment with Ansible

### Step 1: Prepare the Control Machine

Install Ansible on your control machine:

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install ansible

# macOS
brew install ansible

# Or via pip
pip install ansible
```

### Step 2: Configure Inventory

Edit `hosts.yaml` with your server details:

```yaml
all:
  children:
    docker_engine:
      hosts:
        swarm-01:
          ansible_host: 192.168.1.10  # Your server IP
          ansible_port: 22
          ansible_user: your_user
        swarm-02:
          ansible_host: 192.168.1.11
          ansible_port: 22
          ansible_user: your_user
        swarm-03:
          ansible_host: 192.168.1.12
          ansible_port: 22
          ansible_user: your_user
    docker_swarm_manager:
      hosts:
        swarm-01:
          swarm_labels: deploy
    docker_swarm_worker:
      hosts:
        swarm-02:
          swarm_labels: '["docker"]'
        swarm-03:
          swarm_labels: '["docker"]'
```

### Step 3: Configure Environment Variables

Edit `.env_postgres` with your desired credentials:

```bash
POSTGRESQL_POSTGRES_PASSWORD=your_secure_password
POSTGRESQL_USERNAME=ghost_cms
POSTGRESQL_PASSWORD=your_app_password
POSTGRESQL_DATABASE=ghost_cms
REPMGR_PASSWORD=your_repmgr_password
# ... other variables
```

**Security Note**: Use strong, unique passwords in production!

### Step 4: Run Ansible Playbook

Provision the infrastructure:

```bash
ansible-playbook -i hosts.yaml playbook.yml
```

This playbook will:
- Bootstrap Debian systems
- Install Docker and Docker Compose
- Initialize Docker Swarm
- Configure security settings
- Setup GlusterFS for distributed storage

### Step 5: Deploy the Stack

After Ansible provisioning completes:

```bash
# SSH to the manager node
ssh your_user@swarm-01

# Create the overlay network
docker network create --driver=overlay --attachable --subnet=192.168.0.1/24 postgres-network

# Deploy the stack
docker stack deploy --compose-file docker-compose.yml ghost-cms
```

## Option 2: Manual Deployment

### Step 1: Prepare Servers

On each node:

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add user to docker group
sudo usermod -aG docker $USER

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

### Step 2: Initialize Docker Swarm

On the manager node (swarm-01):

```bash
docker swarm init --advertise-addr <manager-ip>
```

Save the join token output. On worker nodes (swarm-02, swarm-03):

```bash
docker swarm join --token <worker-token> <manager-ip>:2377
```

### Step 3: Label Nodes

On the manager node:

```bash
docker node update --label-add node.hostname=swarm-01 swarm-01
docker node update --label-add node.hostname=swarm-02 swarm-02
docker node update --label-add node.hostname=swarm-03 swarm-03
```

### Step 4: Setup GlusterFS (Optional but Recommended)

On all nodes:

```bash
sudo apt install glusterfs-server -y
sudo systemctl enable glusterd
sudo systemctl start glusterd
```

On the manager node:

```bash
# Create peer connections
sudo gluster peer probe swarm-02
sudo gluster peer probe swarm-03

# Create volume
sudo mkdir -p /mnt/gluster-storage
sudo gluster volume create gv0 replica 3 \
  swarm-01:/mnt/gluster-storage \
  swarm-02:/mnt/gluster-storage \
  swarm-03:/mnt/gluster-storage

sudo gluster volume start gv0
```

On all nodes:

```bash
# Mount GlusterFS
sudo mkdir -p /mnt/glusterfs
sudo mount -t glusterfs localhost:/gv0 /mnt/glusterfs

# Add to fstab for persistence
echo "localhost:/gv0 /mnt/glusterfs glusterfs defaults,_netdev 0 0" | sudo tee -a /etc/fstab
```

### Step 5: Create Network

On the manager node:

```bash
docker network create --driver=overlay --attachable --subnet=192.168.0.1/24 postgres-network
```

### Step 6: Deploy Services

Clone the repository on the manager node:

```bash
git clone https://github.com/YOUR_USERNAME/postgres-docker-swarm-ha.git
cd postgres-docker-swarm-ha
```

Configure `.env_postgres` with your credentials, then deploy:

```bash
docker stack deploy -c docker-compose.yml ghost-cms
```

## Verification

### Check Service Status

```bash
# List all services
docker service ls

# Check specific service status
docker service ps ghost-cms_pg-1
docker service ps ghost-cms_pgpool

# View service logs
docker service logs ghost-cms_pg-1
docker service logs ghost-cms_pgpool
```

### Verify PostgreSQL Cluster

```bash
# Check replication status
docker exec -it $(docker ps -q -f name=pg-1) psql -U postgres -c "SELECT * FROM pg_stat_replication;"

# Test connection through PgPool
docker exec -it $(docker ps -q -f name=pgpool) psql -U ghost_cms -d ghost_cms -h localhost -c "SELECT version();"
```

### Verify etcd Cluster

```bash
# Check etcd cluster health
docker exec -it $(docker ps -q -f name=etcd1) etcdctl endpoint health
```

### Test Traefik/Load Balancer

```bash
# Check if Ghost is accessible
curl -I http://your-domain.com

# Or check HTTPS
curl -I https://your-domain.com
```

## Post-Deployment Configuration

### Configure DNS

Point your domain to the manager node IP:

```
A    your-domain.com    -> manager-node-ip
CNAME www.your-domain.com -> your-domain.com
```

### Configure CloudFlare (if using Traefik with DNS challenge)

1. Create a CloudFlare API token
2. Add token to docker-compose.yml or as Docker secret
3. Redeploy Traefik service

### Backup Configuration

Setup regular backups:

```bash
# PostgreSQL backup
docker exec -it $(docker ps -q -f name=pg-1) pg_dump -U postgres ghost_cms > backup.sql

# GlusterFS snapshots (if configured)
sudo gluster snapshot create snap1 gv0
```

## Scaling

### Scale Ghost CMS

```bash
docker service scale ghost-cms_ghost=10
```

### Scale PgPool

```bash
docker service scale ghost-cms_pgpool=3
```

### Add PostgreSQL Replicas

Edit `docker-compose.yml` to add pg-4, pg-5, etc., then:

```bash
docker stack deploy -c docker-compose.yml ghost-cms
```

## Choosing Load Balancer: Traefik vs HAProxy

See [LOAD_BALANCER_OPTIONS.md](LOAD_BALANCER_OPTIONS.md) for detailed comparison and configuration.

### Using Traefik (Default)

Already configured in `docker-compose.yml`.

### Using HAProxy

Use the alternative configuration file:

```bash
docker stack deploy -c docker-compose.haproxy.yml ghost-cms
```

## Troubleshooting

### Services Not Starting

```bash
# Check service errors
docker service ps --no-trunc <service-name>

# Check container logs
docker logs <container-id>
```

### Network Issues

```bash
# Verify network exists
docker network ls | grep postgres-network

# Inspect network
docker network inspect postgres-network
```

### PostgreSQL Replication Issues

```bash
# Check repmgr status
docker exec -it $(docker ps -q -f name=pg-1) repmgr cluster show

# Check PostgreSQL logs
docker service logs ghost-cms_pg-1
```

### Traefik Not Routing

```bash
# Check Traefik logs
docker service logs ghost-cms_traefik

# Verify labels on services
docker service inspect ghost-cms_ghost
```

### GlusterFS Issues

```bash
# Check volume status
sudo gluster volume status gv0

# Check peer status
sudo gluster peer status

# Check mount
df -h | grep glusterfs
```

## Maintenance

### Updating Services

```bash
# Update specific service
docker service update --image bitnami/postgresql-repmgr:15 ghost-cms_pg-1

# Or update entire stack
docker stack deploy -c docker-compose.yml ghost-cms
```

### Backup and Restore

#### Backup PostgreSQL

```bash
# Backup database
docker exec $(docker ps -q -f name=pg-1) pg_dump -U postgres ghost_cms | gzip > backup-$(date +%Y%m%d).sql.gz

# Backup globals
docker exec $(docker ps -q -f name=pg-1) pg_dumpall -U postgres -g | gzip > globals-$(date +%Y%m%d).sql.gz
```

#### Restore PostgreSQL

```bash
# Restore database
gunzip < backup-20231110.sql.gz | docker exec -i $(docker ps -q -f name=pg-1) psql -U postgres ghost_cms
```

### Monitoring

Consider adding monitoring stack:

```bash
# Add Prometheus, Grafana, and exporters
docker stack deploy -c monitoring-compose.yml monitoring
```

## Security Hardening

### Production Checklist

- [ ] Use Docker secrets instead of environment files
- [ ] Enable Docker Content Trust
- [ ] Implement network policies
- [ ] Use specific image versions (not `latest`)
- [ ] Regular security updates
- [ ] Backup encryption
- [ ] SSL/TLS for all connections
- [ ] Firewall configuration
- [ ] Audit logging
- [ ] Limited SSH access

### Using Docker Secrets

```bash
# Create secrets
echo "your_password" | docker secret create postgres_password -

# Update docker-compose.yml to use secrets
secrets:
  - postgres_password

# Reference in service
environment:
  - POSTGRESQL_PASSWORD_FILE=/run/secrets/postgres_password
```

## Advanced Configuration

### Custom PostgreSQL Configuration

Create a custom PostgreSQL config and mount it:

```yaml
volumes:
  - ./postgresql.conf:/bitnami/postgresql/conf/postgresql.conf:ro
```

### Custom PgPool Configuration

```yaml
volumes:
  - ./pgpool.conf:/opt/bitnami/pgpool/etc/pgpool.conf:ro
```

### Resource Limits

```yaml
deploy:
  resources:
    limits:
      cpus: '2'
      memory: 4G
    reservations:
      cpus: '1'
      memory: 2G
```

## Support

If you encounter issues:

1. Check the [troubleshooting section](#troubleshooting)
2. Review service logs
3. Consult the [ARCHITECTURE.md](ARCHITECTURE.md) for system design
4. Open an issue on GitHub with detailed information

## Next Steps

- Configure monitoring and alerting
- Setup automated backups
- Implement disaster recovery procedures
- Tune performance based on workload
- Add additional applications to the cluster
