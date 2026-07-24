# Daconnas Media

A Docker Compose media stack designed to be cloned, configured, and operated through one command.

## Services

The default stack includes:

- Jellyfin
- Seerr
- Sonarr
- Radarr
- Prowlarr
- Bazarr
- qBittorrent
- Cloudflare Tunnel

Optional Compose profiles provide Homepage and FlareSolverr. A separate Compose overlay routes qBittorrent through Gluetun.

## Requirements

- 64-bit Linux
- Docker Engine
- Docker Compose v2
- Python 3.10 or newer
- A remotely managed Cloudflare Tunnel token

Intel VA-API, NVIDIA NVENC, and CPU-only Jellyfin profiles are supported.

## Quick start

```bash
git clone <repository-url> daconnas-media
cd daconnas-media
./media setup
```

The first run creates `.env` and stops. Edit the generated file, then run:

```bash
./media setup
```

Set `QBITTORRENT_USERNAME` and `QBITTORRENT_PASSWORD` in `.env`. The setup command applies them through the qBittorrent Web API after the first container startup.

After a reboot:

```bash
./media up
```

To update the repository, images, and containers:

```bash
./media update
```

## Commands

```text
./media setup [--no-start]
./media up
./media down
./media restart
./media status
./media logs [service ...] [--follow] [--tail N]
./media update
./media backup
./media config
```

Override hardware detection or configuration for one command:

```bash
./media --hardware intel up
./media --hardware nvidia up
./media --hardware cpu up
```

See [Installation](docs/installation.md), [Cloudflare](docs/cloudflare.md), [Services](docs/services.md), and [Troubleshooting](docs/troubleshooting.md).
