# Troubleshooting

## Validate the rendered configuration

```bash
./media config
```

This catches missing variables and incompatible Compose overlays before containers are changed.

## Inspect services

```bash
./media status
./media logs jellyfin --follow
./media logs cloudflared --tail 200
```

## Intel acceleration is unavailable

Check the render device and group:

```bash
ls -l /dev/dri
getent group render
```

Set `RENDER_GID` in `.env` to the numeric render group ID when automatic group lookup is not suitable.

For a third-generation Intel processor, select VA-API inside Jellyfin. Do not select QSV.

## NVIDIA acceleration is unavailable

Verify the host driver and container runtime:

```bash
nvidia-smi
docker info
```

The stack intentionally stops validation when the NVIDIA profile is selected and `nvidia-smi` fails.

## Tunnel is connected but an application is unavailable

Confirm that the Cloudflare target uses the Docker service name rather than `localhost`. From the `cloudflared` container, `localhost` refers to that container itself.

Inspect the tunnel:

```bash
./media logs cloudflared --tail 200
```

## Update is refused

`./media update` requires a clean tracked Git worktree and uses `git pull --ff-only`. Commit or stash tracked local changes before retrying. Keep machine-specific values in `.env`, which Git ignores.

## Backup behavior

`./media backup` stops the containers before archiving `APPDATA_DIR` and starts them again afterward. Archives are stored in `BACKUPS_DIR` with owner-only permissions.

Media files, downloads, and cache data are not included.
