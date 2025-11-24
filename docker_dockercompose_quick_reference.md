# Docker & Docker Compose Quick Reference Guide

## Docker – Essentials

### Images
```bash
docker images
docker pull <image>
docker build -t <name>:tag .
docker rmi <image>
docker image prune
```

### Containers
```bash
docker ps
docker ps -a
docker run -d <image>
docker run -it <image> bash
docker stop <container>
docker start <container>
docker restart <container>
docker rm <container>
```

### Logs & Exec
```bash
docker logs <container>
docker logs -f <container>
docker exec -it <container> bash
```

### Inspect
```bash
docker inspect <container>
docker inspect <image>
```

### Networks
```bash
docker network ls
docker network inspect <network>
docker network create <name>
```

### Volumes
```bash
docker volume ls
docker volume inspect <volume>
```

---

## Docker Compose – Essentials

### Basic Commands
```bash
docker compose up
docker compose up -d
docker compose down
docker compose down --volumes
docker compose ps
docker compose logs
docker compose logs -f
docker compose exec <service> bash
docker compose restart <service>
```

### Rebuild & Refresh
```bash
docker compose build
docker compose build --no-cache
docker compose up --build
```

### Validate Files
```bash
docker compose config
docker compose config --services
docker compose config --volumes
```

### Stopping & Cleaning
```bash
docker compose stop
docker compose rm
```

---

## Useful One-Liners

### Remove all containers
```bash
docker rm -f $(docker ps -aq)
```

### Remove all images
```bash
docker rmi -f $(docker images -q)
```

### Remove unused stuff
```bash
docker system prune -a
```

