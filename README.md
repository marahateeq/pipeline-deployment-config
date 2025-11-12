# Deployment Configuration

Central configuration repository for service deployments across multiple environments.

## Overview

This repository contains:
- **Inventory files**: Server definitions for Dev/QA/Prod
- **Service configurations**: Docker Compose templates and systemd service files
- **Environment variables**: Environment-specific configuration
- **Jenkins pipeline**: Automated deployment orchestration

## Repository Structure

```
deployment-config/
├── inventory.ini                    # Ansible inventory (all environments)
├── jenkinsfile                      # Jenkins deployment pipeline
├── docker-compose.global.yaml       # Global Docker Compose settings
├── services/                        # Docker service configurations
│   ├── user-api/
│   │   └── docker-compose.yaml.j2
│   └── product-frontend/
│       └── docker-compose.yaml.j2
├── system-services/                 # Systemd service configurations
│   └── data-processor/
│       ├── data-processor.service.j2
│       └── config.yaml
├── vars/                            # Environment-specific variables
│   ├── common-vars.yml
│   ├── dev-vars.yml
│   ├── qa-vars.yml
│   └── prod-vars.yml
└── README.md                        # This file
```

## Environments

### Development (dev)
- **Purpose**: Development and testing
- **Servers**: 1 server (10.10.10.101)
- **Auto-deploy**: Yes, on every commit
- **Resource limits**: Minimal
- **Logging**: DEBUG level

### QA (qa)
- **Purpose**: Quality assurance and integration testing
- **Servers**: 2 servers (10.10.10.201-202)
- **Auto-deploy**: On tagged releases
- **Resource limits**: Medium
- **Logging**: INFO level

### Production (prod)
- **Purpose**: Live production environment
- **Servers**: 3 servers (10.10.10.301-303)
- **Auto-deploy**: Manual approval required
- **Resource limits**: Full allocation
- **Logging**: WARNING level

## Services

### Docker Services

#### user-api
- **Type**: Python Flask REST API
- **Port**: 8080
- **Health endpoint**: /health
- **Dependencies**: None
- **Configuration**: `services/user-api/docker-compose.yaml.j2`

#### product-frontend
- **Type**: HTML/JS Frontend (Nginx)
- **Port**: 80 (3000 in dev)
- **Dependencies**: user-api
- **Configuration**: `services/product-frontend/docker-compose.yaml.j2`

### System Services

#### data-processor
- **Type**: Python systemd service
- **Purpose**: Background data processing
- **Configuration**: `system-services/data-processor/`
- **Runs as**: serviceuser
- **Restart**: on-failure

## Configuration Files

### Inventory File (`inventory.ini`)

Defines all servers and their groupings:

```ini
[dev]
dev-server-01 ansible_host=10.10.10.101

[qa]
qa-server-01 ansible_host=10.10.10.201
qa-server-02 ansible_host=10.10.10.202

[prod]
prod-server-01 ansible_host=10.10.10.301
prod-server-02 ansible_host=10.10.10.302
```

Update IP addresses and hostnames for your environment.

### Service Configuration Templates

Service configs use Jinja2 templating (`.j2` extension) with variables from `vars/` directory.

Example template variable:
```yaml
image: "{{ docker_registry }}/user-api:{{ version }}"
ports:
  - "{{ user_api_port }}:8080"
```

Variables are resolved from:
1. `vars/common-vars.yml` - Shared across all environments
2. `vars/<env>-vars.yml` - Environment-specific overrides
3. Ansible extra vars - Runtime overrides

## Usage

### Deploy Service via Jenkins

1. **Go to Jenkins** → Deployment Pipeline
2. **Click "Build with Parameters"**
3. **Set parameters**:
   - Environment: dev/qa/prod
   - Service Name: user-api (or leave empty for auto-detect)
   - Service Type: docker/systemd/auto
4. **Click "Build"**

### Manual Deployment with Ansible

Deploy Docker service:
```bash
cd ../pipeline-deployment
ansible-playbook deploy-services.yml \
  -i ../deployment-config/inventory.ini \
  -e "env=dev service_name=user-api"
```

Deploy system service:
```bash
cd ../pipeline-deployment
ansible-playbook deploy-system-service.yml \
  -i ../deployment-config/inventory.ini \
  -e "env=prod service_name=data-processor"
```

### Deploy All Services to Environment

```bash
# Deploy all Docker services to dev
for service in user-api product-frontend; do
  ansible-playbook deploy-services.yml \
    -i ../deployment-config/inventory.ini \
    -e "env=dev service_name=$service"
done

# Deploy system services
ansible-playbook deploy-system-service.yml \
  -i ../deployment-config/inventory.ini \
  -e "env=dev service_name=data-processor"
```

## Adding a New Service

### Docker Service

1. **Create service directory**:
   ```bash
   mkdir -p services/new-service
   ```

2. **Create docker-compose template**:
   ```bash
   touch services/new-service/docker-compose.yaml.j2
   ```

3. **Add configuration**:
   ```yaml
   version: '3.8'
   services:
     new-service:
       image: "{{ docker_registry }}/new-service:{{ version }}"
       ports:
         - "{{ new_service_port }}:8080"
       environment:
         - ENV={{ environment }}
   ```

4. **Add variables** to `vars/common-vars.yml`:
   ```yaml
   new_service_port: 9000
   ```

5. **Commit and push**:
   ```bash
   git add services/new-service
   git commit -m "Add new-service configuration"
   git push
   ```

### System Service

1. **Create service directory**:
   ```bash
   mkdir -p system-services/new-service
   ```

2. **Create systemd template**:
   ```bash
   touch system-services/new-service/new-service.service.j2
   ```

3. **Add systemd configuration**:
   ```ini
   [Unit]
   Description=New Service
   After=network.target

   [Service]
   Type=simple
   ExecStart=/opt/services/new-service/start.sh
   Restart=on-failure

   [Install]
   WantedBy=multi-user.target
   ```

4. **Add service files** (binaries, scripts, config)

5. **Commit and deploy**

## Variable Management

### Common Variables (`vars/common-vars.yml`)
Default values used across all environments:
- Docker registry
- Default ports
- Logging settings
- Resource limits

### Environment Variables (`vars/<env>-vars.yml`)
Override common values for specific environment:
- Database connections
- API endpoints
- Secret keys (should be vaulted)
- Resource allocations

### Variable Precedence
1. Ansible extra vars (`-e "var=value"`)
2. Environment vars (`<env>-vars.yml`)
3. Common vars (`common-vars.yml`)
4. Template defaults

## Security

### Sensitive Data

**Never commit secrets in plain text!**

Use Ansible Vault for sensitive variables:

```bash
# Create encrypted vars file
ansible-vault create vars/prod-secrets.yml

# Edit encrypted file
ansible-vault edit vars/prod-secrets.yml

# Use in playbook
ansible-playbook deploy-services.yml \
  -i inventory.ini \
  --vault-password-file ~/.vault_pass
```

### SSH Keys

Store SSH keys securely:
```bash
# Generate deployment key
ssh-keygen -t rsa -b 4096 -f ~/.ssh/deployment_key

# Add to inventory
[all:vars]
ansible_ssh_private_key_file=~/.ssh/deployment_key
```

## Monitoring

### Check Service Status

```bash
# Docker services
ansible dev -i inventory.ini -m shell \
  -a "docker ps --format 'table {{.Names}}\t{{.Status}}'"

# System services
ansible prod -i inventory.ini -m shell \
  -a "systemctl status data-processor" --become
```

### View Logs

```bash
# Docker service logs
ssh user@server
docker logs user-api --tail=50 -f

# System service logs
ssh user@server
sudo journalctl -u data-processor -f
```

## Troubleshooting

### Deployment Fails

1. **Check Ansible connectivity**:
   ```bash
   ansible all -i inventory.ini -m ping
   ```

2. **Verify variables**:
   ```bash
   ansible-playbook deploy-services.yml \
     -i inventory.ini \
     -e "env=dev service_name=user-api" \
     --check --diff
   ```

3. **Check service logs** on target server

### Service Won't Start

1. **Docker service**:
   ```bash
   docker logs <container-name>
   docker inspect <container-name>
   ```

2. **System service**:
   ```bash
   systemctl status <service-name>
   journalctl -u <service-name> -n 50
   ```

### Configuration Not Applied

1. Ensure Jinja2 template syntax is correct
2. Check variable precedence
3. Verify file was transferred: `ansible <host> -i inventory.ini -m shell -a "cat /opt/services/<service>/docker-compose.yml"`

## Best Practices

1. **Version Control**: Always commit configuration changes
2. **Testing**: Test in dev before deploying to prod
3. **Documentation**: Document custom variables
4. **Validation**: Use `--check` mode to dry-run deployments
5. **Rollback**: Keep previous configurations for rollback
6. **Monitoring**: Set up alerts for service failures
7. **Secrets**: Use Ansible Vault for all sensitive data
8. **Backups**: Backup configurations before major changes

## Jenkins Integration

The `jenkinsfile` in this repository defines the automated deployment pipeline:

- **Triggered by**: Git push or manual execution
- **Detects changes**: Automatically identifies modified services
- **Deploys**: Calls Ansible playbooks for deployment
- **Notifies**: Sends email on success/failure
- **Verifies**: Checks service health after deployment

## Contributing

1. Create feature branch
2. Make configuration changes
3. Test in dev environment
4. Create pull request
5. Get approval
6. Merge to main

## License

Internal use only - proprietary
