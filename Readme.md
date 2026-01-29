# Monarc Docker Image

[![Docker Pulls](https://img.shields.io/docker/pulls/mitexleo/monarc.lu)](https://hub.docker.com/r/mitexleo/monarc.lu)
[![Monarc Version](https://img.shields.io/badge/Monarc-2.13.3--p6-blue)](https://www.monarc.lu)

This Docker image provides a containerized version of **Monarc** (Method for an Optimised aNalysis of Risks) from NC3 Luxembourg (https://monarc.lu). Since there is no official Docker image, this community-maintained version offers an easy way to deploy and run Monarc in containerized environments.

## 🚀 Quick Start

### Prerequisites
- Docker and Docker Compose
- At least 2GB RAM recommended

### Using Docker Compose (Recommended)

1. **Clone the repository:**
   ```bash
   git clone <repository-url>
   cd monarc-docker

   **Note**: Docker Compose supports multiple file names: `docker-compose.yml`, `compose.yml`, or `docker-compose.yaml`. This repository includes both `docker-compose.yml` and `compose.yml` (they are identical). Use whichever filename you prefer.
   ```

2. **Set up environment variables (optional but recommended for production):**
   ```bash
   cp env.example .env
   # Edit .env and set secure passwords
   # If .env is not created, default passwords will be used
   nano .env
   ```

3. **Start the stack:**
   ```bash
   docker-compose up -d
   ```

4. **Access the application:**
   - URL: http://localhost:8086
   - Default login: `admin@admin.localhost`
   - Default password: `admin`

5. **Check logs:**
   ```bash
   docker-compose logs -f monarc
   ```

### Using Docker Hub Image Directly

```bash
# Create network
docker network create monarc-network

# Start MariaDB
docker run -d \
  --name monarc-db \
  --network monarc-network \
  -e MARIADB_ROOT_PASSWORD=your_root_password \
  -e MARIADB_DATABASE=monarc_cli \
  -e MARIADB_USER=monarc \
  -e MARIADB_PASSWORD=your_password \
  -v db_data:/var/lib/mysql \
  mariadb:lts

# Start Monarc
docker run -d \
  --name monarc \
  --network monarc-network \
  -p 8086:80 \
  -e DB_HOST=monarc-db \
  -e DB_USER=monarc \
  -e DB_PASSWORD=your_password \
  -e DB_ROOT_PASSWORD=your_root_password \
  -e DB_CLI_NAME=monarc_cli \
  -e DB_COMMON_NAME=monarc_common \
  -e APPLICATION_ENV=production \
  -v monarc_data:/var/lib/monarc/fo/data \
  mitexleo/monarc.lu:2.13.3-p6
```

## 📦 Available Tags

| Tag | Description |
|-----|-------------|
| `2.13.3-p6` | Latest patch version (recommended) |
| `2.13.3` | Major version 2.13.3 |
| `latest` | Latest stable version |

## 🛠️ Building from Source

If you prefer to build the Docker image locally instead of using the pre-built images from Docker Hub, follow these steps:

### Prerequisites
- Docker installed on your system
- Git (to clone the repository)
- Sufficient disk space (approximately 2GB during build)

### Build Steps

1. **Clone the repository:**
   ```bash
   git clone <repository-url>
   cd monarc-docker
   ```

2. **Build the Docker image:**
   ```bash
   # Build with default Monarc version (2.13.3-p6)
   docker build -t monarc:local .
   
   # Or specify a custom tag
   docker build -t monarc:2.13.3-p6-local .
   ```

3. **Test the built image:**
   ```bash
   # Run a quick test
   docker run --rm monarc:local --help
   ```

### Custom Build Options

You can customize the build by modifying the `Dockerfile`:

1. **Change Monarc version:** Edit the `MONARC_VERSION` environment variable in the Dockerfile
   ```dockerfile
   ENV MONARC_VERSION=v2.13.2  # Change to desired version
   ```

2. **Add build arguments:** Use Docker build arguments for customization
   ```bash
   docker build --build-arg MONARC_VERSION=v2.13.2 -t monarc:custom .
   ```

3. **Include development tools:** Uncomment XDEBUG in Dockerfile for development
   ```dockerfile
   # Uncomment these lines for development:
   # RUN pecl install xdebug
   # RUN docker-php-ext-enable xdebug
   ```

### Build Optimization

- **Cache utilization:** Docker will cache layers for faster subsequent builds
- **Multi-stage builds:** Consider implementing multi-stage builds for smaller final images
- **Security scanning:** Use `docker scan` to check for vulnerabilities in your built image

### Verification

After building, verify the image was created successfully:
```bash
# List Docker images
docker images | grep monarc

# Check image details
docker inspect monarc:local
```

### Using Your Custom Build

Update your `docker-compose.yml` to use your locally built image:
```yaml
services:
  monarc:
    build: .  # Build from local Dockerfile
    # OR use your built image:
    # image: monarc:local
    # Remove: image: mitexleo/monarc.lu:2.13.3-p6
```

### Troubleshooting Build Issues

- **Network errors during apt-get:** Ensure Docker has network access
- **Out of disk space:** Clean up unused Docker images with `docker system prune`
- **Permission errors:** Run Docker commands with appropriate privileges
- **Build timeouts:** Increase Docker daemon timeout if building on slow connections

## ⚙️ Environment Configuration

### Required Variables
- `DB_PASSWORD`: Password for Monarc database user (default: `monarc_password`)
- `DB_ROOT_PASSWORD`: MariaDB root password (default: `root_password`)

### Optional Variables
- `DB_USER`: Database username (default: `monarc`)
- `DB_CLI_NAME`: CLI database name (default: `monarc_cli`)
- `DB_COMMON_NAME`: Common database name (default: `monarc_common`)
- `APPLICATION_ENV`: Application environment (default: `production`)

### Security Best Practices
1. **Always** change default passwords in production
2. Use Docker secrets in production environments
3. Regularly rotate database passwords
4. Set up proper network isolation
5. Enable HTTPS with reverse proxy

### Default Passwords
If no `.env` file is provided, the following defaults are used:
- Database user password: `monarc_password`
- Database root password: `root_password`
- Database username: `monarc`
- Database names: `monarc_cli` and `monarc_common`

**Important**: These defaults are for testing only. Always use secure, unique passwords in production.

## 🗄️ Persistent Storage

The Docker Compose configuration creates two volumes:

1. **`db_data`**: MariaDB database files
2. **`monarc_data`**: Monarc application data (cache, imports, uploads)

To backup data:
```bash
# Backup database (using default or custom password)
docker exec monarc-db mysqldump -u monarc -p monarc_cli > backup.sql

# Restore database
docker exec -i monarc-db mysql -u monarc -p monarc_cli < backup.sql

# Note: If using custom passwords, adjust the commands accordingly
```

## ☸️ Kubernetes Deployment

### 1st Deployment: MariaDB

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: monarc-db
spec:
  selector:
    matchLabels:
      app: monarc-db
  template:
    metadata:
      labels:
        app: monarc-db
    spec:
      containers:
      - name: monarc-db
        image: mariadb:lts
        env:
        - name: MARIADB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: monarc-secrets
              key: db-root-password
        - name: MARIADB_DATABASE
          value: "monarc_cli"
        - name: MARIADB_USER
          value: "monarc"
        - name: MARIADB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: monarc-secrets
              key: db-password
        volumeMounts:
        - name: db-storage
          mountPath: /var/lib/mysql
        ports:
        - containerPort: 3306
      volumes:
      - name: db-storage
        persistentVolumeClaim:
          claimName: monarc-db-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: monarc-db
spec:
  selector:
    app: monarc-db
  ports:
  - port: 3306
    targetPort: 3306
  type: ClusterIP
```

### 2nd Deployment: Monarc

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: monarc
spec:
  selector:
    matchLabels:
      app: monarc
  template:
    metadata:
      labels:
        app: monarc
    spec:
      containers:
      - name: monarc
        image: mitexleo/monarc.lu:2.13.3-p6
        env:
        - name: DB_HOST
          value: "monarc-db"
        - name: DB_USER
          value: "monarc"
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: monarc-secrets
              key: db-password
        - name: DB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: monarc-secrets
              key: db-root-password
        - name: DB_CLI_NAME
          value: "monarc_cli"
        - name: DB_COMMON_NAME
          value: "monarc_common"
        - name: APPLICATION_ENV
          value: "production"
        volumeMounts:
        - name: monarc-data
          mountPath: /var/lib/monarc/fo/data
        ports:
        - containerPort: 80
      volumes:
      - name: monarc-data
        persistentVolumeClaim:
          claimName: monarc-data-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: monarc
spec:
  selector:
    app: monarc
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

### Create Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: monarc-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  rules:
  - host: monarc.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: monarc
            port:
              number: 80
```

## 🔧 Troubleshooting

### Common Issues

1. **Database connection errors:**
   ```bash
   # Check database logs
   docker-compose logs db
   
   # Test database connectivity
   docker exec monarc-db mysql -u monarc -p monarc_cli
   ```

2. **Application won't start:**
   ```bash
   # Check Monarc logs
   docker-compose logs monarc
   
   # Restart services
   docker-compose restart
   ```

3. **Permission errors:**
   ```bash
   # Fix permissions on data volumes
   docker-compose down
   sudo chown -R 1000:1000 ./volumes/
   docker-compose up -d
   ```

4. **Out of memory:**
   - Increase Docker memory allocation
   - Add swap space if running on limited resources
   - Consider using `--memory` limits in production

### Health Checks
- Monarc health: `curl http://localhost:8086/`
- Database health: `docker exec monarc-db mysqladmin ping`

## 📊 Version Information

### Current Version: 2.13.3-p6
- **Monarc**: 2.13.3-p6
- **PHP**: 8.1
- **Web Server**: Apache 2.4
- **Database**: MariaDB (via separate container)

### New Features in 2.13.3
- Possibility to reset 2FA of users by the admin account
- Global analyses stats limited to users with CEO (global statistics) role
- Import capability for risks with mode (generic | specific) property on BackOffice
- Various bug fixes and improvements

### Upgrade Notes
If upgrading from Monarc v2.12.5 or earlier, PHP 8.x is required (already satisfied by this image).

### Troubleshooting

#### Missing Environment Variables
If you see warnings about missing environment variables, the system will use default values:
- `DB_PASSWORD`: `monarc_password`
- `DB_ROOT_PASSWORD`: `root_password`

To fix, either:
1. Create a `.env` file with custom passwords
2. Or set environment variables directly in the shell before running docker-compose

#### MariaDB vs MySQL Environment Variables
The MariaDB container uses `MARIADB_*` environment variables, not `MYSQL_*`. The docker-compose.yml handles this automatically.

## 🤝 Contributing

This Docker image is community-maintained. Contributions are welcome!

1. Fork the repository
2. Create a feature branch
3. Submit a pull request

## 📄 License

This Docker image is provided under the same license as Monarc itself (AGPL-3.0-or-later). Monarc is developed by the Luxembourg House of Cybersecurity.

## 🔗 Links

- [Monarc Official Website](https://www.monarc.lu)
- [Monarc Documentation](https://www.monarc.lu/documentation/)
- [Docker Hub Repository](https://hub.docker.com/r/mitexleo/monarc.lu)
- [Monarc GitHub](https://github.com/monarc-project)

## ⚠️ Disclaimer

This is not an official NC3 Luxembourg product. Use at your own risk in production environments. Always perform security assessments and follow your organization's security policies.