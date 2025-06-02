# DHIS2 Docker Project

This project sets up a DHIS2 instance with a Postgres database using Docker.

## Project Structure

```
.
├── data/
│   ├── .gitignore
│   ├── dhis/
│   │   └── dhis.conf
│   └── postgres/
└── docker-compose.yml
```

## Prerequisites

Before you can run this project, you need to have Docker and Docker Compose installed on your machine.

- [Install Docker](https://docs.docker.com/get-docker/)
- [Install Docker Compose](https://docs.docker.com/compose/install/)

## Quick Start

### 1. Clone and Setup

```bash
# Clone the repository
git clone <repository-url>
cd dhis2-docker

# Create data directories (if they don't exist)
mkdir -p data/dhis data/postgres
```

### 2. Start DHIS2

```bash
# Start in detached mode (recommended)
docker-compose up -d

# Or start with logs visible
docker-compose up
```

The DHIS2 instance will be accessible at `http://localhost:8080`.

**Default credentials:**
- Username: `admin`
- Password: `district`

### 3. Verify Installation

```bash
# Check if containers are running
docker ps

# Check DHIS2 logs for successful startup
docker logs dhis2 --tail 20

# Test web interface accessibility
curl -I http://localhost:8080
```

If you see a `HTTP/1.1 302` response with a redirect to `/dhis-web-login/`, the installation is successful.

## Custom Domain/URL Configuration

To access DHIS2 on a custom domain or specific URL instead of localhost:

### Method 1: Using a Custom Domain

1. **Update docker-compose.yml environment variables:**
   ```yaml
   environment:
     SERVER_BASE_URL: http://your-domain.com:8080
     # or for HTTPS:
     # SERVER_BASE_URL: https://your-domain.com
     # SERVER_HTTPS: "on"
   ```

2. **Configure your domain to point to your server:**
   - Update DNS records to point to your server's IP address
   - Or add entry to `/etc/hosts` file for local testing:
     ```bash
     echo "127.0.0.1 your-domain.com" | sudo tee -a /etc/hosts
     ```

3. **Restart the containers:**
   ```bash
   docker-compose down
   docker-compose up -d
   ```

### Method 2: Using Different Port

1. **Change the port mapping in docker-compose.yml:**
   ```yaml
   ports:
     - 80:8080  # Access on port 80
     # or
     - 443:8080  # For HTTPS setup
   ```

2. **Update the base URL:**
   ```yaml
   environment:
     SERVER_BASE_URL: http://your-domain.com
     # No port needed if using standard ports (80/443)
   ```

### Method 3: Behind a Reverse Proxy (Nginx/Apache)

1. **Keep internal port mapping:**
   ```yaml
   ports:
     - 127.0.0.1:8080:8080  # Only accessible locally
   ```

2. **Configure your reverse proxy to forward requests to localhost:8080**

3. **Update DHIS2 base URL to match your proxy:**
   ```yaml
   environment:
     SERVER_BASE_URL: https://dhis2.your-domain.com
     SERVER_HTTPS: "on"
   ```

### Important Notes

- Always restart containers after changing environment variables
- For production deployments, use HTTPS with proper SSL certificates
- The `SERVER_BASE_URL` must match exactly how users will access DHIS2
- DHIS2 uses this URL for generating links and redirects

### 3. Stop DHIS2

```bash
# Stop containers (preserves data)
docker-compose down

# Stop and remove volumes (WARNING: This will delete all data)
docker-compose down -v
```

## Upgrading DHIS2 Version

To upgrade DHIS2 to a new version without losing data:

### Method 1: Edit docker-compose.yml

1. **Stop the current containers:**
   ```bash
   docker-compose down
   ```

2. **Edit docker-compose.yml:**
   ```bash
   # Change the image version
   # From: image: dhis2/core:2.40
   # To:   image: dhis2/core:2.41
   ```

3. **Pull the new image and start:**
   ```bash
   docker-compose pull dhis2
   docker-compose up -d
   ```

### Method 2: Using Docker Commands

```bash
# Stop containers
docker-compose down

# Pull new DHIS2 version
docker pull dhis2/core:2.41

# Update the image tag in docker-compose.yml
sed -i 's/dhis2\/core:.*/dhis2\/core:2.41/' docker-compose.yml

# Start with new version
docker-compose up -d
```

### Important Notes for Upgrades

- **Data Persistence:** Your data is stored in `./data/postgres` and `./data/dhis` directories and will be preserved during upgrades
- **Database Migration:** DHIS2 automatically handles database migrations when starting with a new version
- **Backup Recommended:** Always backup your data before major version upgrades:
  ```bash
  # Backup database
  docker-compose exec postgres pg_dump -U dhis dhis2 > backup_$(date +%Y%m%d).sql
  
  # Backup DHIS2 files
  tar -czf dhis_backup_$(date +%Y%m%d).tar.gz data/dhis/
  ```

## Troubleshooting

### Common Issues and Solutions

#### 1. DHIS2 Container Fails to Start - Configuration File Not Found

**Symptoms:**
- Container keeps restarting
- Logs show: `File /opt/dhis2/dhis.conf cannot be read`
- Error: `LocationManagerException: File /opt/dhis2/dhis.conf cannot be read`

**Solution:**
```bash
# Create symbolic link inside the container
docker exec dhis2 bash -c "mkdir -p /opt/dhis2 && ln -sf /DHIS2_home/dhis.conf /opt/dhis2/dhis.conf"

# Restart the container
docker restart dhis2

# Verify the fix
docker logs dhis2 --tail 20
```

**Permanent Fix:** Update your docker-compose.yml to include the `DHIS2_CONF_DIR` environment variable:
```yaml
environment:
  DHIS2_HOME: /DHIS2_home
  DHIS2_CONF_DIR: /DHIS2_home  # Add this line
  # ... other environment variables
```

#### 2. Database Connection Issues

**Symptoms:**
- DHIS2 fails to connect to PostgreSQL
- Logs show database connection errors

**Solution:**
```bash
# Check if PostgreSQL is running
docker ps | grep postgres

# Check PostgreSQL logs
docker logs postgres

# Verify database credentials in docker-compose.yml match
# Restart both containers in correct order
docker-compose down
docker-compose up -d postgres
sleep 10
docker-compose up -d dhis2
```

#### 3. Port Already in Use

**Symptoms:**
- Error: `port is already allocated`
- Cannot start containers

**Solution:**
```bash
# Check what's using port 8080
sudo netstat -tulpn | grep :8080
# or
sudo lsof -i :8080

# Stop the conflicting service or change port in docker-compose.yml
# Example: Change to port 8081
ports:
  - 8081:8080
```

#### 4. Slow Startup or Performance Issues

**Symptoms:**
- DHIS2 takes very long to start
- Web interface is slow

**Solution:**
```bash
# Increase memory allocation in docker-compose.yml
JAVA_OPTS: -Xms4096m -Xmx8192m  # Increase from default 2GB/4GB

# Check system resources
docker stats

# Ensure sufficient disk space
df -h
```

#### 5. Web Interface Not Accessible

**Symptoms:**
- Cannot access http://localhost:8080
- Connection refused or timeout

**Solution:**
```bash
# Check if DHIS2 container is running
docker ps | grep dhis2

# Check DHIS2 logs for startup completion
docker logs dhis2 | grep "Server startup"

# Test connectivity
curl -I http://localhost:8080

# If using custom domain, check SERVER_BASE_URL in docker-compose.yml
```

#### 6. Data Loss After Container Restart

**Symptoms:**
- All data disappears after restarting containers
- Fresh DHIS2 installation every time

**Solution:**
```bash
# Ensure data directories exist and have correct permissions
mkdir -p data/dhis data/postgres
sudo chown -R $USER:$USER data/

# Check volume mounts in docker-compose.yml
# Verify data persistence
ls -la data/postgres/  # Should contain PostgreSQL data files
ls -la data/dhis/      # Should contain dhis.conf
```

### Diagnostic Commands

```bash
# Complete system check
echo "=== Container Status ==="
docker ps

echo "\n=== DHIS2 Logs (last 20 lines) ==="
docker logs dhis2 --tail 20

echo "\n=== PostgreSQL Logs (last 10 lines) ==="
docker logs postgres --tail 10

echo "\n=== Network Connectivity ==="
curl -I http://localhost:8080

echo "\n=== Disk Usage ==="
df -h data/

echo "\n=== Memory Usage ==="
docker stats --no-stream
```

## Monitoring and Logs

```bash
# View logs
docker-compose logs -f dhis2
docker-compose logs -f postgres

# Check container status
docker-compose ps

# Access DHIS2 container shell
docker-compose exec dhis2 bash

# Access PostgreSQL
docker-compose exec postgres psql -U dhis -d dhis2
```

## Complete Removal

To permanently delete everything (containers, images, volumes, and the project):

### Option 1: Quick Removal (Recommended)

```bash
# Stop and remove containers, networks, and volumes
docker-compose down -v

# Remove DHIS2 and PostgreSQL images
docker rmi dhis2/core:2.42 postgis/postgis:16-3.4

# Go back to parent directory and remove the project
cd ..
rm -rf dhis2-docker
```

### Option 2: Step-by-Step Removal

```bash
# 1. Stop and remove containers
docker-compose down

# 2. Remove volumes (this deletes all data)
docker volume rm dhis2-docker_postgres-data 2>/dev/null || true

# 3. Remove network
docker network rm dhis2 2>/dev/null || true

# 4. Remove images
docker rmi dhis2/core:2.42
docker rmi postgis/postgis:16-3.4

# 5. Remove any dangling images (optional)
docker image prune -f

# 6. Remove the project directory
cd ..
rm -rf dhis2-docker
```

### Verify Complete Removal

```bash
# Check for any remaining DHIS2 containers
docker ps -a | grep dhis2

# Check for any remaining DHIS2 images
docker images | grep dhis2

# Check for any remaining volumes
docker volume ls | grep dhis2

# Check for any remaining networks
docker network ls | grep dhis2
```

**Warning:** This will permanently delete all DHIS2 data, configurations, and the entire project. Make sure to backup any important data before proceeding.

## Data Persistence

The `data` directory contains data for the DHIS2 and Postgres containers. The `dhis` directory contains the DHIS2 configuration file (`dhis.conf`), and the `postgres` directory contains the Postgres data. The `dhis.conf` are loaded through environmental variables. Please adjust them accordingly

```yaml
services:
  dhis2:
    container_name: dhis2
    environment:
      DHIS2_HOME: /DHIS2_home
      DHIS2_CONF_DIR: /DHIS2_home
      JAVA_OPTS: -Xms2048m -Xmx4096m
      TZ: Africa/Nairobi
      POSTGRES_HOST: postgres
      POSTGRES_PORT: 5432
      POSTGRES_DB: dhis2
      POSTGRES_USER: dhis
      POSTGRES_PASSWORD: dhis
      SERVER_HTTPS: "off"
      SERVER_BASE_URL: http://localhost:8080
    depends_on:
      - postgres
    image: dhis2/core:2.42
    networks:
      - dhis2
    platform: linux/amd64
    ports:
      - 8080:8080
    restart: unless-stopped
    volumes:
      - ./data/dhis:/DHIS2_home

  postgres:
    container_name: postgres
    environment:
      POSTGRES_DB: dhis2
      POSTGRES_USER: dhis
      POSTGRES_PASSWORD: dhis
    image: postgis/postgis:16-3.4
    networks:
      - dhis2
    platform: linux/amd64
    # ports:
    #   - 5432:5432
    restart: unless-stopped
    volumes:
      - ./data/postgres:/var/lib/postgresql/data

networks:
  dhis2:
    driver: bridge
    name: dhis2
```

## Contributing

Contributions are welcome. Please submit a pull request or create an issue to discuss the changes.

## License

This project is licensed under the MIT License.
