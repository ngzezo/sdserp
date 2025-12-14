# SDS ERP Docker Deployment

This directory contains Docker configuration files to run SDS ERP on Ubuntu 24.04 ARM64 virtual machines with white-labeled branding.

## Overview

The Docker setup is based on the [frappe_docker](https://github.com/frappe/frappe_docker) reference architecture but customized for SDS ERP branding. It includes:

- **Dockerfile**: Multi-stage build for ARM64/AMD64 compatibility
- **docker-compose.yml**: Complete stack with all required services
- **nginx configuration**: Reverse proxy and static asset serving
- **White-labeled branding**: All ERPNext references replaced with "SDS ERP"

## Architecture

The stack consists of the following services:

- **backend**: Gunicorn WSGI server running SDS ERP
- **frontend**: Nginx reverse proxy serving static assets
- **websocket**: Socket.IO server for real-time features
- **db**: MariaDB 10.6 database
- **redis-cache**: Redis for caching
- **redis-queue**: Redis for background job queues
- **scheduler**: Cron-like scheduler for recurring tasks
- **queue-short/long**: Background workers for async jobs
- **configurator**: Initialization service (runs once)
- **create-site**: Site creation service (runs once)

## Prerequisites

- Ubuntu 24.04 LTS (ARM64 or AMD64)
- Docker Engine 24.0+ 
- Docker Compose V2
- At least 4GB RAM
- At least 20GB disk space

### Install Docker on Ubuntu 24.04

```bash
# Update package index
sudo apt-get update

# Install required packages
sudo apt-get install -y ca-certificates curl gnupg

# Add Docker's official GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Add your user to the docker group (optional, to run docker without sudo)
sudo usermod -aG docker $USER
newgrp docker

# Verify installation
docker --version
docker compose version
```

## Quick Start

### 1. Build the Docker Image

```bash
# Navigate to the project root
cd /workspaces/sdserp

# Build the image (this will take 15-30 minutes on first build)
docker compose build

# For ARM64 specifically
docker compose build --build-arg TARGETARCH=arm64
```

### 2. Start the Services

```bash
# Start all services in the background
docker compose up -d

# Monitor the logs
docker compose logs -f

# Watch specifically for site creation
docker compose logs -f create-site
```

### 3. Wait for Initial Setup

The `create-site` service will automatically:
- Wait for database and Redis to be ready
- Configure the common site config
- Create a default site named `sdserp.localhost`
- Install the SDS ERP app

This process takes approximately 5-10 minutes. Watch the logs to see progress.

### 4. Access SDS ERP

Once setup is complete, access SDS ERP at:

**URL**: http://localhost:8462

**Default Credentials**:
- Username: `Administrator`
- Password: `admin`

**Important**: Change the default password immediately after first login!

## Configuration

### Environment Variables

Key environment variables can be set in `docker-compose.yml`:

```yaml
environment:
  # Database
  MYSQL_ROOT_PASSWORD: admin  # Change in production!
  
  # Site name
  FRAPPE_SITE_NAME_HEADER: sdserp.localhost
  
  # Backend services
  BACKEND: backend:8000
  SOCKETIO: websocket:9000
```

### Site Name

To change the site name from `sdserp.localhost`:

1. Edit the site name in the `create-site` service command in `docker-compose.yml`
2. Update `FRAPPE_SITE_NAME_HEADER` in the frontend service
3. Rebuild and restart services

### Ports

By default, SDS ERP is exposed on port 8462. To change:

```yaml
frontend:
  ports:
    - "YOUR_PORT:8080"  # Change YOUR_PORT to desired port
```

## Production Deployment

### Security Checklist

Before deploying to production:

1. **Change all default passwords**:
   - Administrator password
   - Database root password (`MYSQL_ROOT_PASSWORD`)

2. **Use HTTPS**:
   - Set up a reverse proxy (nginx, Traefik) with SSL certificates
   - Use Let's Encrypt for free SSL certificates

3. **Secure the database**:
   - Don't expose MariaDB port externally
   - Use strong passwords
   - Regular backups

4. **Resource limits**:
   - Set memory and CPU limits in docker-compose.yml
   - Monitor resource usage

5. **Firewall**:
   - Only expose necessary ports (80, 443)
   - Block direct access to internal services

### Persistent Data

All data is stored in Docker volumes:

```bash
# List volumes
docker volume ls

# Backup volumes
docker run --rm -v sdserp_sites:/data -v $(pwd):/backup alpine tar czf /backup/sites-backup.tar.gz /data
docker run --rm -v sdserp_db-data:/data -v $(pwd):/backup alpine tar czf /backup/db-backup.tar.gz /data
```

## Common Operations

### View Logs

```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f backend
docker compose logs -f frontend
docker compose logs -f db
```

### Restart Services

```bash
# Restart all
docker compose restart

# Restart specific service
docker compose restart backend
```

### Stop Services

```bash
# Stop all services
docker compose stop

# Stop and remove containers (data preserved in volumes)
docker compose down

# Stop and remove everything including volumes (⚠️ DATA LOSS!)
docker compose down -v
```

### Execute Commands in Container

```bash
# Access backend shell
docker compose exec backend bash

# Run bench commands
docker compose exec backend bench --help
docker compose exec backend bench --site sdserp.localhost migrate
docker compose exec backend bench --site sdserp.localhost backup
```

### Create Additional Sites

```bash
docker compose exec backend bench new-site my-new-site.local \
  --mariadb-user-host-login-scope=% \
  --db-root-password=admin \
  --admin-password=admin \
  --install-app erpnext
```

### Backup and Restore

```bash
# Create backup
docker compose exec backend bench --site sdserp.localhost backup \
  --with-files

# Backups are stored in sites/sdserp.localhost/private/backups/

# Restore from backup
docker compose exec backend bench --site sdserp.localhost restore \
  /home/frappe/frappe-bench/sites/sdserp.localhost/private/backups/[BACKUP_FILE]
```

## Troubleshooting

### Site Not Loading

1. Check if all containers are running:
   ```bash
   docker compose ps
   ```

2. Check logs for errors:
   ```bash
   docker compose logs backend
   docker compose logs frontend
   ```

3. Verify the site was created successfully:
   ```bash
   docker compose exec backend ls -la sites/
   ```

### Database Connection Issues

```bash
# Check if MariaDB is running
docker compose ps db

# Test database connection
docker compose exec db mysql -u root -padmin -e "SHOW DATABASES;"

# Check database logs
docker compose logs db
```

### Performance Issues

1. Check resource usage:
   ```bash
   docker stats
   ```

2. Increase worker count in docker-compose.yml if needed

3. Add more memory to VM if container is being OOM killed

### Port Already in Use

If port 8462 is already in use:

```bash
# Find what's using the port
sudo lsof -i :8462

# Change the port in docker-compose.yml frontend service
```

## Upgrading

To upgrade to a newer version:

```bash
# Pull latest code
git pull

# Rebuild images
docker compose build --no-cache

# Stop services
docker compose down

# Start with new images
docker compose up -d

# Run migrations
docker compose exec backend bench --site sdserp.localhost migrate
```

## ARM64 Specific Notes

This Docker setup is optimized for ARM64 (Apple Silicon, AWS Graviton, etc.):

- Automatically detects architecture and downloads correct wkhtmltopdf
- All base images support ARM64
- Build times may vary between architectures

To explicitly build for ARM64:

```bash
docker buildx build --platform linux/arm64 -t sdserp:latest .
```

## Development vs Production

This setup is suitable for both development and production with modifications:

**Development**:
- Use default passwords
- Expose all ports for debugging
- Mount source code as volumes for live reload
- Enable developer mode in site config

**Production**:
- Change all passwords
- Use HTTPS with proper certificates
- Set up monitoring and alerting  
- Regular automated backups
- Use secrets management
- Configure firewalls
- Set resource limits

## Support

For issues or questions:

1. Check the logs first
2. Review [Frappe Framework Documentation](https://frappeframework.com/docs)
3. Review [frappe_docker GitHub](https://github.com/frappe/frappe_docker)
4. Check Docker and docker-compose versions

## License

SDS ERP inherits its license from ERPNext (GPL v3). See the main LICENSE file for details.
