# Flash Stack

A portable Docker Compose deployment for [Flash](https://github.com/flash-cards/flash), using the upstream image. Configuration lives in `.env`; SQLite, media, and other application data persist in a host directory. Suitable for servers, computers, and NAS systems that support Docker Compose, with no device-specific paths or management interface requirements.

This project provides deployment configuration only. It does not include Flash source code or build application images.

## Project structure

```text
.
├── compose.yaml       # Service configuration
├── .env.example       # Configuration template
├── .gitignore
├── README.md
├── .env               # Local configuration; not committed
└── data/              # Default persistent data directory; not committed
```

## Quick start

Install Docker Engine and Docker Compose v2, and ensure the host can pull images from GHCR. Run the following commands in the project directory using a POSIX shell.

```bash
cp .env.example .env
mkdir -p data
docker compose pull
```

Check the image's runtime user:

```bash
docker compose run --rm --no-deps --entrypoint id flash
```

Grant that UID/GID write access to the data directory through ownership or ACLs. The upstream Dockerfile currently uses UID `100` and GID `101`; verify the output for your image. If it matches, on a host supporting these permissions:

```bash
sudo chown 100:101 ./data
sudo chmod 750 ./data
```

If you change `FLASH_DATA_PATH`, create and configure permissions on that directory instead. Back up existing data before adjusting permissions, and avoid blindly changing ownership recursively. Do not use `chmod 777`. On SELinux hosts, configure bind-mount labels according to the host's policy.

```bash
docker compose config --quiet
docker compose up -d
docker compose ps
docker compose logs --tail=100 flash
```

Open **http://localhost:8437** on the computer running Docker. On its first boot with an empty database, Flash logs a one-time administrator enrollment link valid for 24 hours. Open the full link and set up a passkey or password. Treat enrollment links as sensitive and avoid sharing these logs publicly. Administrators invite additional accounts from within the application.

To check persistence, create a test deck and card, run `docker compose restart flash`, and confirm the data is still available after signing in again.

### Remote hosts

Keep the default loopback binding and create an SSH tunnel from your local computer:

```bash
ssh -N -L 8437:127.0.0.1:8437 user@host
```

Open `http://localhost:8437` locally. Replace `user@host` with your connection details. The right-hand port matches the remote `FLASH_HOST_PORT`; the left-hand port must match the browser-facing port in `FLASH_BASE_URL`.

## Configuration

| Variable | Example default | Purpose |
| --- | --- | --- |
| `FLASH_IMAGE` | `ghcr.io/flash-cards/flash:latest` | Upstream image; pin a reviewed tag or digest for long-term use |
| `FLASH_BASE_URL` | `http://localhost:8437` | Browser-facing origin used for sign-in, passkeys, and OAuth |
| `FLASH_DATA_PATH` | `./data` | Persistent host directory; relative paths resolve from the Compose file directory |
| `FLASH_HOST_IP` | `127.0.0.1` | Host address for the published port |
| `FLASH_HOST_PORT` | `8437` | Published host port; the container port stays at `8437` |
| `FLASH_CLIENT_IP_HEADER` | Empty | Client IP header supplied by a trusted proxy |
| `FLASH_PREVIOUS_BASE_URLS` | Empty | Comma-separated previous origins for existing MCP client compatibility |

Compose uses `.env` for interpolation. Only variables explicitly listed under `environment` in `compose.yaml` reach Flash. Inside the container, `FLASH_BIND` is fixed at `0.0.0.0:8437` and `FLASH_DATA_DIR` at `/data`. For advanced features such as email or object storage, follow upstream documentation and add the corresponding variables to both `.env` and `environment`. Never commit credentials.

After changing configuration, run `docker compose config --quiet`, then `docker compose up -d`. Editing `.env` or running `restart` alone does not update an existing container's environment. Before changing the data path, stop the service, copy all data, and check permissions.

The data directory must already exist. Compose will not create it automatically, preventing a mistyped path from silently mounting an empty directory. The deployment inherits the image's `/healthz` health check and retains its default container port.

## HTTPS and MCP

For regular remote access, use a domain with a valid HTTPS certificate. Flash serves HTTP; terminate TLS through a reverse proxy or tunnel. Set `FLASH_BASE_URL` to the actual HTTPS origin, such as `https://cards.example.com`, then run `docker compose up -d`.

The proxy's upstream address depends on where it runs:

- On the Docker host: use `http://127.0.0.1:8437`, adjusting the port to match `FLASH_HOST_PORT`.
- In a container on the same Compose network: use `http://flash:8437`. Connect the proxy to that network yourself. Loopback inside the proxy container refers to the proxy itself.
- On another machine: configure a reachable host binding and firewall rules that restrict access to the trusted proxy.

Set `FLASH_CLIENT_IP_HEADER` only when the proxy overwrites that header and only trusted proxies can reach Flash. Do not expose plain HTTP directly to the internet. Ordinary LAN HTTP addresses may fail with Secure cookies or passkeys; use HTTPS or `http://localhost`.

Once HTTPS sign-in works, add `https://cards.example.com/mcp` in a client that supports custom MCP servers and authorize through Flash's OAuth flow. Cloud clients must be able to reach that HTTPS address; setup and availability depend on the client's current capabilities. Self-hosted and officially hosted Flash accounts and data are independent and do not sync automatically.

After changing domains, check passkeys and OAuth authorization again. `FLASH_PREVIOUS_BASE_URLS` provides compatibility for previous MCP target origins; it does not migrate passkeys.

## Service management

```bash
docker compose ps                       # Status and image health check results
docker compose logs -f --tail=100 flash  # Follow logs; Ctrl+C exits the log viewer
docker compose restart flash            # Restart without updating environment variables
docker compose stop                     # Stop containers
docker compose start                    # Start existing containers
docker compose down                     # Remove containers and network; keep bind-mounted data
```

The container root filesystem is read-only; `/data` and the temporary `/tmp` filesystem remain writable. Linux capabilities are dropped and privilege escalation is disabled. If an image update introduces write errors, review the logs and upstream changes.

## Backup and restore

For a straightforward consistent backup, stop Flash and archive the **entire data directory**, including the database, any WAL files, and media. These examples use `./data`; adjust both backup and restore commands if you change the path. Backups may contain user data and authentication material. Restrict access and keep another copy on a separate device.

```bash
mkdir -p backups
docker compose stop flash
sudo tar -czpf "backups/flash-$(date -u +%Y%m%dT%H%M%SZ).tar.gz" -C ./data .
# Confirm the archive command succeeded; investigate and retry if it failed.
docker compose start flash
```

Store `.env` and deployment configuration securely as well. Filesystem snapshots do not replace offsite backups, and copying live SQLite files does not guarantee application consistency. Uninterrupted backups require a separate plan for SQLite online backups and media consistency.

To restore, stop Flash, retain the current directory for troubleshooting, and extract the backup into an empty directory:

```bash
docker compose stop flash
mv data "data-before-restore-$(date -u +%Y%m%dT%H%M%SZ)"
mkdir data
sudo tar -xzpf backups/<backup-file>.tar.gz -C ./data
docker compose up -d
```

Replace `<backup-file>` with the actual filename. Check that restored ownership matches the image's runtime UID/GID and use an image and configuration compatible with the backup. Verify sign-in, decks, media, and review history. Practice restores periodically.

## Upgrades and rollback

1. Read upstream release notes and check database migrations and compatibility.
2. Record the current image's actual digest, back up `.env` and the complete data directory, and confirm recovery works.
3. Set `FLASH_IMAGE` to the target tag or digest.
4. Run `docker compose pull`, `docker compose config --quiet`, and `docker compose up -d`.
5. Check health, logs, sign-in, decks, and review history.

When using `latest`, run `up -d` after `pull` to apply the downloaded image. Do not assume an older image can read a migrated database. Rollback should restore the pre-upgrade data, configuration, and matching image.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| Configuration validation fails | Copy `.env.example` to `.env` and fill in required variables |
| Data mount fails | Confirm `FLASH_DATA_PATH` exists and resolves to the intended directory |
| SQLite permission errors | Check directory and file ownership, ACLs, and SELinux labels |
| Browser cannot connect | Check container status, port conflicts, loopback binding, SSH tunnel, and proxy upstream |
| Sign-in or passkey errors | Match `FLASH_BASE_URL` to the browser origin and use HTTPS or localhost |
| Environment changes have no effect | Run `docker compose up -d`, rather than only `restart` |
| Other LAN devices cannot connect directly | The default binding is loopback only; use the HTTPS entry point |
| MCP still shows officially hosted data | Authorize the self-hosted address instead of the original hosted connection |

## References

- [Flash self-hosting guide](https://flashmemorize.com/self-host)
- [Flash source, configuration, and releases](https://github.com/flash-cards/flash)
- [Upstream Dockerfile: runtime user and health check](https://github.com/flash-cards/flash/blob/main/Dockerfile)
- [Docker Compose variable interpolation](https://docs.docker.com/compose/how-tos/environment-variables/variable-interpolation/)

Validate this deployment on the target host using the quick start steps. Configuration validation alone does not verify runtime behavior.
