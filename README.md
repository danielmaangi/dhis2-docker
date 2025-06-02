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
