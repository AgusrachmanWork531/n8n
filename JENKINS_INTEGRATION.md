# Panduan Auto Deployment dengan Jenkins - Complete Guide

## Daftar Isi
1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Step 1: Konfigurasi General](#step-1-konfigurasi-general)
4. [Step 2: Source Code Management](#step-2-source-code-management)
5. [Step 3: Triggers (Auto Build)](#step-3-triggers-auto-build)
6. [Step 4: Environment Variables](#step-4-environment-variables)
7. [Step 5: Build Steps](#step-5-build-steps)
8. [Step 6: Post-build Actions](#step-6-post-build-actions)
9. [Setup GitHub Webhook](#setup-github-webhook)
10. [Contoh Implementasi untuk Berbagai Stack](#contoh-implementasi-untuk-berbagai-stack)
11. [Troubleshooting](#troubleshooting)

---

## Overview

Panduan ini akan mengajarkan cara setup **Auto Deployment** menggunakan Jenkins yang akan:
- ✅ Otomatis pull code dari GitHub ketika ada push
- ✅ Build aplikasi secara otomatis
- ✅ Deploy ke server production
- ✅ Kirim notifikasi status deployment

**Berdasarkan project:** `web-agusrachman.my.id`

---

## Prerequisites

### 1. Jenkins Sudah Terinstall
```bash
# Verifikasi Jenkins running
docker ps | grep jenkins
```

### 2. Install Required Plugins
Dashboard → Manage Jenkins → Manage Plugins → Available

**Required Plugins:**
- ✅ Git Plugin
- ✅ GitHub Plugin
- ✅ Pipeline Plugin
- ✅ SSH Agent Plugin (untuk deployment)
- ✅ Credentials Plugin
- ✅ Publish Over SSH (opsional)
- ✅ Email Extension Plugin (untuk notifikasi)
- ✅ Slack Notification (opsional)

### 3. Setup Credentials
Dashboard → Manage Jenkins → Manage Credentials

**A. GitHub Credentials (untuk private repo)**
```
Kind: Username with password
Username: your-github-username
Password: your-github-personal-access-token
ID: github-credentials
Description: GitHub Access Token
```

**B. SSH Credentials (untuk deployment ke server)**
```
Kind: SSH Username with private key
Username: root (atau user deployment)
Private Key: Enter directly (paste SSH private key)
ID: deployment-server-ssh
Description: Production Server SSH Key
```

### 4. Generate GitHub Personal Access Token
1. GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Generate new token (classic)
3. Select scopes:
   - ✅ repo (full control)
   - ✅ admin:repo_hook (untuk webhook)
4. Copy token dan simpan di Jenkins Credentials

---

## Step 1: Konfigurasi General

### Create New Item
1. Dashboard → New Item
2. Enter name: `web-agusrachman-deployment`
3. Pilih: **Freestyle project**
4. Klik OK

### General Configuration

```
✅ Enabled: Yes (pastikan job aktif)

Description:
Auto deployment untuk web-agusrachman.my.id
- Auto pull dari GitHub
- Build aplikasi
- Deploy ke production server

✅ GitHub project: [CHECKED]
Project url: https://github.com/your-username/web-agusrachman

☐ Discard old builds (optional - untuk save space)
  Strategy: Log Rotation
  Days to keep builds: 30
  Max # of builds to keep: 10

☐ This project is parameterized (optional - untuk custom deployment)

☐ Throttle builds (optional - limit concurrent builds)

☐ Execute concurrent builds if necessary
```

**Screenshot Reference:** Image 1

---

## Step 2: Source Code Management

### Configure Git Repository

```
Source Code Management:
○ None
● Git

Repositories:
  Repository URL: https://github.com/your-username/web-agusrachman.git
  
  Credentials: 
    - Pilih: github-credentials (yang sudah dibuat sebelumnya)
    - Atau klik [+ Add] untuk add credential baru
```

### Advanced Git Configuration

```
Advanced (klik untuk expand):

  Branches to build:
    Branch Specifier: */master
    (atau */main, atau */development sesuai branch Anda)
  
  ✅ Add Branch:
    Branch Specifier #1: */master
    Branch Specifier #2: */development (opsional)
  
  Repository browser: (Auto)
  
  Additional Behaviours: (klik Add)
    - Clean before checkout (recommended)
    - Wipe out repository & force clone (untuk fresh clone)
    - Check out to specific local branch
      Branch name: master
```

**Screenshot Reference:** Image 2 & 3

---

## Step 3: Triggers (Auto Build)

### Configure Build Triggers

```
Build Triggers:

☐ Trigger builds remotely (e.g., from scripts)
  (untuk trigger via API)

☐ Build after other projects are built
  (jika depend on project lain)

☐ Build periodically
  (gunakan jika ingin scheduled build)
  Schedule: H 2 * * * (setiap hari jam 2 pagi)

✅ GitHub hook trigger for GITScm polling [CHECKED]
  (PENTING: Ini yang membuat auto deployment dari GitHub)

☐ Poll SCM
  (alternative jika webhook tidak bisa digunakan)
  Schedule: H/5 * * * * (check setiap 5 menit)
```

**Penjelasan:**
- **GitHub hook trigger**: Jenkins akan receive webhook dari GitHub ketika ada push
- **Poll SCM**: Jenkins akan check GitHub secara periodic (gunakan jika webhook gagal)

**Screenshot Reference:** Image 4

---

## Step 4: Environment Variables

### Configure Build Environment

```
Build Environment:

☐ Delete workspace before build starts
  (clean workspace setiap build)

☐ Use secret text(s) or file(s)
  (untuk inject credentials as environment variables)

☐ Add timestamps to the Console Output
  (recommended untuk debugging)

☐ Inspect build log for published build scans

☐ Terminate a build if it's stuck
  Timeout strategy:
    ○ Absolute
    ● Elastic
    Timeout minutes: 20
    Timeout actions: Abort the build

☐ With Ant
  (jika menggunakan Apache Ant)
```

### Setup Environment Variables (Opsional)

Jika aplikasi butuh environment variables:

```
☐ Use secret text(s) or file(s) [CHECK THIS]

Bindings:
  + Add → Secret text
    Variable: DB_PASSWORD
    Credentials: db-password-credential
  
  + Add → Secret text
    Variable: API_KEY
    Credentials: api-key-credential
```

**Screenshot Reference:** Image 4 & 5

---

## Step 5: Build Steps

### Add Build Step
Klik: **Add build step** → Pilih sesuai kebutuhan

### A. Untuk Aplikasi Node.js/React/Vue

```
Add build step → Execute shell

Command:
#!/bin/bash
set -e  # Exit on error

echo "========================================="
echo "Starting Build Process"
echo "========================================="

# Check Node version
node --version
npm --version

# Install dependencies
echo "Installing dependencies..."
npm install

# Run tests (optional)
echo "Running tests..."
npm test

# Build application
echo "Building application..."
npm run build

# Create deployment package
echo "Creating deployment package..."
tar -czf dist.tar.gz dist/

echo "========================================="
echo "Build completed successfully!"
echo "========================================="
```

### B. Untuk Aplikasi PHP/Laravel

```
Add build step → Execute shell

Command:
#!/bin/bash
set -e

echo "========================================="
echo "Starting Laravel Build Process"
echo "========================================="

# Check PHP version
php --version
composer --version

# Install dependencies
echo "Installing Composer dependencies..."
composer install --no-dev --optimize-autoloader

# Clear cache
php artisan cache:clear
php artisan config:clear
php artisan route:clear
php artisan view:clear

# Run migrations (optional - berhati-hati di production!)
# php artisan migrate --force

# Optimize
php artisan config:cache
php artisan route:cache
php artisan view:cache

echo "========================================="
echo "Build completed successfully!"
echo "========================================="
```

### C. Untuk Aplikasi Python/Django

```
Add build step → Execute shell

Command:
#!/bin/bash
set -e

echo "========================================="
echo "Starting Django Build Process"
echo "========================================="

# Create virtual environment
python3 -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Run tests
python manage.py test

# Collect static files
python manage.py collectstatic --noinput

# Run migrations (optional)
# python manage.py migrate

echo "========================================="
echo "Build completed successfully!"
echo "========================================="
```

### D. Untuk Docker Build

```
Add build step → Execute shell

Command:
#!/bin/bash
set -e

echo "========================================="
echo "Building Docker Image"
echo "========================================="

# Define variables
IMAGE_NAME="web-agusrachman"
IMAGE_TAG="latest"
DOCKER_REGISTRY="your-registry.com"

# Build Docker image
docker build -t $IMAGE_NAME:$IMAGE_TAG .

# Tag for registry
docker tag $IMAGE_NAME:$IMAGE_TAG $DOCKER_REGISTRY/$IMAGE_NAME:$IMAGE_TAG

# Push to registry
docker push $DOCKER_REGISTRY/$IMAGE_NAME:$IMAGE_TAG

echo "========================================="
echo "Docker image built and pushed successfully!"
echo "========================================="
```

**Screenshot Reference:** Image 5

---

## Step 6: Post-build Actions

### Deploy ke Production Server

```
Add post-build action → Send build artifacts over SSH

SSH Server:
  Name: production-server (harus dikonfigurasi dulu di Manage Jenkins)

Transfers:
  Source files: dist.tar.gz  (atau sesuai build output)
  Remove prefix: (kosongkan)
  Remote directory: /tmp/
  Exec command:
    #!/bin/bash
    set -e
    
    echo "========================================="
    echo "Starting Deployment"
    echo "========================================="
    
    # Define variables
    APP_DIR="/var/www/web-agusrachman"
    BACKUP_DIR="/var/backups/web-agusrachman"
    
    # Create backup
    echo "Creating backup..."
    mkdir -p $BACKUP_DIR
    tar -czf $BACKUP_DIR/backup_$(date +%Y%m%d_%H%M%S).tar.gz -C $APP_DIR .
    
    # Extract new files
    echo "Extracting new files..."
    cd $APP_DIR
    tar -xzf /tmp/dist.tar.gz --strip-components=1
    
    # Set permissions
    echo "Setting permissions..."
    chown -R www-data:www-data $APP_DIR
    chmod -R 755 $APP_DIR
    
    # Restart services
    echo "Restarting services..."
    systemctl reload nginx
    # systemctl restart php8.1-fpm  # untuk PHP
    # pm2 restart all  # untuk Node.js
    
    # Cleanup
    rm -f /tmp/dist.tar.gz
    
    echo "========================================="
    echo "Deployment completed successfully!"
    echo "========================================="
```

### Alternative: Deploy via SSH Script

```
Add build step → Execute shell script on remote host using ssh

SSH site: production-server
Command:
  #!/bin/bash
  set -e
  
  cd /var/www/web-agusrachman
  git pull origin master
  npm install
  npm run build
  pm2 restart web-agusrachman
```

### Email Notification

```
Add post-build action → Editable Email Notification

Project Recipient List: your-email@example.com
Project Reply-To List: jenkins@yourdomain.com

Advanced Settings:
  Triggers:
    - Success
    - Failure
    - Unstable
  
  Subject:
    Jenkins Build $BUILD_STATUS - $PROJECT_NAME #$BUILD_NUMBER
  
  Content:
    Build URL: $BUILD_URL
    Build Status: $BUILD_STATUS
    Build Duration: $BUILD_DURATION
    
    Console Output:
    ${BUILD_LOG, maxLines=50}
```

### Slack Notification (Opsional)

```
Add post-build action → Slack Notifications

✅ Notify Success
✅ Notify Failure
✅ Notify Unstable
☐ Notify Repeated Failure

Advanced:
  Team Domain: your-workspace
  Integration Token: (dari Slack App)
  Channel: #deployments
  Custom message:
    Deployment ${BUILD_STATUS} - web-agusrachman.my.id
    Build: #${BUILD_NUMBER}
    Duration: ${BUILD_DURATION}
```

**Screenshot Reference:** Image 5

---

## Setup GitHub Webhook

### 1. Configure Jenkins URL
Dashboard → Manage Jenkins → Configure System

```
Jenkins Location:
  Jenkins URL: https://jenkins.yourdomain.com/
  (atau http://YOUR_VPS_IP:8080/)
```

### 2. GitHub Repository Settings

1. Buka repository: https://github.com/your-username/web-agusrachman
2. Settings → Webhooks → Add webhook

```
Payload URL: https://jenkins.yourdomain.com/github-webhook/
  (PENTING: harus diakhiri dengan /github-webhook/)

Content type: application/json

Secret: (kosongkan atau isi jika ingin extra security)

Which events would you like to trigger this webhook?
  ● Just the push event (untuk auto deployment on push)
  ○ Let me select individual events
    ✅ Pushes
    ✅ Pull requests
    ✅ Releases

✅ Active (pastikan checked)
```

3. Klik **Add webhook**

### 3. Test Webhook

```bash
# Push dummy commit untuk test
git commit --allow-empty -m "Test Jenkins webhook"
git push origin master
```

Check:
- GitHub → Repository → Settings → Webhooks → Recent Deliveries
- Jenkins → Job → Build History

### 4. Troubleshooting Webhook

**Problem: Webhook gagal connect**

```bash
# Check Jenkins dapat diakses dari internet
curl -I https://jenkins.yourdomain.com/github-webhook/

# Check firewall
sudo ufw status

# Allow webhook (jika pakai Cloudflare atau proxy)
# Whitelist GitHub webhook IPs
```

**Problem: Webhook return 403**

Solution:
```
Dashboard → Manage Jenkins → Configure Global Security
Authentication:
  Uncheck: Prevent Cross Site Request Forgery exploits (temporary)
  Atau: Configure GitHub webhooks dengan token
```

---

## Contoh Implementasi untuk Berbagai Stack

### 1. React Application dengan Docker

**Dockerfile:**
```dockerfile
# Build stage
FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Production stage
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**Jenkins Build Steps:**
```bash
#!/bin/bash
set -e

# Build Docker image
docker build -t web-agusrachman:latest .

# Stop old container
docker stop web-agusrachman || true
docker rm web-agusrachman || true

# Run new container
docker run -d \
  --name web-agusrachman \
  --restart unless-stopped \
  -p 80:80 \
  web-agusrachman:latest

# Cleanup old images
docker image prune -f
```

---

### 2. Laravel Application

**Jenkins Build Steps:**
```bash
#!/bin/bash
set -e

echo "Deploying Laravel Application..."

# Variables
APP_DIR="/var/www/web-agusrachman"
BACKUP_DIR="/var/backups/web-agusrachman"

# Create backup
mkdir -p $BACKUP_DIR
tar -czf $BACKUP_DIR/backup_$(date +%Y%m%d_%H%M%S).tar.gz \
  -C $APP_DIR \
  --exclude='storage' \
  --exclude='vendor' \
  .

# Pull latest code
cd $APP_DIR
git pull origin master

# Install dependencies
composer install --no-dev --optimize-autoloader --no-interaction

# Clear and cache config
php artisan down
php artisan config:clear
php artisan cache:clear
php artisan view:clear
php artisan route:clear

# Run migrations
php artisan migrate --force

# Optimize
php artisan config:cache
php artisan route:cache
php artisan view:cache

# Set permissions
chown -R www-data:www-data $APP_DIR/storage $APP_DIR/bootstrap/cache
chmod -R 775 $APP_DIR/storage $APP_DIR/bootstrap/cache

# Restart PHP-FPM
systemctl reload php8.1-fpm

# Bring application up
php artisan up

echo "Deployment completed successfully!"
```

---

### 3. Node.js/Express API

**Jenkins Build Steps:**
```bash
#!/bin/bash
set -e

echo "Deploying Node.js Application..."

APP_DIR="/var/www/web-agusrachman-api"
cd $APP_DIR

# Pull latest code
git pull origin master

# Install dependencies
npm install --production

# Run database migrations (jika ada)
npm run migrate

# Restart application with PM2
pm2 restart web-agusrachman-api

# Check application status
pm2 status
pm2 logs web-agusrachman-api --lines 20

echo "Deployment completed!"
```

---

### 4. Static Website (HTML/CSS/JS)

**Jenkins Build Steps:**
```bash
#!/bin/bash
set -e

echo "Deploying Static Website..."

# Build (jika ada build process)
npm install
npm run build

# Sync to web server
rsync -avz --delete \
  dist/ \
  root@production-server:/var/www/web-agusrachman/

# Set permissions
ssh root@production-server "chown -R www-data:www-data /var/www/web-agusrachman"

# Reload Nginx
ssh root@production-server "systemctl reload nginx"

echo "Deployment completed!"
```

---

### 5. Python/Django Application

**Jenkins Build Steps:**
```bash
#!/bin/bash
set -e

echo "Deploying Django Application..."

APP_DIR="/var/www/web-agusrachman-django"
cd $APP_DIR

# Activate virtual environment
source venv/bin/activate

# Pull latest code
git pull origin master

# Install dependencies
pip install -r requirements.txt

# Run migrations
python manage.py migrate

# Collect static files
python manage.py collectstatic --noinput

# Restart Gunicorn
systemctl restart gunicorn

# Restart Nginx
systemctl reload nginx

echo "Deployment completed!"
```

---

## Complete Jenkins Pipeline Script

Alternatif menggunakan **Pipeline** (lebih modern):

```groovy
pipeline {
    agent any
    
    environment {
        APP_NAME = 'web-agusrachman'
        APP_DIR = '/var/www/web-agusrachman'
        GIT_BRANCH = 'master'
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out code from GitHub...'
                git branch: "${GIT_BRANCH}", 
                    credentialsId: 'github-credentials',
                    url: 'https://github.com/your-username/web-agusrachman.git'
            }
        }
        
        stage('Install Dependencies') {
            steps {
                echo 'Installing dependencies...'
                sh 'npm install'
            }
        }
        
        stage('Run Tests') {
            steps {
                echo 'Running tests...'
                sh 'npm test'
            }
        }
        
        stage('Build') {
            steps {
                echo 'Building application...'
                sh 'npm run build'
            }
        }
        
        stage('Deploy') {
            steps {
                echo 'Deploying to production...'
                sh '''
                    # Create backup
                    ssh root@production-server "mkdir -p /var/backups/${APP_NAME}"
                    ssh root@production-server "tar -czf /var/backups/${APP_NAME}/backup_$(date +%Y%m%d_%H%M%S).tar.gz -C ${APP_DIR} ."
                    
                    # Deploy new version
                    rsync -avz --delete dist/ root@production-server:${APP_DIR}/
                    
                    # Restart services
                    ssh root@production-server "pm2 restart ${APP_NAME}"
                '''
            }
        }
    }
    
    post {
        success {
            echo 'Deployment successful!'
            emailext (
                subject: "✅ Deployment Success - ${APP_NAME} #${BUILD_NUMBER}",
                body: "Deployment completed successfully!\n\nBuild URL: ${BUILD_URL}",
                to: 'your-email@example.com'
            )
        }
        failure {
            echo 'Deployment failed!'
            emailext (
                subject: "❌ Deployment Failed - ${APP_NAME} #${BUILD_NUMBER}",
                body: "Deployment failed!\n\nBuild URL: ${BUILD_URL}\n\nConsole: ${BUILD_URL}console",
                to: 'your-email@example.com'
            )
        }
    }
}
```

---

## Monitoring dan Logs

### 1. View Build Logs
```
Dashboard → Job → Build History → #Build Number → Console Output
```

### 2. Real-time Monitoring
```bash
# Jenkins logs
docker logs -f jenkins

# Application logs
tail -f /var/www/web-agusrachman/storage/logs/laravel.log  # Laravel
pm2 logs web-agusrachman  # Node.js
tail -f /var/log/nginx/access.log  # Nginx
```

### 3. Build Status Badge

Add ke README.md:
```markdown
[![Build Status](https://jenkins.yourdomain.com/buildStatus/icon?job=web-agusrachman-deployment)](https://jenkins.yourdomain.com/job/web-agusrachman-deployment/)
```

---

## Security Best Practices

### 1. Secure Credentials
```bash
# JANGAN hardcode credentials di script
# Gunakan Jenkins Credentials Store

# Bad ❌
DB_PASSWORD="mypassword123"

# Good ✅
withCredentials([string(credentialsId: 'db-password', variable: 'DB_PASSWORD')]) {
    sh 'echo $DB_PASSWORD'
}
```

### 2. Restricted Deployment
```bash
# Deploy hanya dari branch tertentu
if [ "$GIT_BRANCH" != "master" ]; then
    echo "Deployment only allowed from master branch"
    exit 1
fi
```

### 3. Health Check
```bash
# Check aplikasi health setelah deployment
echo "Running health check..."
sleep 5
response=$(curl -s -o /dev/null -w "%{http_code}" http://web-agusrachman.my.id/health)

if [ $response -ne 200 ]; then
    echo "Health check failed! Rolling back..."
    # Rollback logic here
    exit 1
fi

echo "Health check passed!"
```

---

## Troubleshooting

### Problem 1: Build gagal - Permission denied

**Solusi:**
```bash
# Fix Jenkins user permissions
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins

# Or jalankan command dengan sudo
sudo npm install
```

### Problem 2: Git pull failed - Authentication required

**Solusi:**
```
1. Verify GitHub credentials di Jenkins
2. Test credentials:
   Dashboard → Credentials → Test Connection

3. Atau gunakan SSH key instead of HTTPS:
   git remote set-url origin git@github.com:username/repo.git
```

### Problem 3: Deployment timeout

**Solusi:**
```
Build Environment → Terminate a build if it's stuck
Increase timeout: 30 minutes
```

### Problem 4: Webhook tidak trigger build

**Solusi:**
```bash
# Check Jenkins logs
docker logs jenkins | grep webhook

# Test webhook manually
curl -X POST https://jenkins.yourdomain.com/github-webhook/

# Verify GitHub webhook delivery
GitHub → Settings → Webhooks → Recent Deliveries
```

### Problem 5: Build success tapi aplikasi error

**Solusi:**
```bash
# Add health check di post-build
curl -f http://web-agusrachman.my.id || exit 1

# Check application logs
ssh root@server "tail -100 /var/www/app/logs/error.log"
```

---

## Checklist Deployment

### Pre-deployment
- [ ] Jenkins job dikonfigurasi dengan benar
- [ ] GitHub webhook active dan tested
- [ ] Credentials tersimpan di Jenkins
- [ ] SSH access ke production server working
- [ ] Backup script configured

### During deployment
- [ ] Build berhasil tanpa error
- [ ] Tests passed
- [ ] Dependencies terinstall
- [ ] Files ter-deploy dengan benar

### Post-deployment
- [ ] Health check passed
- [ ] Application accessible via browser
- [ ] Logs tidak ada error critical
- [ ] SSL certificate valid
- [ ] Email notification received
- [ ] Backup created successfully

---

## Next Steps

1. **Setup Staging Environment**
   - Buat job terpisah untuk staging
   - Test deployment di staging sebelum production

2. **Implement Blue-Green Deployment**
   - Zero-downtime deployment
   - Easy rollback

3. **Add Monitoring**
   - Integrate dengan monitoring tools (Prometheus, Grafana)
   - Setup alerts untuk failed deployments

4. **Automated Testing**
   - Unit tests
   - Integration tests
   - E2E tests

5. **Documentation**
   - Document deployment process
   - Create runbook untuk incident

---

## Kesimpulan

Dengan setup ini, Anda memiliki:
- ✅ Auto deployment ketika push ke GitHub
- ✅ Build automation
- ✅ Testing integration
- ✅ Secure deployment dengan SSH
- ✅ Automatic backup sebelum deployment
- ✅ Notification pada success/failure
- ✅ Easy rollback capability

**Happy Deploying! 🚀**

---

## Resources

- [Jenkins Documentation](https://www.jenkins.io/doc/)
- [GitHub Webhooks Guide](https://docs.github.com/en/webhooks)
- [Jenkins Pipeline Syntax](https://www.jenkins.io/doc/book/pipeline/syntax/)
- [Best Practices for CI/CD](https://www.jenkins.io/doc/book/pipeline/best-practices/)

---

**Created:** November 2025  
**Version:** 1.0  
**Project:** web-agusrachman.my.id  
**Author:** DevOps Documentation Team
