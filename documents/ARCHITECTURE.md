# Project Architecture

## Overview

This project implements a highly available (HA) PostgreSQL cluster using Docker Swarm orchestration. The architecture is designed for production environments requiring high availability, automatic failover, and load balancing.

## Components

### PostgreSQL Cluster
- **Technology**: Bitnami PostgreSQL with repmgr
- **Nodes**: 3 instances (pg-1, pg-2, pg-3)
- **Replication**: Automatic primary-replica replication
- **Failover**: Automatic failover via repmgr

### Connection Pooling
- **Technology**: PgPool-II
- **Instances**: 2 replicas for redundancy
- **Features**: 
  - Connection pooling
  - Load balancing across PostgreSQL nodes
  - Health checking
  - Query routing

### Distributed Configuration
- **Technology**: etcd v3.5.8
- **Nodes**: 3 instances (etcd1, etcd2, etcd3)
- **Purpose**: Distributed consensus and configuration storage
- **Cluster**: Quorum-based cluster for high availability

### Reverse Proxy & Load Balancer

#### Option 1: Traefik (Default)
- **Version**: Latest (supports v2 and v3+)
- **Features**:
  - Automatic service discovery via Docker Swarm
  - Automatic SSL/TLS with Let's Encrypt
  - HTTP to HTTPS redirection
  - DNS challenge via CloudFlare
- **Configuration**: Label-based routing

#### Option 2: HAProxy (Alternative)
- **Version**: Latest stable
- **Features**:
  - High-performance TCP/HTTP load balancing
  - Manual SSL/TLS certificate management
  - Static configuration
- **Use Case**: When Traefik is not suitable

### Distributed Storage
- **Technology**: GlusterFS
- **Mount Point**: /mnt/glusterfs
- **Purpose**: Shared storage for application data and certificates
- **High Availability**: Replicated across all nodes

### Example Application
- **Application**: Ghost CMS
- **Instances**: 6 replicas
- **Database**: Uses PgPool for PostgreSQL access
- **Storage**: GlusterFS for content files

## Network Architecture

```
                                    ┌─────────────┐
                                    │   Internet  │
                                    └──────┬──────┘
                                           │
                                           │ HTTPS
                                           ▼
                            ┌──────────────────────────┐
                            │    Traefik/HAProxy       │
                            │  (SSL Termination)       │
                            └──────────┬───────────────┘
                                       │
                                       │ HTTP
                                       ▼
                            ┌──────────────────────────┐
                            │    Ghost CMS (x6)        │
                            └──────────┬───────────────┘
                                       │
                                       │ SQL
                                       ▼
                            ┌──────────────────────────┐
                            │    PgPool (x2)           │
                            │  Connection Pool         │
                            └──────────┬───────────────┘
                                       │
                                       │ SQL
                            ┌──────────┼───────────────┐
                            │          │               │
                            ▼          ▼               ▼
                      ┌─────────┬─────────┬─────────┐
                      │  pg-1   │  pg-2   │  pg-3   │
                      │ Primary │ Replica │ Replica │
                      └────┬────┴────┬────┴────┬────┘
                           │         │         │
                           │    Replication    │
                           └─────────┼─────────┘
                                     │
                                     │ Coordination
                                     ▼
                      ┌──────────────────────────────┐
                      │  etcd Cluster (x3)           │
                      │  Distributed Config Store    │
                      └──────────────────────────────┘
```

## Data Flow

### Write Operations
1. Client connects to PgPool
2. PgPool routes write to Primary PostgreSQL (pg-1)
3. Primary replicates to replicas (pg-2, pg-3)
4. repmgr monitors replication status

### Read Operations
1. Client connects to PgPool
2. PgPool distributes reads across all healthy PostgreSQL nodes
3. Load balancing improves performance

### Failover Process
1. Primary node fails
2. repmgr detects failure via etcd
3. repmgr promotes a replica to primary
4. PgPool automatically detects new primary
5. Application continues with minimal downtime

## Node Distribution

The cluster is designed for 3 physical/virtual nodes:

- **Node swarm-01**: pg-1, etcd1 (Primary node)
- **Node swarm-02**: pg-2, etcd2
- **Node swarm-03**: pg-3, etcd3
- **Any Manager Node**: Traefik/HAProxy

PgPool and Ghost CMS are distributed across available nodes without constraints.

## Deployment Strategy

### Infrastructure Setup (Ansible)
1. Bootstrap Debian systems
2. Install and configure Docker
3. Initialize Docker Swarm
4. Configure security (SSH, firewall)
5. Setup GlusterFS

### Application Deployment (Docker Stack)
1. Create overlay network
2. Deploy etcd cluster
3. Deploy PostgreSQL nodes
4. Deploy PgPool
5. Deploy Traefik or HAProxy
6. Deploy applications (Ghost CMS)

## Scalability Considerations

### Horizontal Scaling
- **Ghost CMS**: Can scale to many replicas
- **PgPool**: Can scale to multiple replicas
- **PostgreSQL**: Limited to read scaling (add more replicas)

### Vertical Scaling
- Increase resources per container
- Adjust PostgreSQL memory/CPU settings
- Increase PgPool connection limits

## High Availability Features

### Component Redundancy
- 3 PostgreSQL nodes (1 primary, 2 replicas)
- 3 etcd nodes (quorum-based)
- 2 PgPool instances
- Multiple Ghost CMS replicas

### Automatic Recovery
- PostgreSQL: Automatic failover via repmgr
- PgPool: Automatic backend detection
- Docker Swarm: Automatic container restart
- etcd: Quorum-based consensus

### Data Persistence
- PostgreSQL: Local volumes with replication
- GlusterFS: Distributed file system
- etcd: Persistent cluster state

## Security Architecture

### Network Isolation
- Internal overlay network (postgres-network)
- Services not exposed unless explicitly configured
- Traefik handles external exposure

### Access Control
- SSH on non-standard port (15122)
- Passwordless sudo for devops user
- PostgreSQL password authentication
- PgPool admin authentication

### SSL/TLS
- Automatic certificate management (Traefik)
- Let's Encrypt integration
- HTTPS enforcement via middleware

## Monitoring Considerations

While not implemented in the base configuration, consider adding:
- Prometheus for metrics collection
- Grafana for visualization
- PostgreSQL exporter for database metrics
- Node exporter for system metrics
- Traefik metrics endpoint

## Backup Strategy

Consider implementing:
- PostgreSQL WAL archiving
- Regular pg_dump backups
- GlusterFS snapshots
- etcd backup automation

## Performance Tuning

### PostgreSQL
- Adjust shared_buffers based on RAM
- Configure max_connections
- Tune checkpoint settings
- Enable query logging for optimization

### PgPool
- Adjust num_init_children (connection pool size)
- Configure max_pool (connections per child)
- Enable query caching if appropriate

### Traefik
- Configure rate limiting if needed
- Adjust timeout values
- Enable compression middleware
