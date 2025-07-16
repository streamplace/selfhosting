# Streamplace Server Setup

This directory contains configuration files for running your own Streamplace server with a hybrid Docker + systemd approach.

> [!NOTE]
> This setup isn't designed for production use, but rather for personal/small-scale use.
> No warranty, etc, etc. If you read this, please don't sue me.

## Overview

This setup runs:
- **Streamplace** as a systemd service on the host (for simplicity and performance)
  - In Docker, you'd need to passthrough a lot of ports for WebRTC
- **Caddy** in Docker for reverse proxy and automatic TLS certificates
- **MistServer** in Docker for RTMP streaming support

## Prerequisites

- Docker and Docker Compose
- systemd (Linux)
- A domain name pointed to your server
- Cloudflare DNS (for automatic certificate management)

## Quick Start

1. **Clone/copy these files** to your server
2. **Edit configuration files** (see Configuration section below)
3. **Install Streamplace binary** on your host system
4. **Start the services:**
   ```bash
   # Start Docker services
   docker-compose up -d

   # Install and start Streamplace systemd service
   sudo cp streamplace.service /etc/systemd/system/
   sudo systemctl daemon-reload
   sudo systemctl enable streamplace
   sudo systemctl start streamplace
   ```

## Configuration

### 1. Environment Variables

Edit `docker-compose.yml` and update:
- `DOMAIN`: Your domain name
- `EMAIL`: Your email for Let's Encrypt certificates
- `CLOUDFLARE_API_TOKEN`: Your Cloudflare API token with DNS edit permissions

### 2. Streamplace Service

Edit `streamplace.service` and update:
- `--public-host`: Your domain name
- `--app-bundle-id`: Your app bundle ID (optional)

### 3. Caddy Configuration

Edit `Caddyfile` and update:
- `stream.example.com`: Replace with your actual domain
- `email youremail@domain.co.uk`: Replace with your email

## TLS Certificate Management

The setup handles TLS certificates automatically, but you have options for how Streamplace accesses them:

### Option 1: Use Caddy-generated certificates (Recommended)

Caddy will automatically generate and manage certificates. To use them with Streamplace:

```bash
# Create the TLS directory in Streamplace data directory
sudo mkdir -p /var/lib/streamplace/tls

# Copy certificates from Caddy (you may need to set up a script to do this automatically)
sudo cp ./caddy_data/certificates/acme-v02.api.letsencrypt.org-directory/your-domain/your-domain.crt /var/lib/streamplace/tls/tls.crt
sudo cp ./caddy_data/certificates/acme-v02.api.letsencrypt.org-directory/your-domain/your-domain.key /var/lib/streamplace/tls/tls.key
```

```bash
# In your streamplace.service file, add:
--tls-cert /path/to/your/caddy_data/certificates/acme-v02.api.letsencrypt.org-directory/your-domain/your-domain.crt \
--tls-key /path/to/your/caddy_data/certificates/acme-v02.api.letsencrypt.org-directory/your-domain/your-domain.key \
```

## Service Management

### Docker Services

```bash
# Start all Docker services
docker-compose up -d

# View logs
docker-compose logs -f caddy
docker-compose logs -f streamplace-mistserver

# Stop services
docker-compose down
```

### Streamplace Service

```bash
# Start/stop/restart
sudo systemctl start streamplace
sudo systemctl stop streamplace
sudo systemctl restart streamplace

# View status and logs
sudo systemctl status streamplace
sudo journalctl -u streamplace -f
```

## Ports

- **80/443**: HTTP/HTTPS (Caddy)
- **31935**: RTMP (MistServer)
- **38443**: Streamplace HTTPS (internal, proxied by Caddy)
- **39090**: Streamplace internal API (localhost only)

## Troubleshooting

### Common Issues

1. **Certificates not working**: Check that your domain DNS points to your server and Cloudflare API token has correct permissions

2. **RTMP not working**: Ensure MistServer is running and can reach the Streamplace internal API at `host.docker.internal:39090`

3. **Service won't start**: Check logs with `journalctl -u streamplace -f` and ensure the Streamplace binary is installed at `/usr/bin/streamplace`

### Logs

- **Caddy logs**: `docker-compose logs caddy`
- **MistServer logs**: `docker-compose logs streamplace-mistserver`
- **Streamplace logs**: `sudo journalctl -u streamplace -f`

### Testing

1. **Web interface**: Visit `https://your-domain.com`
2. **RTMP streaming**: Use OBS or similar with `rtmp://your-domain.com:31935/live/your-stream-key`
3. **API health**: `curl http://localhost:39090/health` (from host)

## File Structure

```
streamplace/
├── README.md                 # This file
├── docker-compose.yml        # Docker services configuration
├── Caddyfile                 # Caddy reverse proxy configuration
├── streamplace.service       # systemd service file
├── Dockerfile.caddy          # Custom Caddy build with Cloudflare DNS
├── Dockerfile.mistserver     # MistServer container
├── mistserver.json           # MistServer configuration
└── caddy_data/              # Caddy data directory (created by Docker)
    └── certificates/        # Let's Encrypt certificates
```

## Security Notes

- The setup uses `tls_insecure_skip_verify` in Caddy config for the internal connection to Streamplace. This is acceptable for localhost connections but should be reviewed for production use.
- Ensure your firewall only allows necessary ports (80, 443, 1935)
  - Or at least restrict access to the RTMP port (31935) and internal API port (39090) to known IPs
  - These known IPs (at least for the internal API port) should INCLUDE your Mistserver container IP
- Consider using Tailscale or similar for remote management access
- Regularly update all components (Streamplace, Caddy, MistServer)

## Advanced Configuration

### Custom Streamplace Flags

Common flags you might want to change in `streamplace.service`:

- `--rate-limit-per-second 10`: Rate limiting
- `--rate-limit-burst 20`: Burst rate limiting
- `--secure`: Enable secure mode
- `--app-bundle-id tv.your.app`: Custom app bundle ID
- the `--tls-cert` and `--tls-key` flags for custom TLS certificates, mentioned above

### MistServer Configuration

Edit `mistserver.json` to customize:
- RTMP port (currently 31935)
- HTTP port (currently 28080)
- Stream processing settings
- Authentication settings
