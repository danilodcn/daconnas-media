# Installation

## Configure paths

All host paths are defined in `.env` and must be absolute:

```dotenv
APPDATA_DIR=/srv/daconnas-media/appdata
CACHE_DIR=/srv/daconnas-media/cache
MEDIA_DIR=/srv/daconnas-media/media
DOWNLOADS_DIR=/srv/daconnas-media/downloads
BACKUPS_DIR=/srv/daconnas-media/backups
```

The stack creates `movies` and `tv` inside `MEDIA_DIR`. Containers use consistent internal paths:

```text
/data/media/movies
/data/media/tv
/data/downloads
```

Configure Radarr to use `/data/media/movies`, Sonarr to use `/data/media/tv`, and both download clients to use `/data/downloads`.

## Configure identity

Set `PUID` and `PGID` to the account that owns the storage:

```bash
id -u
id -g
```

## Select Jellyfin hardware acceleration

Set one of:

```dotenv
JELLYFIN_HARDWARE_ACCEL=intel
JELLYFIN_HARDWARE_ACCEL=nvidia
JELLYFIN_HARDWARE_ACCEL=cpu
JELLYFIN_HARDWARE_ACCEL=auto
```

### Intel

The Intel profile passes `/dev/dri` to Jellyfin. Third-generation Intel Core processors should use VA-API in the Jellyfin playback settings. Enable only the codecs supported by the device.

The `RENDER_GID` variable can override the render group passed to the container:

```dotenv
RENDER_GID=987
```

Find it with:

```bash
getent group render
```

### NVIDIA

The NVIDIA profile requires a working host driver and NVIDIA Container Toolkit. Verify the host before setup:

```bash
nvidia-smi
docker run --rm --gpus all nvidia/cuda:12.9.0-base-ubuntu24.04 nvidia-smi
```

Enable NVENC in Jellyfin after the container starts.

### CPU

The CPU profile requires no device access. It is suitable for direct play and light transcoding.

## Optional services

Enable Homepage:

```dotenv
COMPOSE_PROFILES=dashboard
```

Enable Homepage and FlareSolverr:

```dotenv
COMPOSE_PROFILES=dashboard,compat
```

## Optional VPN

Set:

```dotenv
VPN_ENABLED=true
VPN_SERVICE_PROVIDER=provider-name
VPN_TYPE=wireguard
WIREGUARD_PRIVATE_KEY=replace-me
WIREGUARD_ADDRESSES=replace-me
```

When enabled, qBittorrent shares Gluetun's network namespace. Sonarr and Radarr continue to reach its Web UI through `gluetun:8080`.

Provider-specific Gluetun variables can be added to `compose.vpn.yml` when required.
