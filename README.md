# Traefik v3.5 with Let's Encrypt

A production-ready Traefik v3.5 setup with automatic Let's Encrypt SSL certificates using command line arguments and environment variables.

## 📋 Prerequisites

- Docker and Docker Compose installed
- A domain name pointing to your server's IP address
- Ports 80 and 443 open on your firewall

## 🚀 Quick Start

### 1. Create Environment File

Copy the example environment file and update it with your values:

```bash
cp .env.example .env
```

Edit `.env` and configure:

```bash
# Your email for Let's Encrypt notifications
ACME_EMAIL=your-email@example.com

# Domain for Traefik dashboard
TRAEFIK_DOMAIN=traefik.yourdomain.com

# Basic Auth for dashboard
TRAEFIK_AUTH=admin:$$apr1$$8EVjn/nj$$GiLUZqcbueTFeD23SuB6x0
```

### 2. Generate Secure Password (Optional)

Generate a secure password for the dashboard:

```bash
# Using htpasswd with bcrypt (install with: apt-get install apache2-utils)
echo $(htpasswd -nbB admin your-secure-password)
```

Copy the output and update `TRAEFIK_AUTH` in `.env`.

**Note:** Use `$$` to escape the `$` character in .env files.

### 3. Create Docker Network

```bash
docker network create proxy
```

### 4. Start Traefik

```bash
docker compose up -d
```

### 5. Verify Installation

- Check logs: `docker logs traefik`
- Access dashboard: `https://traefik.yourdomain.com`
- The `acme.json` file will be created in `traefik-data/letsencrypt/`

## 📦 Example Application

The `example-app` folder contains a simple whoami application demonstrating Traefik integration with HTTPS.

### Deploy Example App

1. Update the domain in `example-app/docker-compose.yml`:
   ```yaml
   - traefik.http.routers.whoami.rule=Host(`whoami.yourdomain.com`)
   ```

2. Start the example application:
   ```bash
   cd example-app
   docker compose up -d
   ```

3. Access your application with HTTPS:
   - Whoami: `https://whoami.yourdomain.com`

## 🔧 Configuration Overview

### Traefik Features Enabled

- ✅ **HTTP to HTTPS redirect** - All traffic automatically redirected to HTTPS
- ✅ **Let's Encrypt SSL** - Automatic certificate generation and renewal
- ✅ **Dashboard** - Web UI for monitoring (password protected)
- ✅ **Docker provider** - Automatic service discovery
- ✅ **Access logs** - Request logging enabled

### Environment Variables

Configuration is managed through `.env` file:

- `ACME_EMAIL` - Your email for Let's Encrypt notifications
- `TRAEFIK_DOMAIN` - Domain for accessing the Traefik dashboard
- `TRAEFIK_AUTH` - Basic authentication credentials (user:hashedpassword)

### Command Line Arguments Used

```yaml
# API and Dashboard
- --api.dashboard=true
- --api.insecure=false

# Entrypoints (ports)
- --entrypoints.web.address=:80
- --entrypoints.websecure.address=:443
- --entrypoints.web.http.redirections.entrypoint.to=websecure

# Docker Provider
- --providers.docker=true
- --providers.docker.exposedbydefault=false
- --providers.docker.network=proxy

# Let's Encrypt (uses ${ACME_EMAIL} from .env)
- --certificatesresolvers.letsencrypt.acme.email=${ACME_EMAIL}
- --certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json
- --certificatesresolvers.letsencrypt.acme.httpchallenge=true
- --certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web
```

## 🏷️ Adding New Services

To add a new service to Traefik with HTTPS, add these labels to your docker-compose service:

```yaml
services:
  your-app:
    image: your-image
    networks:
      - proxy
    labels:
      # Enable Traefik
      - traefik.enable=true
      
      # Router configuration with HTTPS
      # REMEMBER: Change 'your-app' in router name to match your service name
      # REMEMBER: Update 'app.yourdomain.com' to your actual domain
      - traefik.http.routers.your-app.rule=Host(`app.yourdomain.com`)
      - traefik.http.routers.your-app.entrypoints=websecure
      - traefik.http.routers.your-app.tls=true
      - traefik.http.routers.your-app.tls.certresolver=letsencrypt
      
      # Service configuration (specify port)
      # REMEMBER: Change 'your-app' in service name to match your router name
      - traefik.http.services.your-app.loadbalancer.server.port=80

networks:
  proxy:
    external: true
```

## 🔒 Security Features

### Basic Authentication

The dashboard is protected with basic auth. Generate credentials:

```bash
echo $(htpasswd -nbB username password)
```

### Security Headers

Add security headers to your services using middleware:

```yaml
labels:
  - traefik.http.routers.app.middlewares=security-headers
  - traefik.http.middlewares.security-headers.headers.customResponseHeaders.X-Frame-Options=SAMEORIGIN
  - traefik.http.middlewares.security-headers.headers.customResponseHeaders.X-Content-Type-Options=nosniff
  - traefik.http.middlewares.security-headers.headers.customResponseHeaders.X-XSS-Protection=1; mode=block
```

## 📁 File Structure

```
traefik/
├── docker-compose.yml              # Main Traefik configuration
├── .env.example                    # Environment variables template
├── .env                            # Your configuration (not in git)
├── traefik-data/
│   └── letsencrypt/
│       └── acme.json              # SSL certificates (auto-generated)
├── example-app/
│   └── docker-compose.yml         # Example whoami app
└── README.md
```

## 🐛 Troubleshooting

### Certificate Issues

1. **Check logs:** `docker logs traefik`
2. **Verify DNS:** Ensure your domain points to the server
3. **Port 80 must be accessible** for HTTP challenge
4. **Rate limits:** Let's Encrypt has rate limits (50 certs/week per domain)

### Testing with Staging

For testing, use Let's Encrypt staging server to avoid rate limits:

```yaml
- --certificatesresolvers.letsencrypt.acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory
```

### Permissions on acme.json

If you encounter permission issues, set proper permissions:

```bash
chmod 600 traefik-data/letsencrypt/acme.json
```

## 📚 Additional Resources

- [Traefik Documentation](https://doc.traefik.io/traefik/)
- [Let's Encrypt Documentation](https://letsencrypt.org/docs/)
- [Docker Provider](https://doc.traefik.io/traefik/providers/docker/)

## 🔄 Updating Traefik

```bash
docker compose pull
docker compose up -d
```

## 📝 Notes

- The `acme.json` file contains your SSL certificates - **back it up!**
- Certificates auto-renew 30 days before expiration
- The proxy network must exist before starting services
- All services using Traefik must be on the `proxy` network

## ⚠️ Important

- Create `.env` from `.env.example` and configure all variables
- Replace example domains with your actual domains
- Generate a secure password for `TRAEFIK_AUTH` before deploying to production
- The setup uses HTTP challenge - ensure port 80 is accessible for Let's Encrypt validation
- All services will automatically get HTTPS certificates
