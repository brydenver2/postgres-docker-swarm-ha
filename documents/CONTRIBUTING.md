# Contributing to PostgreSQL Docker Swarm HA

Thank you for your interest in contributing to this project! This guide will help you understand how to contribute effectively.

## Code of Conduct

- Be respectful and constructive
- Focus on what is best for the community
- Show empathy towards other community members

## Getting Started

### Prerequisites

- Docker Engine 18.06.0+ with Swarm mode enabled
- Docker Compose 1.24.0+
- Ansible 2.9+ (for infrastructure provisioning)
- Basic understanding of:
  - Docker and Docker Swarm
  - PostgreSQL and replication
  - YAML and configuration management

### Setting Up Development Environment

1. Fork the repository
2. Clone your fork:
   ```bash
   git clone https://github.com/YOUR_USERNAME/postgres-docker-swarm-ha.git
   cd postgres-docker-swarm-ha
   ```
3. Create a feature branch:
   ```bash
   git checkout -b feature/your-feature-name
   ```

## Project Structure

```
.
├── docker-compose.yml           # Main service definitions
├── .env_postgres               # Environment variables (sensitive)
├── playbook.yml                # Ansible provisioning playbook
├── hosts.yaml                  # Ansible inventory
├── start.sh                    # Deployment script
├── documents/                  # Project documentation
│   ├── COPILOT_INSTRUCTIONS.md
│   ├── ARCHITECTURE.md
│   ├── CONTRIBUTING.md
│   ├── DEPLOYMENT_GUIDE.md
│   └── LOAD_BALANCER_OPTIONS.md
└── roles/                      # Ansible roles (external)
```

## Making Changes

### Code Style

#### YAML Files
- Use 2 spaces for indentation
- Use lowercase for keys
- Add comments for complex configurations
- Maintain YAML anchor patterns for reusability

Example:
```yaml
service-1: &base_service
  image: example:latest
  networks:
    - app-network
  deploy:
    replicas: 1

service-2:
  <<: *base_service  # Reuse base configuration
  environment:
    - CUSTOM_VAR=value
```

#### Shell Scripts
- Use bash shebang: `#!/bin/bash`
- Add error handling: `set -e`
- Comment complex logic
- Use meaningful variable names

### Docker Compose Guidelines

1. **Service Naming**: Use descriptive names with hyphens (e.g., `pg-1`, `etcd1`)
2. **Networks**: Always use the `postgres-network` for service communication
3. **Volumes**: Define volumes at the bottom of the file
4. **Environment Variables**: Use `.env_postgres` for sensitive data
5. **Health Checks**: Add health checks for critical services
6. **Deployment Constraints**: Use node constraints when pinning services

### Adding New Services

1. Define the service in `docker-compose.yml`
2. Add required environment variables to `.env_postgres`
3. Configure networking and volumes
4. Add deployment constraints if needed
5. Configure Traefik labels for external access (if applicable)
6. Update documentation

Example:
```yaml
new-service:
  image: example/service:latest
  env_file: .env_postgres
  networks:
    - postgres-network
  deploy:
    replicas: 2
    labels:
      - traefik.enable=true
      - traefik.http.routers.service.rule=Host(`service.example.com`)
```

### Modifying Existing Services

1. Understand the current configuration
2. Test changes in a non-production environment
3. Maintain backward compatibility when possible
4. Update related documentation
5. Validate with `docker-compose config`

## Testing

### Validation Commands

```bash
# Validate docker-compose syntax
docker-compose config

# Check for syntax errors without deploying
docker-compose -f docker-compose.yml config > /dev/null

# Validate YAML files
yamllint docker-compose.yml
```

### Testing in Docker Swarm

1. Initialize a test swarm:
   ```bash
   docker swarm init
   ```

2. Create the required network:
   ```bash
   docker network create --driver=overlay --attachable postgres-network
   ```

3. Deploy the stack:
   ```bash
   docker stack deploy -c docker-compose.yml test-stack
   ```

4. Check service status:
   ```bash
   docker service ls
   docker service ps test-stack_pg-1
   ```

5. View logs:
   ```bash
   docker service logs test-stack_pg-1
   ```

6. Clean up:
   ```bash
   docker stack rm test-stack
   ```

### Testing Database Connectivity

```bash
# Connect to PgPool
docker exec -it $(docker ps -q -f name=pgpool) psql -U ghost_cms -d ghost_cms -h localhost

# Check replication status
docker exec -it $(docker ps -q -f name=pg-1) psql -U postgres -c "SELECT * FROM pg_stat_replication;"
```

## Documentation

### When to Update Documentation

- Adding new features or services
- Changing configuration options
- Modifying deployment procedures
- Fixing bugs that affect documentation accuracy

### Documentation Files to Update

- **README.md**: Overview and quick start
- **ARCHITECTURE.md**: Architecture changes
- **DEPLOYMENT_GUIDE.md**: Deployment procedure changes
- **LOAD_BALANCER_OPTIONS.md**: Load balancer configuration changes
- **COPILOT_INSTRUCTIONS.md**: Project structure or pattern changes

## Pull Request Process

1. **Create a Feature Branch**: 
   ```bash
   git checkout -b feature/descriptive-name
   ```

2. **Make Your Changes**: Follow the guidelines above

3. **Test Your Changes**: Validate and test thoroughly

4. **Commit Your Changes**:
   ```bash
   git add .
   git commit -m "feat: descriptive commit message"
   ```
   
   Use conventional commit messages:
   - `feat:` for new features
   - `fix:` for bug fixes
   - `docs:` for documentation changes
   - `chore:` for maintenance tasks
   - `refactor:` for code refactoring

5. **Push to Your Fork**:
   ```bash
   git push origin feature/descriptive-name
   ```

6. **Create a Pull Request**:
   - Provide a clear title and description
   - Reference any related issues
   - Include testing steps
   - Add screenshots for UI changes

### Pull Request Checklist

- [ ] Code follows project style guidelines
- [ ] Changes have been tested
- [ ] Documentation has been updated
- [ ] Commit messages are clear and descriptive
- [ ] No sensitive data (passwords, tokens) in commits
- [ ] YAML files pass validation
- [ ] Changes are backward compatible (or breaking changes are documented)

## Common Issues and Solutions

### Issue: Service Won't Start
- Check logs: `docker service logs <service-name>`
- Verify environment variables in `.env_postgres`
- Ensure required networks exist
- Check node constraints match your environment

### Issue: PostgreSQL Replication Not Working
- Verify all nodes can communicate
- Check repmgr configuration
- Ensure etcd cluster is healthy
- Review PostgreSQL logs

### Issue: Traefik Not Routing Traffic
- Verify labels are correct
- Check Traefik logs
- Ensure DNS is configured correctly
- Verify SSL certificates are valid

## Security Considerations

### Do Not Commit Sensitive Data
- Never commit passwords, API keys, or tokens
- Use `.env_postgres` for sensitive configuration
- Review changes before committing: `git diff`
- Use `.gitignore` to exclude sensitive files

### Security Best Practices
- Use strong passwords
- Limit SSH access
- Keep images updated
- Use specific image versions in production
- Enable Docker secrets for production
- Implement network policies

## Getting Help

- **Issues**: Open an issue for bugs or feature requests
- **Discussions**: Use GitHub Discussions for questions
- **Documentation**: Check the `documents/` folder

## Recognition

Contributors will be recognized in:
- GitHub contributors list
- Release notes (for significant contributions)
- Project documentation (for major features)

Thank you for contributing to make this project better!
