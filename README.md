
# Testing stufff

To test:
`pytest`

To test with coverage, install extension Coverage Gutters, set in the command pallete the setting to watch and run
`pytest --cov=app --cov-report=xml`

# Fixing GitHub Actions Runner Permission Errors


If you encounter permission errors like "Error: EACCES: permission denied" during GitHub Actions workflows, use these commands to reset permissions:


**RUN EACH COMMAND-SEPERATED BY COMMENT- SEPARATELY**
```bash
# Stop the GitHub Actions runner
sudo systemctl stop actions.runner.LargeLingo.xpsserver.service

# Completely reset the _work directory with wide-open permissions
sudo rm -rf /home/largelingo/actions-runner/_work
sudo mkdir -p /home/largelingo/actions-runner/_work
sudo chmod -R 777 /home/largelingo/actions-runner/_work
sudo chown -R largelingo:largelingo /home/largelingo/actions-runner/_work

# Reset permissions on the service directories too
sudo rm -rf /home/largelingo/largelingo_services/*/logs
sudo mkdir -p /home/largelingo/largelingo_services/authentication/logs
sudo mkdir -p /home/largelingo/largelingo_services/apigateway/logs
sudo mkdir -p /home/largelingo/largelingo_services/inference/logs

# Set very permissive permissions on all service directories
sudo chmod -R 777 /home/largelingo/largelingo_services
sudo chown -R largelingo:largelingo /home/largelingo/largelingo_services

# Protect .env files though
sudo find /home/largelingo/largelingo_services -name ".env" -exec chmod 600 {} \;

# Restart the GitHub Actions runner
sudo systemctl start actions.runner.LargeLingo.xpsserver.service
```
# Permission update

```
sudo chown -R 1001:1001 /home/largelingo/
```


# Adding a New Service to the Deployment Pipeline

This document outlines the steps needed to add a new service to our GitHub Actions deployment pipeline.

## Prerequisites

- Debian-based server with sudo access
- GitHub Actions runner already configured
- Docker and Docker Compose installed
- Working largelingo user with sudo permissions

## Step-by-Step Process

1. **Stop the GitHub Actions runner**

   ```bash
   sudo su - largelingo
   sudo systemctl stop actions.runner.LargeLingo.xpsserver.service
   ```

2. **Set up workspace directories for the new service**

   Replace `[new-service-name]` with your service name (e.g., "userservice"):

   ```bash
   # Create and configure workspace directory
   sudo rm -rf /home/largelingo/actions-runner/_work/[new-service-name]
   sudo mkdir -p /home/largelingo/actions-runner/_work/[new-service-name]
   sudo chown -R largelingo:largelingo /home/largelingo/actions-runner/_work/
   ```

3. **Create service directory and initial environment file**

   ```bash
   # Create service directory 
   mkdir -p ~/largelingo_services/[new-service-name]
   
   # Create logs directory with proper permissions
   mkdir -p ~/largelingo_services/[new-service-name]/logs
   chmod -R 777 ~/largelingo_services/[new-service-name]/logs
   
   # Create initial .env file (customize as needed for your service)
   cat > ~/largelingo_services/[new-service-name]/.env << EOF
   PROJECT_NAME=[new-service-name]
   VERSION=1.0.0
   ENV=production
   SECRET_KEY=your_secure_production_key
   LOG_LEVEL=info
   # Add any additional environment variables your service needs
   EOF
   
   # Set appropriate permissions
   chmod 600 ~/largelingo_services/[new-service-name]/.env
   ```

4. **Ensure Docker network exists**

   ```bash
   docker network create largelingo-network 2>/dev/null || true
   ```

5. **Restart the GitHub Actions runner**

   ```bash
   sudo systemctl start actions.runner.LargeLingo.xpsserver.service
   ```

## Creating the GitHub Actions Workflow File

Create a `.github/workflows/deploy.yml` file in your new service repository:

```yaml
name: Test and Deploy [New Service Name]

on:
  push:
    branches:
      - main

jobs:
  test:
    runs-on: self-hosted
    container:
      image: python:3.10-slim
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Install dependencies
        run: |
          pip install --upgrade pip
          pip install -r requirements.txt

      - name: Create .env for Pytest
        run: |
          echo "PROJECT_NAME=[new-service-name]" >> .env
          # Add other environment variables needed for testing
          
      - name: Run Pytest
        run: pytest

  deploy:
    needs: test
    runs-on: self-hosted
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Sync repository to production directory
        run: |
          # Get service name from GitHub repository
          SERVICE_NAME=$(echo $GITHUB_REPOSITORY | cut -d'/' -f2)
          
          # Backup existing .env file
          if [ -f "/home/largelingo/largelingo_services/$SERVICE_NAME/.env" ]; then
            cp /home/largelingo/largelingo_services/$SERVICE_NAME/.env /tmp/${SERVICE_NAME}.env.bak
          fi
          
          # Ensure logs directory exists with proper permissions
          mkdir -p /home/largelingo/largelingo_services/$SERVICE_NAME/logs
          chmod -R 777 /home/largelingo/largelingo_services/$SERVICE_NAME/logs
          
          # Sync files excluding .env and logs
          rsync -av --delete --exclude='.env' --exclude='logs/' "$GITHUB_WORKSPACE/" "/home/largelingo/largelingo_services/$SERVICE_NAME/"
          
          # Restore .env from backup
          if [ -f "/tmp/${SERVICE_NAME}.env.bak" ]; then
            cp /tmp/${SERVICE_NAME}.env.bak /home/largelingo/largelingo_services/$SERVICE_NAME/.env
            rm /tmp/${SERVICE_NAME}.env.bak
          fi
          
      - name: Deploy Code (Docker Compose)
        run: |
          SERVICE_NAME=$(echo $GITHUB_REPOSITORY | cut -d'/' -f2)
          cd /home/largelingo/largelingo_services/$SERVICE_NAME
          docker compose up -d --build
```

## Docker Compose Template

Ensure your Docker Compose file uses the shared network:

```yaml
version: '3.8'

services:
  [service-name]:
    build: .
    container_name: largelingo-[service-name]
    ports:
      - "[port]:[container-port]"  # Adjust as needed
    env_file:
      - .env
    volumes:
      - ./logs:/app/logs
      - ./app:/app/app  # For hot reloading
    networks:
      - largelingo-network
    restart: unless-stopped
    
    # Add any other service-specific configuration here

networks:
  largelingo-network:
    external: true
```

## Troubleshooting

If you encounter permission issues after setup:

```bash
sudo su - largelingo
sudo systemctl stop actions.runner.LargeLingo.xpsserver.service
sudo rm -rf ~/actions-runner/_work/[service-name]
sudo mkdir -p ~/actions-runner/_work/[service-name]
sudo chown -R largelingo:largelingo ~/actions-runner/_work/
sudo systemctl start actions.runner.LargeLingo.xpsserver.service
```

For log file permission issues:

```bash
sudo rm -rf /home/largelingo/largelingo_services/[service-name]/logs
mkdir -p /home/largelingo/largelingo_services/[service-name]/logs
chmod -R 777 /home/largelingo/largelingo_services/[service-name]/logs
```

## Important Notes

- Always ensure the Docker network exists before deploying services that need to communicate with each other
- Keep .env files secure with appropriate permissions (600)
- Make logs directories writable by all users (777) to prevent permission issues
- The Docker Compose file should use the `largelingo-network` with `external: true`
- Each service repository should have its own `.github/workflows/deploy.yml` file
- Service communication within Docker happens using service names as hostnames


# AI Chat App Deployment Setup

## Overview

This document outlines the steps taken to deploy an AI Chat application with a Next.js frontend, API Gateway, and Inference service using Cloudflare for DNS and proxying.

## Architecture

- **Frontend**: Next.js application running on port 3000
- **API Gateway**: FastAPI service running on port 8000
- **Inference Service**: AI model service for text generation
- **Nginx**: Used as a reverse proxy to route traffic between services
- **Cloudflare**: Provides DNS, CDN, and Cloudflare Tunnels for secure access

## Setup Steps

### 1. Domain Registration and DNS Configuration

1. Registered the domain `bartstolarek.com` on Namecheap
2. Set up Cloudflare account and added the domain to Cloudflare
3. Updated nameservers on Namecheap to point to Cloudflare's nameservers
4. Configured DNS records in Cloudflare:
   - Added an A record for the root domain pointing to the server IP
   - Added a CNAME record for www pointing to the root domain

### 2. Cloudflare Tunnel Setup

1. Set up Cloudflare Tunnel (`bart-tunnel`) to securely connect the server to Cloudflare's network
2. The tunnel has ID: `7c1b8cd7-9b0f-4b3b-aaac-c808e23d0376`
3. Added a CNAME record pointing to the tunnel's address (`7c1b8cd7-9b0f-4b3b-aaac-c808e23d0376.cfargotunnel.com`)
4. Created a local configuration file for the tunnel

### 3. Server Setup on XPS Server

1. Running Debian 12 on XPS 13 9350 with Intel i5-6200U and 8GB RAM
2. Deployed services using Docker:
   - Frontend container on port 3000
   - API Gateway container on port 8000
   - Inference service as needed

### 4. Nginx Configuration

1. Installed Nginx: `sudo apt install nginx`
2. Disabled Apache (which was previously installed): `sudo systemctl stop apache2 && sudo systemctl disable apache2`
3. Created Nginx configuration directories:
   ```bash
   sudo mkdir -p /etc/nginx/sites-available
   sudo mkdir -p /etc/nginx/sites-enabled
   sudo mkdir -p /var/log/nginx
   ```
4. Created the main Nginx configuration file at `/etc/nginx/nginx.conf`:
   ```nginx
   user www-data;
   worker_processes auto;
   pid /run/nginx.pid;

   events {
       worker_connections 768;
   }

   http {
       sendfile on;
       tcp_nopush on;
       types_hash_max_size 2048;
       include /etc/nginx/mime.types;
       default_type application/octet-stream;
       
       access_log /var/log/nginx/access.log;
       error_log /var/log/nginx/error.log;
       
       include /etc/nginx/conf.d/*.conf;
       include /etc/nginx/sites-enabled/*;
   }
   ```

5. Created a site configuration at `/etc/nginx/sites-available/default`:
   ```nginx
   server {
       listen 80 default_server;
       listen [::]:80 default_server;
       
       server_name _;
       
       # Frontend Next.js application
       location / {
           proxy_pass http://localhost:3000;
           proxy_http_version 1.1;
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection 'upgrade';
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
       }
       
       # API Gateway service
       location /api/ {
           proxy_pass http://localhost:8000/api/;
           proxy_http_version 1.1;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
           
           # Extended timeout for AI model inference
           proxy_read_timeout 300s;
           proxy_connect_timeout 300s;
           proxy_send_timeout 300s;
       }
   }
   ```

6. Enabled the site with a symbolic link:
   ```bash
   sudo ln -s /etc/nginx/sites-available/default /etc/nginx/sites-enabled/
   ```

7. Started and enabled Nginx:
   ```bash
   sudo systemctl start nginx
   sudo systemctl enable nginx
   ```

### 5. Cloudflare Configuration

1. Set up a Page Rule for API paths to bypass cache:
   - URL pattern: `*bartstolarek.com/api/*`
   - Cache Level: Bypass

2. Added appropriate SSL/TLS settings in Cloudflare:
   - Set SSL/TLS encryption mode to "Full"
   - Enabled "Always Use HTTPS"

### 6. Frontend Configuration

1. Updated the Next.js application to use relative URLs for API calls:
   ```javascript
   // Changed from absolute URLs:
   // const response = await fetch(`${apiUrl}/api/v1/inference/infer`, {...});
   
   // To relative URLs:
   const response = await fetch(`/api/v1/inference/infer`, {...});
   ```

2. Environment variables setup:
   - Production: No need to set `NEXT_PUBLIC_API_URL` (using relative URLs)
   - Development: Set `NEXT_PUBLIC_API_URL` to point to API Gateway (e.g., `http://localhost:8000`)

### 7. API Gateway Configuration

1. Added CORS middleware to allow cross-origin requests:
   ```python
   app.add_middleware(
       CORSMiddleware,
       allow_origins=["*"],  # Restrict this in production
       allow_credentials=True,
       allow_methods=["*"],
       allow_headers=["*"],
   )
   ```

### 8. Troubleshooting

1. Identified and resolved a timeout issue with Cloudflare for long-running inference requests
2. Added appropriate headers and routing in Nginx for both frontend and API requests
3. Ensured Docker containers exposed the correct ports
4. Disabled Apache which was conflicting with Nginx

## Challenges and Solutions

### Cloudflare Timeout Issues

- **Problem**: Cloudflare has a 100-second timeout limit for requests on non-Enterprise plans, causing 524 errors for long-running inference operations.
- **Solution options**:
  1. Implement a job queue system in the API Gateway
  2. Use streaming responses
  3. Optimize inference to complete faster

### DNS and Routing

- **Problem**: Configuring proper DNS records to work with Cloudflare Tunnels.
- **Solution**: Used a CNAME record for www subdomain pointing to the main domain, and configured the tunnel to route traffic to Nginx.

### Secure Connections

- **Problem**: Setting up SSL/TLS with Cloudflare and Nginx.
- **Solution**: Used Cloudflare's Full SSL mode, letting Cloudflare handle the certificates while traffic between Cloudflare and the origin remained encrypted via the tunnel.

## Future Improvements

1. Implement a job queue system for handling long-running inference requests
2. Set up monitoring for the services
3. Add deployment automation
4. Implement caching for common requests
5. Set up automatic backups
