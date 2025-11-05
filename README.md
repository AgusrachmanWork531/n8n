# SETUP n8n Workflow pada server VPS dengan containerize

## Prerequisites

- VPS dengan minimal 2GB RAM
- Docker dan Docker Compose terinstall
- Domain (opsional, tapi direkomendasikan)
- Basic knowledge Linux command line

## 1. Persiapan Server

```bash
# Update sistem
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Install Docker Compose
sudo apt install docker-compose -y

# Tambahkan user ke docker group (agar tidak perlu sudo)
sudo usermod -aG docker $USER
```

## 2. Setup n8n dengan Docker Compose

Buat direktori untuk n8n:

```bash
mkdir -p ~/n8n-docker
cd ~/n8n-docker
```

Buat file `docker-compose.yml`:

```yaml
version: '3.8'

services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      # Basic Configuration
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=${N8N_BASIC_AUTH_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_BASIC_AUTH_PASSWORD}
      
      # Host Configuration
      - N8N_HOST=${N8N_HOST}
      - N8N_PORT=5678
      - N8N_PROTOCOL=${N8N_PROTOCOL}
      - WEBHOOK_URL=${WEBHOOK_URL}
      
      # Timezone
      - GENERIC_TIMEZONE=Asia/Jakarta
      - TZ=Asia/Jakarta
      
      # Execution Configuration
      - EXECUTIONS_DATA_SAVE_ON_ERROR=all
      - EXECUTIONS_DATA_SAVE_ON_SUCCESS=all
      - EXECUTIONS_DATA_SAVE_MANUAL_EXECUTIONS=true
      
      # Database (PostgreSQL recommended untuk production)
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
    volumes:
      - n8n_data:/home/node/.n8n
      - ./local-files:/files
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - n8n-network

  postgres:
    image: postgres:15-alpine
    container_name: n8n-postgres
    restart: unless-stopped
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - n8n-network

  # Nginx Reverse Proxy (Optional tapi recommended)
  nginx:
    image: nginx:alpine
    container_name: n8n-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
      - certbot_data:/var/www/certbot
    depends_on:
      - n8n
    networks:
      - n8n-network

volumes:
  n8n_data:
  postgres_data:
  certbot_data:

networks:
  n8n-network:
    driver: bridge
```

## 3. Setup Environment Variables

Buat file `.env`:

```bash
# n8n Configuration
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=your_secure_password_here

# Domain Configuration (gunakan IP jika tidak ada domain)
N8N_HOST=n8n.yourdomain.com
N8N_PROTOCOL=https
WEBHOOK_URL=https://n8n.yourdomain.com/

# PostgreSQL Configuration
POSTGRES_USER=n8n
POSTGRES_PASSWORD=your_postgres_password_here
POSTGRES_DB=n8n
```

## 4. Setup Nginx Reverse Proxy

Buat file `nginx.conf`:

```nginx
events {
    worker_connections 1024;
}

http {
    upstream n8n {
        server n8n:5678;
    }

    server {
        listen 80;
        server_name n8n.yourdomain.com;

        location /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }

        location / {
            return 301 https://$host$request_uri;
        }
    }

    server {
        listen 443 ssl http2;
        server_name n8n.yourdomain.com;

        ssl_certificate /etc/nginx/ssl/fullchain.pem;
        ssl_certificate_key /etc/nginx/ssl/privkey.pem;

        client_max_body_size 50M;

        location / {
            proxy_pass http://n8n;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            # WebSocket support
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            
            # Timeouts
            proxy_connect_timeout 7d;
            proxy_send_timeout 7d;
            proxy_read_timeout 7d;
        }
    }
}
```

## 5. Setup SSL Certificate (dengan Let's Encrypt)

```bash
# Install certbot
sudo apt install certbot -y

# Generate certificate
sudo certbot certonly --standalone -d n8n.yourdomain.com

# Copy certificates
sudo mkdir -p ./ssl
sudo cp /etc/letsencrypt/live/n8n.yourdomain.com/fullchain.pem ./ssl/
sudo cp /etc/letsencrypt/live/n8n.yourdomain.com/privkey.pem ./ssl/
sudo chown -R $USER:$USER ./ssl
```

## 6. Jalankan n8n

```bash
# Start containers
docker-compose up -d

# Check logs
docker-compose logs -f n8n

# Check status
docker-compose ps
```

## 7. Backup Strategy

Buat script backup `backup.sh`:

```bash
#!/bin/bash

BACKUP_DIR="/backup/n8n"
DATE=$(date +%Y%m%d_%H%M%S)

# Create backup directory
mkdir -p $BACKUP_DIR

# Backup PostgreSQL
docker exec n8n-postgres pg_dump -U n8n n8n > $BACKUP_DIR/n8n_db_$DATE.sql

# Backup n8n data
docker run --rm \
  -v n8n-docker_n8n_data:/data \
  -v $BACKUP_DIR:/backup \
  alpine tar czf /backup/n8n_data_$DATE.tar.gz -C /data .

# Keep only last 7 days backups
find $BACKUP_DIR -name "*.sql" -mtime +7 -delete
find $BACKUP_DIR -name "*.tar.gz" -mtime +7 -delete

echo "Backup completed: $DATE"
```

Setup cron untuk auto backup:

```bash
chmod +x backup.sh
crontab -e

# Tambahkan baris ini untuk backup harian jam 2 pagi
0 2 * * * /home/your_user/n8n-docker/backup.sh >> /var/log/n8n-backup.log 2>&1
```

## 8. Monitoring & Maintenance

### Health Check Script

Buat `healthcheck.sh`:

```bash
#!/bin/bash

# Check if n8n is running
if ! docker ps | grep -q n8n; then
    echo "n8n container is not running!"
    docker-compose restart n8n
fi

# Check if postgres is running
if ! docker ps | grep -q n8n-postgres; then
    echo "PostgreSQL container is not running!"
    docker-compose restart postgres
fi
```

### Update n8n

```bash
cd ~/n8n-docker

# Pull latest image
docker-compose pull n8n

# Restart with new image
docker-compose up -d n8n
```

## 9. Security Best Practices

### 1. Firewall Configuration

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

### 2. Additional Security Measures

- **Gunakan strong password** untuk N8N_BASIC_AUTH_PASSWORD
- **Enable 2FA** jika tersedia
- **Regular updates** untuk Docker images
- **Monitor logs** secara berkala
- **Limit resource** dengan Docker:

```yaml
# Tambahkan di service n8n dalam docker-compose.yml
deploy:
  resources:
    limits:
      cpus: '2'
      memory: 2G
    reservations:
      cpus: '1'
      memory: 1G
```

## 10. Troubleshooting

### Check logs
```bash
docker-compose logs -f n8n
docker-compose logs -f postgres
```

### Restart services
```bash
docker-compose restart
```

### Clean restart
```bash
docker-compose down
docker-compose up -d
```

### Check database connection
```bash
docker exec -it n8n-postgres psql -U n8n -d n8n
```

### Common Issues

**Issue: Container won't start**
```bash
# Check logs
docker-compose logs n8n

# Check if ports are already in use
sudo netstat -tulpn | grep :5678
```

**Issue: Database connection failed**
```bash
# Check if postgres is healthy
docker-compose ps

# Restart postgres
docker-compose restart postgres
```

**Issue: SSL certificate errors**
```bash
# Renew certificates
sudo certbot renew

# Copy new certificates
sudo cp /etc/letsencrypt/live/n8n.yourdomain.com/fullchain.pem ./ssl/
sudo cp /etc/letsencrypt/live/n8n.yourdomain.com/privkey.pem ./ssl/

# Restart nginx
docker-compose restart nginx
```

## 11. Struktur Direktori

```
~/n8n-docker/
├── docker-compose.yml
├── .env
├── nginx.conf
├── backup.sh
├── healthcheck.sh
├── ssl/
│   ├── fullchain.pem
│   └── privkey.pem
├── local-files/
│   └── (files yang dapat diakses oleh n8n)
└── README.md
```

## 12. Quick Commands Reference

```bash
# Start n8n
docker-compose up -d

# Stop n8n
docker-compose down

# Restart n8n
docker-compose restart n8n

# View logs
docker-compose logs -f

# Update n8n
docker-compose pull && docker-compose up -d

# Backup database
docker exec n8n-postgres pg_dump -U n8n n8n > backup.sql

# Restore database
docker exec -i n8n-postgres psql -U n8n -d n8n < backup.sql

# Access postgres shell
docker exec -it n8n-postgres psql -U n8n -d n8n

# Check container status
docker-compose ps

# Check resource usage
docker stats
```

## Kesimpulan

Setup ini memberikan:

- ✅ Containerized environment yang mudah di-manage
- ✅ PostgreSQL database untuk reliabilitas
- ✅ SSL/HTTPS untuk keamanan
- ✅ Reverse proxy dengan Nginx
- ✅ Automated backup
- ✅ Health monitoring
- ✅ Easy scaling dan update

## Untuk Production

Pertimbangkan juga:

- Load balancer jika traffic tinggi
- Redis untuk queue management
- Monitoring tools (Prometheus, Grafana)
- CI/CD pipeline untuk workflow deployment
- Multiple replicas untuk high availability
- Log aggregation (ELK Stack)

## Support & Resources

- [n8n Documentation](https://docs.n8n.io/)
- [Docker Documentation](https://docs.docker.com/)
- [Let's Encrypt](https://letsencrypt.org/)
- [n8n Community Forum](https://community.n8n.io/)

## License

This documentation is provided as-is for educational purposes.

---

**Created by:** Your Name  
**Last Updated:** November 2025  
**Version:** 1.0.0
