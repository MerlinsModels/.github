# Munera Backend API Deployment Documentation

## Project Overview

Munera is a fitness tracking application integrated with Strava, deployed across staging and production environments using Cloudflare Tunnels for secure API exposure.

## Architecture

### Environment Overview

| Environment | Server | Frontend URL | API URL | Frontend Port | Backend Port | Tunnel ID |
|------------|--------|--------------|---------|---------------|--------------|-----------|
| **Staging** | XPS Server | https://staging.munera.app | https://staging-api.munera.app | 3010 | 8010 | eeec6f03-16cf-421d-a885-02d5071469fc |
| **Production** | Auros Server | https://munera.app | https://api.munera.app | 3010 | 8000 | 7800be99-6cfa-4025-958a-80a6f32e7aa6 |

## DNS Configuration

### Domain Provider
- **Registrar**: IONOS
- **DNS Provider**: Cloudflare
- **Nameservers**: 
  - keira.ns.cloudflare.com
  - ryan.ns.cloudflare.com

### DNS Records

| Type | Name | Target | Proxied | Environment |
|------|------|--------|---------|-------------|
| CNAME | @ (munera.app) | 7800be99-6cfa-4025-958a-80a6f32e7aa6.cfargotunnel.com | ✅ | Production |
| CNAME | api | 7800be99-6cfa-4025-958a-80a6f32e7aa6.cfargotunnel.com | ✅ | Production |
| CNAME | staging | eeec6f03-16cf-421d-a885-02d5071469fc.cfargotunnel.com | ✅ | Staging |
| CNAME | staging-api | eeec6f03-16cf-421d-a885-02d5071469fc.cfargotunnel.com | ✅ | Staging |

## SSL Certificates

### Cloudflare Origin Certificates
- **Provider**: Cloudflare Origin Certificates
- **Validity**: 15 years
- **Certificate Path**: `/etc/ssl/cloudflare/munera.app.pem`
- **Private Key Path**: `/etc/ssl/cloudflare/munera.app.key`
- **Permissions**: 
  - Certificate: 644
  - Private Key: 600

### Certificate Coverage
The wildcard certificate covers:
- `*.munera.app`
- `*.staging.munera.app`
- `munera.app`
- `staging.munera.app`
- `staging-api.munera.app` (covered by *.munera.app)
- `api.munera.app` (covered by *.munera.app)

## Nginx Configuration

### Staging Server (XPS Server)

#### API Configuration (`/etc/nginx/sites-available/staging-api.munera.app.conf`)
```nginx
server {
    listen 80;
    listen 443 ssl http2;
    server_name staging-api.munera.app;

    # SSL Configuration
    ssl_certificate /etc/ssl/cloudflare/munera.app.pem;
    ssl_certificate_key /etc/ssl/cloudflare/munera.app.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # Debug header
    add_header X-Server-Name "staging-api.munera.app" always;

    # Proxy settings
    location / {
        proxy_pass http://localhost:8010;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;
        proxy_read_timeout 300s;
        proxy_connect_timeout 75s;
        proxy_cache_bypass $http_upgrade;
    }
}
```

#### Frontend Configuration (`/etc/nginx/sites-available/staging.munera.app.conf`)
```nginx
server {
    listen 80;
    listen 443 ssl http2;
    server_name staging.munera.app;

    ssl_certificate /etc/ssl/cloudflare/munera.app.pem;
    ssl_certificate_key /etc/ssl/cloudflare/munera.app.key;
    ssl_protocols TLSv1.2 TLSv1.3;

    add_header X-Server-Name "staging.munera.app" always;

    location / {
        proxy_pass http://localhost:3010;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;
        proxy_read_timeout 300s;
        proxy_cache_bypass $http_upgrade;
    }
}
```

### Production Server (Auros Server)

#### API Configuration (`/etc/nginx/sites-available/api.munera.app.conf`)
```nginx
server {
    listen 80;
    listen 443 ssl http2;
    server_name api.munera.app;

    ssl_certificate /etc/ssl/cloudflare/munera.app.pem;
    ssl_certificate_key /etc/ssl/cloudflare/munera.app.key;
    ssl_protocols TLSv1.2 TLSv1.3;

    add_header X-Server-Name "api.munera.app" always;

    location / {
        proxy_pass http://localhost:8000;
        # Same proxy headers as staging
    }
}
```

#### Frontend Configuration (`/etc/nginx/sites-available/munera.app.conf`)
```nginx
server {
    listen 80;
    listen 443 ssl http2;
    server_name munera.app www.munera.app;

    ssl_certificate /etc/ssl/cloudflare/munera.app.pem;
    ssl_certificate_key /etc/ssl/cloudflare/munera.app.key;
    ssl_protocols TLSv1.2 TLSv1.3;

    add_header X-Server-Name "munera.app" always;

    location / {
        proxy_pass http://localhost:3010;
        # Same proxy headers as staging
    }
}
```

## Cloudflare Tunnel Configuration

### Staging Server (`/etc/cloudflared/config.yml`)
```yaml
tunnel: eeec6f03-16cf-421d-a885-02d5071469fc
credentials-file: /home/largelingo/.cloudflared/eeec6f03-16cf-421d-a885-02d5071469fc.json

ingress:
  - hostname: bartstolarek.com
    service: http://localhost:80
  - hostname: www.bartstolarek.com
    service: http://localhost:80
  - hostname: warren.bartstolarek.com
    service: http://localhost:80
  - hostname: carbie.app
    service: http://localhost:80
  - hostname: staging.munera.app
    service: http://localhost:80
  - hostname: staging-api.munera.app
    service: http://localhost:80
  - service: http_status:404
```

### Production Server (`/etc/cloudflared/config.yml`)
```yaml
tunnel: 7800be99-6cfa-4025-958a-80a6f32e7aa6
credentials-file: /home/bart/.cloudflared/7800be99-6cfa-4025-958a-80a6f32e7aa6.json

ingress:
  - hostname: test.bartstolarek.com
    service: http://localhost:80
  - hostname: munera.app
    service: http://localhost:80
  - hostname: www.munera.app
    service: http://localhost:80
  - hostname: api.munera.app
    service: http://localhost:80
  - service: http_status:404
```

## Docker Container Configuration

### Staging Server Containers

| Container | Name | Port Mapping | Internal Port | Host Port |
|-----------|------|--------------|---------------|-----------|
| Backend | munera_backend_staging | 8010:8000 | 8000 | 8010 |
| Frontend | munera_frontend_staging | 3010:3010 | 3010 | 3010 |
| Database | munera_postgres_staging | 5433:5432 | 5432 | 5433 |

### Production Server Containers

| Container | Name | Port Mapping | Internal Port | Host Port |
|-----------|------|--------------|---------------|-----------|
| Backend | munera_backend_prod | 8000:8000 | 8000 | 8000 |
| Frontend | munera_frontend_prod | 3010:3010 | 3010 | 3010 |

## Environment Configuration

### Staging Backend (`.env.staging`)
```env
ENVIRONMENT=staging
PORT=8010
DATABASE_URL=postgresql://postgres:password@postgres:5432/munera_db
FRONTEND_URL=https://staging.munera.app
BACKEND_URL=https://staging-api.munera.app
STRAVA_CLIENT_ID=158320
STRAVA_CLIENT_SECRET=[secret]
STRAVA_WEBHOOK_VERIFY_TOKEN=[token]
```

### Staging Frontend (`.env.staging`)
```env
NEXT_PUBLIC_ENVIRONMENT=staging
PORT=3010
NEXT_PUBLIC_API_URL=https://staging-api.munera.app
```

## Cloudflare Settings

- **SSL/TLS Mode**: Full
- **Always Use HTTPS**: Enabled
- **Minimum TLS Version**: TLS 1.0

### Page Rules

| URL Pattern | Settings |
|------------|----------|
| `staging-api.munera.app/*` | Cache Level: Bypass, Browser Cache TTL: Respect Existing Headers |
| `api.munera.app/*` | Cache Level: Bypass, Browser Cache TTL: Respect Existing Headers |

## Strava Webhook Endpoints

- **Staging**: https://staging-api.munera.app/webhooks/strava
- **Production**: https://api.munera.app/webhooks/strava

## Deployment Commands

### SSL Certificate Setup
```bash
# Create directory
sudo mkdir -p /etc/ssl/cloudflare

# Add certificate and key
sudo nano /etc/ssl/cloudflare/munera.app.pem
sudo nano /etc/ssl/cloudflare/munera.app.key

# Set permissions
sudo chmod 600 /etc/ssl/cloudflare/munera.app.key
sudo chmod 644 /etc/ssl/cloudflare/munera.app.pem
```

### Nginx Configuration
```bash
# Create configuration files
sudo nano /etc/nginx/sites-available/staging-api.munera.app.conf
sudo nano /etc/nginx/sites-available/staging.munera.app.conf

# Enable sites
sudo ln -s /etc/nginx/sites-available/staging-api.munera.app.conf /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/staging.munera.app.conf /etc/nginx/sites-enabled/

# Test and reload
sudo nginx -t
sudo systemctl reload nginx
```

### Cloudflare Tunnel Management
```bash
# Restart tunnel
sudo systemctl restart cloudflared

# Check status
sudo systemctl status cloudflared

# View logs
sudo journalctl -u cloudflared -f
```

## Testing and Verification

### Quick Health Checks
```bash
# Test DNS resolution
nslookup staging-api.munera.app

# Test SSL handshake
openssl s_client -connect staging-api.munera.app:443 -servername staging-api.munera.app < /dev/null

# Test API endpoint
curl -I https://staging-api.munera.app/docs

# Test frontend
curl -I https://staging.munera.app
```

### Verification Checklist
- [ ] DNS resolves correctly to Cloudflare IPs
- [ ] SSL certificate is valid and covers all domains
- [ ] API documentation loads at `/docs`
- [ ] Frontend connects to API successfully
- [ ] Strava webhooks are received and processed
- [ ] Database connections are working

## Troubleshooting

### Common Issues

#### SSL Handshake Failures
- **Issue**: ERR_SSL_VERSION_OR_CIPHER_MISMATCH on sub-subdomains
- **Solution**: Use single-level subdomains (e.g., `staging-api` instead of `api.staging`)

#### Port Mapping Confusion
- **Remember**: Docker internal port != Host port
- **Staging Backend**: Container listens on 8000, mapped to host 8010
- **Production Backend**: Container listens on 8000, mapped to host 8000

#### Nginx Configuration Conflicts
```bash
# Check for conflicts
grep -r "server_name" /etc/nginx/sites-enabled/

# Remove duplicates
sudo rm /etc/nginx/sites-enabled/conflicting-config
```

#### Cloudflare Tunnel Issues
```bash
# Check tunnel connections
cloudflared tunnel list

# Restart tunnel service
sudo systemctl restart cloudflared

# Check logs for errors
sudo journalctl -u cloudflared -n 50
```

## Historical Issues and Resolutions

### Domain Restructuring (September 2025)
- **Problem**: `api.staging.munera.app` (sub-subdomain) experienced SSL handshake failures with Cloudflare
- **Root Cause**: Cloudflare edge servers handle sub-subdomains differently during SSL negotiation
- **Solution**: Restructured to `staging-api.munera.app` (single-level subdomain)
- **Result**: SSL handshake issues resolved, all endpoints working correctly

## Maintenance Notes

### Regular Tasks
1. Monitor SSL certificate expiration (15-year validity)
2. Keep Cloudflare tunnel updated
3. Review and rotate API keys and secrets
4. Monitor Docker container logs for errors
5. Check Nginx access and error logs

### Update Procedures
```bash
# Update Cloudflared
curl -L --output cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared.deb
sudo systemctl restart cloudflared

# Restart all services
sudo systemctl restart nginx
sudo systemctl restart cloudflared
docker-compose restart
```

## Security Considerations

1. SSL certificates use Cloudflare Origin Certificates (15-year validity)
2. All traffic proxied through Cloudflare for DDoS protection
3. Sensitive environment variables stored in `.env` files with 600 permissions
4. API endpoints use CORS configuration appropriate for production
5. Database credentials isolated per environment

## Contact and Resources

- **Cloudflare Dashboard**: https://dash.cloudflare.com
- **Strava App Settings**: https://www.strava.com/settings/api
- **Server Access**: SSH to respective servers (XPS for staging, Auros for production)

---

*Last Updated: September 2025*
*Document Version: 2.0 - Post domain restructuring*
