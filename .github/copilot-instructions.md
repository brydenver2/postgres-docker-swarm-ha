# GitHub Copilot Instructions

This file helps GitHub Copilot understand the project structure and provide better assistance.

For detailed instructions, see [../documents/COPILOT_INSTRUCTIONS.md](../documents/COPILOT_INSTRUCTIONS.md)

## Quick Reference

### Project Type
Docker Swarm infrastructure for highly available PostgreSQL cluster

### Key Files
- `docker-compose.yml` - Main configuration with Traefik
- `docker-compose.haproxy.yml` - Alternative with HAProxy
- `.env_postgres` - Environment variables and credentials
- `playbook.yml` - Ansible infrastructure provisioning

### Technologies
- Docker Swarm (orchestration)
- PostgreSQL with repmgr (database clustering)
- PgPool (connection pooling)
- etcd (distributed configuration)
- Traefik or HAProxy (load balancing)
- GlusterFS (distributed storage)

### Code Patterns
- YAML anchors for configuration reuse
- Swarm deployment constraints
- Label-based routing (Traefik)
- Environment file configuration

### Testing
- Validation: `docker compose config`
- No unit tests (infrastructure project)

See the full guide in `documents/COPILOT_INSTRUCTIONS.md` for comprehensive information about architecture, patterns, and best practices.
