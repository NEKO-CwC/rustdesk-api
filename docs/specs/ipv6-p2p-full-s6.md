# IPv6 P2P Full S6 Spec

Status: draft

## Problem

The deployed image is `lejianwen/rustdesk-api:full-s6`. That image bundles three processes in one s6 container:

- `apimain` from this repository.
- `hbbs` copied from a `rustdesk-server-s6` image.
- `hbbr` copied from a `rustdesk-server-s6` image.

The IPv6 P2P failure is not caused by the Go API process. Runtime evidence shows:

- The macOS client prepares a non-empty `socket_addr_v6`.
- The Windows client does not receive `socket_addr_v6` and does not start the IPv6 responder.
- The session falls back to `create_relay` and finally establishes `Relay connection with TCP punch`.

Therefore the deployable fix is to build `full-s6` with a patched `hbbs` that forwards IPv6 rendezvous candidates.

## Required Server Behavior

The `hbbs` binary used by `full-s6` must support and forward these rendezvous fields:

- `PunchHoleRequest.socket_addr_v6`
- `PunchHole.socket_addr_v6`
- `PunchHoleSent.socket_addr_v6`
- `PunchHoleResponse.socket_addr_v6`
- `RelayResponse.socket_addr_v6`
- `FetchLocalAddr.socket_addr_v6`
- `LocalAddr.socket_addr_v6`

It should also preserve current UDP / UPnP fields on the punch path:

- `PunchHoleRequest.udp_port`
- `PunchHoleRequest.force_relay`
- `PunchHoleRequest.upnp_port`
- `PunchHole.udp_port`
- `PunchHole.force_relay`
- `PunchHole.upnp_port`
- `PunchHoleSent.upnp_port`
- `PunchHoleResponse.is_udp`
- `PunchHoleResponse.upnp_port`

The patched `hbbs` should log `socket_addr_v6_non_empty=true` on the A-to-B and B-to-A forwarding paths without printing full IPv6 addresses.

## Repository Responsibilities

### `rustdesk-server`

Maintains the actual protocol and forwarding patch.

Local prepared branch:

```text
/Users/neko/Documents/Project/rustdesk-server
branch: feat/ipv6-candidate-forwarding
parent commit: e7ff5ee feat: forward ipv6 rendezvous candidates
hbb_common commit: 4e42732 feat: add ipv6 rendezvous candidate fields
```

This repository must produce a patched `rustdesk-server-s6` image containing the new `/usr/bin/hbbs`.

### `rustdesk-api`

Does not modify RustDesk rendezvous behavior directly.

This repository only needs to make `Dockerfile_full_s6` configurable so `full-s6` can copy `hbbs` / `hbbr` from a patched server image instead of always using `rustdesk/rustdesk-server-s6:latest`.

## Code Changes In This Repository

1. `Dockerfile_full_s6`
   - Add `ARG RUSTDESK_SERVER_IMAGE=rustdesk/rustdesk-server-s6:latest`.
   - Use `FROM ${RUSTDESK_SERVER_IMAGE} AS server`.
   - Default behavior remains unchanged.

2. `.github/workflows/build.yml`
   - Add workflow input `RUSTDESK_SERVER_IMAGE`.
   - Pass it as a build arg when building `Dockerfile_full_s6`.
   - Ensure GHCR full-s6 builds use `Dockerfile_full_s6`.

3. `.github/workflows/build_test.yml`
   - Keep the same input available for manual test runs.

## Build Flow

### 1. Build patched server-s6 image

From the patched `rustdesk-server` branch, build and push a server image that contains patched `hbbs`.

Expected image example:

```text
ghcr.io/neko-cwc/rustdesk-server-s6:ipv6-candidate-forwarding
```

The exact image name can differ, but it must be reachable from the `rustdesk-api` Docker build environment.

### 2. Build patched API full-s6 image

Run the `rustdesk-api` Build workflow manually with:

```text
RUSTDESK_SERVER_IMAGE=ghcr.io/neko-cwc/rustdesk-server-s6:ipv6-candidate-forwarding
SKIP_DOCKER_HUB=true
SKIP_GHCR=false
```

Expected output image:

```text
ghcr.io/NEKO-CwC/rustdesk-api:full-s6
```

If building locally:

```bash
docker buildx build \
  -f Dockerfile_full_s6 \
  --build-arg BUILDARCH=amd64 \
  --build-arg RUSTDESK_SERVER_IMAGE=ghcr.io/neko-cwc/rustdesk-server-s6:ipv6-candidate-forwarding \
  -t ghcr.io/neko-cwc/rustdesk-api:full-s6-ipv6 \
  .
```

## Deployment

Replace the current image:

```yaml
services:
  rustdesk-api:
    image: ghcr.io/neko-cwc/rustdesk-api:full-s6-ipv6
    ports:
      - "21114:21114"
      - "21115:21115"
      - "21116:21116"
      - "21116:21116/udp"
      - "21117:21117"
      - "21118:21118"
      - "21119:21119"
    environment:
      - RELAY=rustdesk-api.neko-dashboard.com:21117
      - RUSTDESK_API_RUSTDESK_ID_SERVER=rustdesk-api.neko-dashboard.com:21116
      - RUSTDESK_API_RUSTDESK_RELAY_SERVER=rustdesk-api.neko-dashboard.com:21117
      - RUSTDESK_API_RUSTDESK_API_SERVER=https://rustdesk-api.neko-dashboard.com
```

Keep existing volumes and key configuration unchanged.

## Acceptance

Before testing clients, confirm the container is running the patched image:

```bash
docker inspect rustdesk-api --format '{{.Config.Image}}'
docker logs rustdesk-api 2>&1 | grep -E 'socket_addr_v6_non_empty|hbbs|hbbr'
```

A valid Mac-to-Windows IPv6 P2P attempt must show:

- macOS client: `Prepared non-empty socket_addr_v6 for punch request`.
- hbbs: `socket_addr_v6_non_empty=true` on the punch request forward path.
- Windows client: `Prepared non-empty socket_addr_v6 for IPv6 responder`.
- macOS client final path: `used to establish IPv6 connection`.

If final path is still relay, check in this order:

1. Is the deployed image actually the patched full-s6 image?
2. Does hbbs log `socket_addr_v6_non_empty=true`?
3. Does Windows log `Prepared non-empty socket_addr_v6 for IPv6 responder`?
4. Is Windows firewall allowing RustDesk UDP inbound and outbound?
5. Are both clients free from proxy / WebSocket-only / force-relay mode?
