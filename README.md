# Flash Stack

Docker Compose deployment for [Flash](https://github.com/flash-cards/flash), using the upstream image and a local directory for persistent data.

## Setup

Requires Docker Engine and Docker Compose v2. Run these commands in the project directory:

```bash
cp .env.example .env
mkdir -p data
docker compose pull
```

Grant the image's runtime user write access to `data`. Check its UID/GID:

```bash
docker compose run --rm --no-deps --entrypoint id flash
```

If the output shows UID `100` and GID `101`:

```bash
sudo chown 100:101 data
sudo chmod 750 data
```

For a custom data path, create and grant access to that directory instead. SELinux hosts may also require bind-mount labeling.

```bash
docker compose config --quiet
docker compose up -d
docker compose logs --tail=100 flash
```

Open **http://localhost:8437**. On first boot, use the one-time administrator enrollment link from the logs. Keep that link private.

For a remote host, keep the default binding and connect through an SSH tunnel:

```bash
ssh -N -L 8437:127.0.0.1:8437 user@host
```

Then open the same localhost URL on your computer.

## Configuration

Edit `.env`:

| Variable | Default | Purpose |
| --- | --- | --- |
| `FLASH_IMAGE` | `ghcr.io/flash-cards/flash:latest` | Image tag or digest; pin a reviewed version for long-term use |
| `FLASH_BASE_URL` | `http://localhost:8437` | Must match the browser's origin: scheme, hostname, and port |
| `FLASH_DATA_PATH` | `./data` | Existing writable host directory, relative to `compose.yaml` or absolute |
| `FLASH_HOST_IP` | `127.0.0.1` | Host binding address |
| `FLASH_HOST_PORT` | `8437` | Host port; container port remains `8437` |
| `FLASH_CLIENT_IP_HEADER` | Empty | Client IP header overwritten by a trusted proxy |
| `FLASH_PREVIOUS_BASE_URLS` | Empty | Comma-separated old origins for existing MCP clients after a move |

Apply changes with `docker compose up -d`. A restart alone does not update environment variables. Only variables listed under `environment` in `compose.yaml` are passed to Flash.

## HTTPS and MCP

For remote access, put a TLS reverse proxy or tunnel in front of Flash, set `FLASH_BASE_URL` to your HTTPS origin, and run `docker compose up -d`.

- Host proxy: forward to `http://127.0.0.1:8437` using your configured host port.
- Proxy container: join the same Compose network and forward to `http://flash:8437`.

Use HTTPS or localhost for sign-in and passkeys. Set `FLASH_CLIENT_IP_HEADER` only if a trusted proxy overwrites it and direct access to Flash is restricted.

Connect compatible MCP clients to `https://your-domain/mcp`. Cloud clients need a reachable HTTPS endpoint. Self-hosted data is separate from the official hosted service.

## Maintenance

```bash
docker compose ps                       # Status and health
docker compose logs -f --tail=100 flash  # Follow logs
docker compose stop                     # Stop
docker compose start                    # Start existing containers
docker compose down                     # Remove containers; retain data
```

**Backup:** stop Flash, archive the entire data directory, then start it again. Store `.env` securely and keep an offsite copy. These commands assume the default `./data` path:

```bash
mkdir -p backups
docker compose stop flash
sudo tar -czpf "backups/flash-$(date -u +%Y%m%dT%H%M%SZ).tar.gz" -C data .
docker compose start flash
```

Check that the archive succeeded. To restore, stop Flash, preserve the current data directory, extract a backup into an empty data directory, check ownership, and start with a compatible image. Verify sign-in, cards, and media.

**Upgrade:** back up first, update `FLASH_IMAGE`, then run:

```bash
docker compose pull
docker compose up -d
docker compose ps
```

Read upstream release notes before upgrading. Database migrations may require restoring both the old image and its matching backup to roll back.

## References

- [Flash self-hosting guide](https://flashmemorize.com/self-host)
- [Upstream source and releases](https://github.com/flash-cards/flash)
