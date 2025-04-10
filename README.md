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
   sudo rm -rf /home/largelingo/actions-runner/_work/[service-name]
   sudo mkdir -p /home/largelingo/actions-runner/_work/[service-name]
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
