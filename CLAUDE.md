# docker-qbittorrent

This repository contains a Dockerfile (and supporting files) that builds a multi-platform Docker image for [qBittorrent](https://www.qbittorrent.org/).

## Project Architecture

### Multi-Stage Build
The Dockerfile uses a multi-stage build process:
1. **libtorrent-src** - Clones libtorrent source
2. **qbittorrent-src** - Clones qBittorrent source
3. **build** - Compiles both from source using cross-compilation (xx toolkit)
4. **final** - Creates minimal Alpine-based runtime image

### Key Version Variables
- `QBITTORRENT_VERSION` - Currently 5.1.2 (Dockerfile:3)
- `LIBTORRENT_VERSION` - Currently 2.0.11 (Dockerfile:4)
- `ALPINE_VERSION` - Currently 3.22 (Dockerfile:6)

Update these ARGs in the Dockerfile to change versions.

### Supported Platforms
Multi-platform builds via docker-bake.hcl:
- linux/amd64
- linux/arm/v6
- linux/arm/v7
- linux/arm64

## Key Files

- **Dockerfile** - Multi-stage build for qBittorrent-nox (no GUI)
- **entrypoint.sh** - Initialization script that:
  - Sets up user/group IDs (PUID/PGID)
  - Resolves WAN IP for tracker reporting
  - Creates directory structure in /data
  - Generates default qBittorrent.conf
  - Fixes permissions
- **docker-bake.hcl** - Build configuration for docker buildx
- **test/compose.yml** - Test setup
- **examples/** - Example Docker Compose configurations

## Building and Testing

### Local Build
```bash
# Build for local platform
docker buildx bake

# Build all platforms
docker buildx bake image-all
```

### Testing Changes
```bash
# Start test container
cd test && docker compose up -d

# Check logs
docker compose logs -f

# Test API (healthcheck)
docker compose exec qbittorrent curl http://127.0.0.1:8080/api/v2/app/version

# Cleanup
docker compose down
```

## Common Tasks

### Updating qBittorrent or libtorrent Version
1. Update `QBITTORRENT_VERSION` or `LIBTORRENT_VERSION` in Dockerfile
2. Build and test locally
3. Check GitHub releases for compatibility:
   - qBittorrent: https://github.com/qbittorrent/qBittorrent/releases
   - libtorrent: https://github.com/arvidn/libtorrent/releases

### Modifying Entrypoint Behavior
- Edit entrypoint.sh for initialization logic
- Shell script uses `/bin/sh` (not bash)
- Always test permission handling with different PUID/PGID values

### Adding New Environment Variables
1. Add to Dockerfile ENV section (Dockerfile:87-91)
2. Handle in entrypoint.sh
3. Document in README.md Environment variables section

## Code Conventions

### Shell Scripts
- Use POSIX-compliant `/bin/sh` syntax (not bash)
- Quote variables: `"${VAR}"`
- Use `set -ex` for debugging in heredocs

### Dockerfile
- Follow multi-stage pattern
- Use heredocs (`<<EOT`) for complex RUN commands
- Minimize layers in final image
- Always run as non-root (qbittorrent user)

## Testing Checklist

When making changes, verify:
- [ ] Build succeeds for all platforms (docker buildx bake image-all)
- [ ] Container starts successfully
- [ ] Healthcheck passes (curl API endpoint)
- [ ] Permissions are correct (test with custom PUID/PGID)
- [ ] qBittorrent WebUI accessible on port 8080
- [ ] Default credentials work (admin/adminadmin)
- [ ] Log files appear in /var/log/qbittorrent

## CI/CD

- **build.yml** - Builds and pushes images to Docker Hub and GHCR
- **test.yml** - Runs test suite
- Builds triggered on:
  - Push to master
  - Tags
  - Pull requests
  - Schedule (every 6 days for cache refresh)

## Important Notes

- The image runs qBittorrent-nox (headless, WebUI only)
- WAN IP auto-detected via OpenDNS unless WAN_IP set
- Data persisted in /data volume
- Default WebUI port is 8080 (configurable via WEBUI_PORT)
- Container uses yasu for privilege dropping
