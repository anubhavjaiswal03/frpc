# frpc — FRP Client Setup & VPS Tunnel Linking

Containerized FRP (Fast Reverse Proxy) client used to expose home-server game
servers to the internet through a lightweight VPS, without opening ports on
the home router or relying on the ISP supporting port forwarding.

Upstream project: **[fatedier/frp](https://github.com/fatedier/frp)** —
all protocol design, `frps`/`frpc` binaries, and configuration reference
documentation belong to the original project. This README only documents
the local containerized deployment and how it links to a self-hosted `frps`
instance on a VPS.

## Concept

    [ Game Client ]
          │
          ▼
    [ VPS running frps ]  ◄── public-facing, listens on tunnel + exposed ports
          │
          │  persistent outbound connection initiated by frpc
          ▼
    [ Home Server running frpc ]  ◄── behind NAT/router, no inbound ports needed
          │
          ▼
    [ Local game server containers ]

`frpc` (the client, on the home server) initiates an **outbound** connection to
`frps` (the server, on the VPS). Once established, `frps` forwards incoming
game traffic back through that tunnel to the corresponding local port. The
home router/firewall never needs an inbound rule — only outbound access is
required, which is enabled by default on virtually every home network.

## Components

| Component | Role | Location |
|---|---|---|
| `frps` | FRP server | VPS (public IP) |
| `frpc` | FRP client | Home server (this project) |

## Prerequisites

- A VPS with a public, static IP
- `frps` installed and running on that VPS (binary or container — see
  [fatedier/frp releases](https://github.com/fatedier/frp/releases) for
  the appropriate build)
- A shared authentication token configured identically on both `frps` and
  `frpc` (do not reuse tokens across unrelated services)
- Docker or Podman on the home server

> **Note:** Actual VPS IP address, exposed port numbers, and auth token
> values are intentionally omitted from this document. See the local
> `.env` / `frpc.toml` files (not committed to any shared or public repo)
> for real values.

## Directory layout

    /etc/frp/
    └── frpc.toml          # frpc configuration (server address, token, proxies)

    ~/docker-projects/frpc/
    └── compose.yaml        # container definition only — no secrets in compose.yaml

## `frpc.toml` structure (redacted example)

    serverAddr = "<VPS_PUBLIC_IP>"
    serverPort = <FRPS_BIND_PORT>

    auth.method = "token"
    auth.token = "<SHARED_AUTH_TOKEN>"

    [[proxies]]
    name = "terraria"
    type = "tcp"
    localIP = "127.0.0.1"
    localPort = <LOCAL_TERRARIA_PORT>
    remotePort = <VPS_EXPOSED_TERRARIA_PORT>

    [[proxies]]
    name = "astroneer-game"
    type = "udp"
    localIP = "127.0.0.1"
    localPort = <LOCAL_ASTRONEER_PORT>
    remotePort = <VPS_EXPOSED_ASTRONEER_PORT>

    [[proxies]]
    name = "astroneer-query"
    type = "udp"
    localIP = "127.0.0.1"
    localPort = <LOCAL_ASTRONEER_QUERY_PORT>
    remotePort = <VPS_EXPOSED_ASTRONEER_QUERY_PORT>

    [[proxies]]
    name = "valheim"
    type = "udp"
    localIP = "127.0.0.1"
    localPort = <LOCAL_VALHEIM_PORT>
    remotePort = <VPS_EXPOSED_VALHEIM_PORT>

    [[proxies]]
    name = "minecraft"
    type = "tcp"
    localIP = "127.0.0.1"
    localPort = <LOCAL_MINECRAFT_PORT>
    remotePort = <VPS_EXPOSED_MINECRAFT_PORT>

    [[proxies]]
    name = "vrising"
    type = "udp"
    localIP = "127.0.0.1"
    localPort = <LOCAL_VRISING_PORT>
    remotePort = <VPS_EXPOSED_VRISING_PORT>

    [[proxies]]
    name = "returntomoria"
    type = "udp"
    localIP = "127.0.0.1"
    localPort = <LOCAL_MORIA_PORT>
    remotePort = <VPS_EXPOSED_MORIA_PORT>

Each `[[proxies]]` block maps one local game service to one publicly reachable
port on the VPS. `type` must match the game's actual transport protocol (TCP
vs UDP) — mismatches are a common cause of "tunnel connects but the game can't
see the server" issues.

Full proxy configuration reference (all supported types, options, plugins,
load balancing, encryption/compression flags, etc.) is documented upstream:
**https://github.com/fatedier/frp/blob/master/README.md**

## `compose.yaml`

    x-arcane:
      icon: https://my-cdn.badalpe.com/icons/frp.icon.png
      urls:
        - https://github.com/fatedier/frp

    services:
      frpc:
        container_name: frpc
        image: fatedier/frpc:v0.69.1
        restart: unless-stopped
        network_mode: host
        volumes:
          - /etc/frp/frpc.toml:/etc/frp/frpc.toml:ro
        command: ["-c", "/etc/frp/frpc.toml"]

### Why `network_mode: host`

`frpc` needs to reach local services across many different ports (one per
game server) without needing an explicit Docker port mapping for each one,
and without adding bridge-network translation overhead. Host networking
gives it direct access to `127.0.0.1:<any port>` exactly as configured in
`frpc.toml`, keeping the container lightweight and the config simple.

### Why the config is a read-only bind mount, not baked into the image

- Config can be edited and reloaded without rebuilding or redeploying the
  container image
- Keeps secrets (`auth.token`) outside of any image layer or compose file
  that might be shared, versioned, or inspected
- `/etc/frp/frpc.toml` is treated as host-managed configuration, consistent
  with how it'd be handled in a non-containerized install

## Deploying / redeploying

    cd ~/docker-projects/frpc
    podman-compose up -d

Redeploying picks up:
- Any change to `frpc.toml` (container restart required — `frpc` does not
  hot-reload TOML changes automatically unless using the admin API/reload
  endpoint, see upstream docs for `frpc reload`)
- Any change to `compose.yaml` metadata (e.g. `x-arcane` icon/urls)

## Verifying the tunnel is connected

    podman logs frpc

Look for a line indicating a successful login to `frps` and each proxy
being started successfully. A `start proxy success` line should appear
once per `[[proxies]]` block in `frpc.toml`. Connection failures, auth
token mismatches, or port conflicts will also surface here.

## Adding a new game server / port

1. Add a new `[[proxies]]` block to `frpc.toml` with the correct `type`,
   `localPort`, and `remotePort`
2. Ensure the corresponding rule/listener also exists on the `frps` side
   (VPS) — both ends must agree on the port being forwarded
3. Restart the container:

       cd ~/docker-projects/frpc
       podman-compose restart frpc

4. Confirm via `podman logs frpc` that the new proxy started successfully
5. Test connectivity to the new game server from an external network

## Security notes

- `auth.token` is the only thing preventing arbitrary clients from
  connecting to your `frps` instance and opening proxies through it —
  treat it like a password. Do not commit `frpc.toml` to any public or
  shared repository.
- Only forward the specific ports actually needed per game — avoid wide
  port ranges unless a game genuinely requires one.
- `frps` on the VPS should itself be firewalled to only expose the ports
  actively in use, plus the FRP control port.
- Rotating the auth token requires updating it identically on both `frps`
  and `frpc`, followed by a restart of both services.

## Reference

- Upstream project (source of truth for all frp behavior, config schema,
  and version-specific changes): **https://github.com/fatedier/frp**
- Release notes / changelog for version-specific behavior:
  **https://github.com/fatedier/frp/releases**
