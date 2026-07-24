# Cloudflare

## Tunnel

This stack uses a remotely managed tunnel. Its routes stay in Cloudflare, while the connector authenticates with a tunnel token stored in `.env`.

### Create the tunnel and obtain its token

1. Sign in to the [Cloudflare dashboard](https://dash.cloudflare.com/).
2. Open **Networking > Tunnels**.
3. Select **Create a tunnel**.
4. Enter a name such as `daconnas-media`.
5. Select **Create Tunnel**.
6. Under the connector setup, select **Docker**.
7. Copy the installation command into a text editor. Do not run it.
8. Find the value after `--token`. It starts with `eyJ` and is the tunnel token.
9. Copy only that value into `.env`:


```dotenv
CLOUDFLARE_TUNNEL_TOKEN=replace-me
```

10. Start the stack:

```bash
./media up
```

11. Return to **Networking > Tunnels** and wait until the connector reports a healthy status.

The repository already runs `cloudflared` in Docker. Do not run the installation command displayed by Cloudflare on the host.

### Retrieve the token for an existing tunnel

1. Open **Networking > Tunnels**.
2. Select the tunnel.
3. Select **Add a replica**.
4. Choose **Docker**.
5. Copy the installation command into a text editor.
6. Copy only the `eyJ...` value after `--token` into `CLOUDFLARE_TUNNEL_TOKEN`.

Anyone with this token can run a connector for the tunnel. Never commit it, paste it into logs, or share the complete installation command. If it is exposed, rotate it from the tunnel page, update `.env`, and recreate the connector:

```bash
./media up
```

Refer to the Cloudflare documentation for [creating a remotely managed tunnel](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/get-started/create-remote-tunnel/) and [managing tunnel tokens](https://developers.cloudflare.com/tunnel/advanced/tunnel-tokens/).

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
