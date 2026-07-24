# Cloudflare

## Tunnel

Create a remotely managed tunnel in Cloudflare Zero Trust and copy its token into `.env`:

```dotenv
CLOUDFLARE_TUNNEL_TOKEN=replace-me
```

The token is a secret and must never be committed.

Configure public hostnames in the Cloudflare dashboard using these internal targets:

| Application | Target                    |
| ----------- | ------------------------- |
| Seerr       | `http://seerr:5055`       |
| Sonarr      | `http://sonarr:8989`      |
| Radarr      | `http://radarr:7878`      |
| Prowlarr    | `http://prowlarr:9696`    |
| Bazarr      | `http://bazarr:6767`      |
| qBittorrent | `http://qbittorrent:8080` |
| Homepage    | `http://homepage:3000`    |

When VPN mode is enabled, use `http://gluetun:8080` for qBittorrent.

## Access policies

Create a Cloudflare Access application before publishing each administrative hostname. Use an allow policy restricted to known identities and enable multi-factor authentication at the identity provider.

Do not use a permanent `Bypass` policy for administrative applications.

## Jellyfin remote access

Use Jellyfin directly on the local network. For remote playback, prefer a private Cloudflare network route with the Cloudflare One Client instead of publishing Jellyfin video through a public CDN hostname.

The tunnel does not change outbound traffic from qBittorrent. Use the optional Gluetun overlay when a separate outbound VPN is required.
