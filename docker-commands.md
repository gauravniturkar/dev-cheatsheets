# Docker Commands I Actually Use

Started keeping this after spending way too long googling the same flags over and over. It's grown into a fairly complete reference covering everything from day-to-day container management to the Compose and Dockerfile patterns I reach for in production.

Written the way I'd explain it to someone — not copy-pasted from docs.

---

## Table of Contents

- [Images](#images)
- [Running Containers](#running-containers)
- [Managing Containers](#managing-containers)
- [Debugging & Inspection](#debugging--inspection)
- [Volumes](#volumes)
- [Networks](#networks)
- [Registry — Push & Pull](#registry--push--pull)
- [Docker Compose](#docker-compose)
- [System Cleanup](#system-cleanup)
- [Dockerfile — Key Instructions](#dockerfile--key-instructions)
- [Pro Tips](#pro-tips)

---

## Images

Everything to do with building and managing images locally.

| Command | What it does |
|---|---|
| `docker pull nginx` | Downloads an image from Docker Hub. Pulls `latest` by default |
| `docker pull nginx:1.25` | Pull a specific version. Always pin versions in production — `latest` can break things quietly |
| `docker images` | Lists all images stored locally — name, tag, ID, size, when it was created |
| `docker build -t myapp:v1 .` | Build an image from the Dockerfile in the current directory. `-t` gives it a name and tag |
| `docker build --no-cache .` | Build from scratch, ignoring cached layers — use when you're getting stale/weird build results |
| `docker build -f Dockerfile.prod .` | Build using a specific Dockerfile. Handy when you have separate dev/prod/test files |
| `docker rmi image:tag` | Delete a local image. `-f` to force-remove even if containers reference it |
| `docker image prune -a` | Delete ALL unused images. Without `-a`, only removes dangling (untagged) ones |
| `docker tag app:v1 app:latest` | Create a new alias/tag for an existing image — no data duplication |
| `docker inspect image:tag` | Full JSON metadata about an image — layers, env vars, entrypoint, exposed ports |
| `docker history image:tag` | Shows every layer in an image and which Dockerfile command created it |
| `docker save -o out.tar img` | Export an image to a `.tar` file — useful for moving images without a registry |
| `docker load -i out.tar` | Import an image from a `.tar` file exported with `docker save` |

---

## Running Containers

The flags you'll actually use.

| Command | What it does |
|---|---|
| `docker run nginx` | Creates and starts a container. Downloads the image if it's not local. `Ctrl+C` to stop |
| `docker run -d nginx` | Detached mode — runs in the background, prints the container ID. Use this for servers |
| `docker run -it ubuntu bash` | Interactive + terminal. Opens a shell inside the container — like SSHing into it |
| `docker run --rm alpine sh` | Auto-removes the container when it stops. Perfect for one-off tasks |
| `docker run -p 8080:80 nginx` | Port mapping — `host:container`. Your machine's `8080` → container's `80` |
| `docker run -P nginx` | Publishes all exposed ports to random host ports. Use `docker port container` to see what was assigned |
| `docker run -v /host:/app img` | Bind mount — maps a host directory into the container. Changes reflect in both directions |
| `docker run -e DB_URL=... img` | Pass an environment variable into the container. `--env-file .env` passes a whole file at once |
| `docker run --name myapp img` | Give it a proper name instead of a random one — much easier to reference later |
| `docker run --network mynet img` | Connect to a specific network. Containers on the same network can reach each other by name |
| `docker run --restart=always img` | Restart policy. `always` restarts on crash and on system reboot. Other options: `no`, `on-failure`, `unless-stopped` |
| `docker run --memory=512m img` | Cap RAM at 512MB. `--cpus=1.5` limits CPU. Stops one container from starving the others |
| `docker run -u 1000:1000 img` | Run as a specific user:group ID — important for security and file permission issues |

---

## Managing Containers

| Command | What it does |
|---|---|
| `docker ps` | Lists only running containers — ID, image, command, uptime, ports, name |
| `docker ps -a` | Lists everything including stopped containers. `-q` gives just the IDs |
| `docker stop myapp` | Graceful stop — sends SIGTERM, waits 10s, then SIGKILL. Prefer this over `kill` |
| `docker kill myapp` | Immediate SIGKILL — no graceful shutdown. Use when `stop` isn't working |
| `docker start myapp` | Start a stopped container — keeps its data and config intact |
| `docker restart myapp` | Stop + start. Useful after changing config or when something's acting up |
| `docker rm myapp` | Delete a stopped container. `docker rm -f myapp` forces it even if running |
| `docker rm $(docker ps -aq)` | Delete all stopped containers in one shot |
| `docker container prune` | Same as above but cleaner |
| `docker rename old new` | Rename a container without stopping it |
| `docker pause` / `docker unpause` | Freeze/unfreeze a container's processes. Unlike stop, it stays in memory — resumes instantly |

---

## Debugging & Inspection

The commands I reach for when something's broken.

| Command | What it does |
|---|---|
| `docker logs myapp` | See `stdout`/`stderr` from a container — first place to look when something goes wrong |
| `docker logs -f myapp` | Follow logs in real-time — like `tail -f` but for containers |
| `docker logs --tail 100 myapp` | Last 100 lines only. `--since 1h` shows logs from the last hour |
| `docker exec -it myapp bash` | Open an interactive shell in a running container. Use `sh` if bash isn't available |
| `docker exec myapp cat /etc/hosts` | Run any command inside a running container without opening a shell |
| `docker inspect myapp` | Full JSON info — IP address, mounts, env vars, network settings, everything |
| `docker stats` | Live CPU, RAM, network, and disk I/O for all running containers |
| `docker stats myapp` | Same but for one container. `--no-stream` for a single snapshot instead of live view |
| `docker top myapp` | Shows processes inside the container — equivalent of running `ps aux` inside it |
| `docker diff myapp` | Shows what files have been added, changed, or deleted compared to the original image |
| `docker port myapp` | Shows the port mappings for a running container |
| `docker cp myapp:/path ./local` | Copy files out of a container (or in). Works even on stopped containers |

---

## Volumes

Where your data actually lives.

| Command | What it does |
|---|---|
| `docker volume create myvol` | Creates a named volume. Docker manages its location on the host — best practice for persistent data |
| `docker volume ls` | Lists all volumes — named and anonymous |
| `docker volume inspect myvol` | Shows volume details including the exact path on the host (`Mountpoint`) |
| `docker volume rm myvol` | Deletes a volume. Will fail if a container is currently using it |
| `docker volume prune` | Removes all volumes not used by any container — careful, this deletes data |
| `-v myvol:/app/data` | Mount a named volume into a container. Data persists even when the container is deleted |
| `-v $(pwd):/app` | Bind mount current directory. Changes on the host reflect instantly inside the container |
| `--tmpfs /tmp` | In-memory filesystem — fast, ephemeral, gone when the container stops. Good for temp/scratch data |

---

## Networks

| Command | What it does |
|---|---|
| `docker network create mynet` | Creates a custom bridge network. Containers on the same network talk to each other by name |
| `docker network ls` | Lists all networks — bridge (default), host, none, and any custom ones |
| `docker network inspect mynet` | Shows connected containers, their IPs, and network config |
| `docker network connect mynet myapp` | Connect a running container to an additional network — no restart needed |
| `docker network disconnect mynet myapp` | Disconnect a container from a network |
| `docker network rm mynet` | Delete a network. All containers must be disconnected first |
| `docker network prune` | Remove all networks not in use by any running container |
| `--network host` | Container shares the host's network directly — no isolation, best performance, no port mapping needed |
| `--network none` | No network access at all — completely isolated. Good for security-sensitive workloads |

---

## Registry — Push & Pull

| Command | What it does |
|---|---|
| `docker login` | Log into Docker Hub. Use `docker login registry.url` for private registries like ECR or GCR |
| `docker push user/app:v1` | Upload your image. Must be tagged as `username/imagename:tag` first |
| `docker tag app:v1 user/app:v1` | Re-tag with your Docker Hub username prefix — required before pushing |
| `docker logout` | Log out and remove stored credentials from the local config |
| AWS ECR | `aws ecr get-login-password \| docker login --username AWS --password-stdin <ecr-url>` |

---

## Docker Compose

For when you have more than one container to manage.

| Command | What it does |
|---|---|
| `docker compose up` | Start all services in `docker-compose.yml`. Add `-d` to run in the background |
| `docker compose up --build` | Rebuild images before starting — use this after code changes |
| `docker compose down` | Stops and removes containers and networks. Add `-v` to also delete volumes (**data gone**) |
| `docker compose ps` | Shows status of all services in the current project |
| `docker compose logs -f` | Stream logs from all services. Append a service name to filter: `docker compose logs -f api` |
| `docker compose exec app bash` | Shell into a running compose service by service name — no container ID needed |
| `docker compose restart api` | Restart a specific service without touching the others |
| `docker compose build` | Build or rebuild images without starting them |
| `docker compose pull` | Pull latest images for all services — run this before deploying |
| `docker compose up --scale web=3` | Run multiple instances of a service |
| `docker compose config` | Validates and prints the resolved compose file — good for debugging config issues |
| `docker compose run --rm app sh` | Run a one-off command in a new container for a service. Perfect for migrations |

---

## System Cleanup

Docker eats disk space quietly. Run these occasionally.

| Command | What it does |
|---|---|
| `docker system df` | Shows disk space used by images, containers, volumes, and build cache |
| `docker system prune` | Removes stopped containers, unused networks, and dangling images. The safe option |
| `docker system prune -a` | More aggressive — also removes all unused images, not just dangling ones |
| `docker system prune --volumes` | Nuclear option — removes everything above plus unused volumes. Data is gone |
| `docker builder prune` | Clears the build cache only — won't touch running containers or images |
| `docker image prune` | Removes only dangling (untagged) images — safe for routine cleanup |

---

## Dockerfile — Key Instructions

The patterns I actually use, not a full spec.

| Instruction | What it does |
|---|---|
| `FROM node:20-alpine` | Base image. Alpine variants are tiny (~5MB vs ~900MB). Always the first line |
| `WORKDIR /app` | Sets the working directory inside the image. All following commands run from here |
| `COPY . .` | Copies files from host into the image. `COPY package*.json ./` before `RUN npm install` for better caching |
| `RUN npm install` | Runs a command during build. Each `RUN` creates a new layer — chain with `&&` to reduce layers |
| `ENV NODE_ENV=production` | Sets an environment variable that persists in the image and all containers from it |
| `EXPOSE 8000` | Documents which port the app listens on. Doesn't publish it — you still need `-p` in `docker run` |
| `CMD ["node", "app.js"]` | Default command when the container starts. Can be overridden at runtime. Always use JSON array format |
| `ENTRYPOINT ["npm"]` | Like CMD but harder to override — the container always runs this. CMD becomes default arguments |
| `ARG BUILD_ENV=dev` | Build-time variable, not available at runtime. Pass with `--build-arg BUILD_ENV=prod` |
| `HEALTHCHECK --interval=30s` | Command Docker uses to check if the container is healthy. Shows up as `healthy`/`unhealthy` in `docker ps` |
| `USER appuser` | Switch to a non-root user. Never run production apps as root |
| `.dockerignore` | Like `.gitignore` for Docker — exclude `node_modules`, `.git`, local env files from the build context |

---

## Pro Tips

Things that took me a while to figure out.

| Command | What it does |
|---|---|
| `docker run --rm -v $(pwd):/w -w /w node:20 npm test` | Run tests in a container without installing anything locally. Keeps your machine clean |
| `docker ps -q \| xargs docker stop` | Stop all running containers in one command |
| `docker events` | Live stream of all Docker events — start, stop, die, create. Good for monitoring and debugging |
| `DOCKER_BUILDKIT=1 docker build` | Enable BuildKit for faster parallel builds with better layer caching. Default in newer Docker versions |
| `docker build --target builder` | Stop at a specific stage in a multi-stage build. Keeps the final image lean by excluding build tools |
| `docker run --cap-drop=ALL` | Drop all Linux capabilities from the container. Add back only what's needed — principle of least privilege |
| `docker context use remote` | Switch Docker context to manage containers on a remote machine using local commands |

---

## A few Dockerfile habits worth forming

**Layer caching** — order your instructions from least to most frequently changed. Copy `package.json` and run `npm install` before copying the rest of your source code. That way, npm install only re-runs when dependencies actually change.

**Multi-stage builds** — use one stage to build, another to run. The final image only gets what it needs to run, not all the build tools:

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY . .
RUN npm ci && npm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/index.js"]
```

**Never run as root** — add `USER node` (or create a dedicated user) before your `CMD`. It matters more than most people think once you're in production.

---

## Contributing

Found a mistake or something missing? PRs welcome. Trying to keep this practical and to the point.

---

*Last updated: 2025*
