---
title: Homepage Background Issue - Solution
date: 2026-04-07T06:11:00.000Z
---

**Problem:** Background image not displaying on homepage behind reverse proxy (NPM)

**Root Cause:** 
- Homepage container serves static files from `/app/public/images/`
- NPM proxy forwards all requests to container, but images were in wrong location
- Initial volume mount was `./config:/app/config`, not `/app/public/images/`

**Solution:**

### 1. Update docker-compose.yaml

Add volume mount for images folder:

```yaml
services:
  homepage:
    image: ghcr.io/gethomepage/homepage:latest
    container_name: homepage
    ports:
      - 127.0.0.1:3002:3000
    volumes:
      - ./config:/app/config
      - ./config/images:/app/public/images
    environment:
      HOMEPAGE_ALLOWED_HOSTS: homepage.hyserver.homelab
      PUID: $PUID
      PGID: $PGID
    networks:
      - homepage_default
      - uptime-kuma_kuma_network

networks:
  homepage_default:
    external: true
  uptime-kuma_kuma_network:
    external: true
```

### 2. Create images folder and copy image

```bash
mkdir -p config/images
cp config/xxx.jpg config/images/
chmod 777 config/images/xxx.jpg
```

### 3. Update settings.yaml

Use relative path (not absolute):

```yaml
background:
  image: /images/xxx.jpg
  blur: sm
  opacity: 50
```

### 4. Restart container

```bash
podman compose up -d
# or
podman restart homepage
```

**Key Points:**

- Homepage serves static files from `/app/public/` directory
- Local image path must be relative: `/images/xxx.jpg` (not `/app/config/xxx.jpg`)
- NPM proxy rule already exists for homepage.hyserver.homelab → port 3002
- All paths (including `/images/*`) are automatically proxied
