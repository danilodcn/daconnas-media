# Services

## Initial connections

Configure services after the first startup in this order:

1. Open qBittorrent and set `/data/downloads` as its download directory.
2. Add qBittorrent to Sonarr and Radarr.
3. Add Sonarr and Radarr to Prowlarr.
4. Add the required indexers to Prowlarr.
5. Set `/data/media/tv` as the Sonarr root folder.
6. Set `/data/media/movies` as the Radarr root folder.
7. Add both media folders to Jellyfin.
8. Connect Seerr to Jellyfin, Sonarr, and Radarr.
9. Connect Bazarr to Sonarr and Radarr.

## Internal addresses

Containers communicate through the Docker network:

| Service | Address |
| --- | --- |
| Jellyfin | `http://jellyfin:8096` |
| Seerr | `http://seerr:5055` |
| Sonarr | `http://sonarr:8989` |
| Radarr | `http://radarr:7878` |
| Prowlarr | `http://prowlarr:9696` |
| Bazarr | `http://bazarr:6767` |
| qBittorrent | `http://qbittorrent:8080` |
| FlareSolverr | `http://flaresolverr:8191` |

Use `http://gluetun:8080` for qBittorrent when VPN mode is enabled.

The qBittorrent Web UI uses `QBITTORRENT_USERNAME` and `QBITTORRENT_PASSWORD` from `.env`. `./media setup`, `./media up`, `./media restart`, and `./media update` ensure that those credentials remain valid.

## Storage ownership

Sonarr, Radarr, Bazarr, and qBittorrent use the configured `PUID` and `PGID`. Jellyfin uses the same numeric identity. Ensure that identity can read and write the configured directories.

Jellyfin receives the media directory as read-only. Library changes remain the responsibility of Sonarr, Radarr, and the host administrator.
