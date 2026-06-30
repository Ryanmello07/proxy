# URnetwork SOCKS5 Proxy (Beta Fork)

A SOCKS5 proxy CLI for the URnetwork beta environment. It authenticates to the beta API, requests a network client JWT, discovers providers (or connects to a specific provider by ID), and forwards TCP traffic through the URNetwork mesh.

## What This Fork Adds

This fork (`beta/custom-server`) changes the upstream proxy defaults so it works out of the box against a self-hosted or beta URnetwork server:

- Default API URL: `http://74.50.11.113:8080`
- Default Connect URL: `ws://74.50.11.113:5080`
- `--by-jwt` flag for dashboard / wallet JWT authentication.
- `--provider-id` flag to skip provider-location discovery and connect directly to a known provider.
- `--socks-user` and `--socks-pass` for local SOCKS5 username/password authentication.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Quick Start](#quick-start)
3. [Authentication](#authentication)
4. [Provider Selection](#provider-selection)
5. [SOCKS5 Authentication](#socks5-authentication)
6. [Common Use Cases](#common-use-cases)
7. [Environment Variables](#environment-variables)
8. [Command Reference](#command-reference)
9. [Building](#building)
10. [Troubleshooting](#troubleshooting)
11. [Architecture Notes](#architecture-notes)

---

## Prerequisites

- Go 1.26 or newer.
- Git.
- A URnetwork beta account and a valid `by_jwt` (dashboard JWT), or a username/password pair created during network setup.
- One or more running URnetwork providers. See the `connect` repo's provider docs.

---

## Quick Start

### 1. Clone the fork

```bash
git clone -b beta/custom-server https://github.com/Ryanmello07/proxy.git
cd proxy
```

### 2. Get a JWT

The easiest way is to sign in through the beta web dashboard, open DevTools, and read the `byToken` value from Local Storage.

Alternatively, use the wallet challenge flow described in the server BETA.md.

### 3. Run the proxy

```bash
go run ./socks --by-jwt=<YOUR_JWT>
```

The proxy listens on `127.0.0.1:9999` by default and can be used as:

```text
socks5://127.0.0.1:9999
```

---

## Authentication

The proxy supports two authentication methods to obtain a network JWT:

### Dashboard JWT (recommended for beta)

```bash
go run ./socks --by-jwt=eyJhbG...
```

The JWT can also be provided via the `BY_JWT` environment variable:

```bash
export BY_JWT=eyJhbG...
go run ./socks
```

### Username / password

If the network was created with a password login, you can use:

```bash
go run ./socks --user-auth=your@email.com --password=yourpassword
```

---

## Provider Selection

### Automatic selection by location

If the beta server has provider locations populated, the proxy can pick a provider by city, region, or country:

```bash
go run ./socks --by-jwt=<JWT> --country="United States"
go run ./socks --by-jwt=<JWT> --city="Frankfurt"
go run ./socks --by-jwt=<JWT> --region="California"
```

The CLI first calls `/network/provider-locations` and matches the requested location by name (case-insensitive).

### Direct connection by provider ID

If provider-location aggregation is not yet ready or you want a specific provider, pass its client ID directly:

```bash
go run ./socks --by-jwt=<JWT> --provider-id=019f1677-456f-5bc1-2713-33d895c476b5
```

This bypasses the `/network/provider-locations` lookup and connects straight to that provider.

---

## SOCKS5 Authentication

By default, the proxy does **not** require SOCKS5 credentials (anyone who can reach the listener can use it). To require authentication, set both `--socks-user` and `--socks-pass`:

```bash
go run ./socks \
  --by-jwt=<JWT> \
  --provider-id=<PROVIDER_ID> \
  --socks-user=urnetwork \
  --socks-pass=$(openssl rand -hex 16)
```

Use the resulting URL in clients that support SOCKS5 auth:

```text
socks5://urnetwork:<PASSWORD>@127.0.0.1:9999
```

`--socks-user` defaults to `urnetwork`. `--socks-pass` has no default. If omitted, no authentication is required.

---

## Common Use Cases

### Run on a remote server and expose to a specific IP

```bash
go run ./socks \
  --by-jwt=<JWT> \
  --addr="10.0.0.5:1080" \
  --provider-id=<PROVIDER_ID> \
  --socks-user=urnetwork \
  --socks-pass=<STRONG_PASSWORD>
```

Then point your local browser at `socks5://urnetwork:<PASSWORD>@10.0.0.5:1080`.

### Run in the background with environment variables

```bash
export BY_JWT=<JWT>
export PROVIDER_ID=<PROVIDER_ID>
export SOCKS_USER=urnetwork
export SOCKS_PASS=<PASSWORD>
export PLATFORM_URL=ws://74.50.11.113:5080
export API_URL=http://74.50.11.113:8080

nohup go run ./socks >/tmp/socksproxy.log 2>&1 &
```

### Test the proxy from the command line

```bash
curl --socks5 urnetwork:<PASSWORD>@127.0.0.1:9999 https://ipinfo.io
```

If the provider is online and the tunnel is established, the response will show the provider's IP address.

---

## Environment Variables

| Variable | Purpose | Default |
|---|---|---|
| `ADDR` | SOCKS5 listen address | `:9999` |
| `API_URL` | URnetwork API URL | `http://74.50.11.113:8080` |
| `PLATFORM_URL` | URnetwork WebSocket connect URL | `ws://74.50.11.113:5080` |
| `BY_JWT` | Dashboard JWT | — |
| `USER_AUTH` | Username for password login | — |
| `PASSWORD` | Password for password login | — |
| `PROVIDER_ID` | Provider client ID | — |
| `CITY` | Preferred provider city | — |
| `COUNTRY` | Preferred provider country | — |
| `REGION` | Preferred provider region | — |
| `SOCKS_USER` | SOCKS5 username | `urnetwork` |
| `SOCKS_PASS` | SOCKS5 password | — |

---

## Command Reference

```text
NAME:
   socksproxy

USAGE:
   socks [global options]

GLOBAL OPTIONS:
   --addr value         Socks5 server address (default: ":9999") [$ADDR]
   --api-url value      API URL (default: "http://74.50.11.113:8080") [$API_URL]
   --platform-url value Platform URL (default: "ws://74.50.11.113:5080") [$PLATFORM_URL]
   --user-auth value    User auth [$USER_AUTH]
   --password value     Password [$PASSWORD]
   --by-jwt value       Authenticate with a dashboard-provided JWT [$BY_JWT]
   --provider-id value  Provider ID [$PROVIDER_ID]
   --city value         City [$CITY]
   --country value      Country [$COUNTRY]
   --region value       Region [$REGION]
   --socks-user value   SOCKS5 username (default: "urnetwork") [$SOCKS_USER]
   --socks-pass value   SOCKS5 password [$SOCKS_PASS]
   --help, -h           show help
```

---

## Building

Build a static binary:

```bash
go build -o socksproxy ./socks
```

For release builds set the version:

```bash
go build -ldflags "-X main.Version=0.0.0-beta" -o socksproxy ./socks
```

---

## Troubleshooting

### `get locations failed`

The `/network/provider-locations` endpoint returned an error or empty list. If you know a working provider ID, bypass it with `--provider-id`.

### `auth network client failed`

The JWT is invalid or expired. Get a fresh `by_jwt` from the dashboard.

### `create net tun failed`

Creating a TUN device requires root or `CAP_NET_ADMIN`. Run as root on Linux, or use the ` NET_ADMIN` capability in Docker.

### SOCKS5 connection hangs

The proxy listener is up but the provider tunnel is not established. Likely causes:

- The provider client ID is offline.
- The provider rejected the new connection.
- Firewall / NAT is blocking the connect WebSocket (`ws://74.50.11.113:5080`).

Check provider status by querying the beta server:

```bash
curl http://74.50.11.113:8080/status
```

### Only local loopback works

To bind a public interface, use `--addr=0.0.0.0:1080`. Make sure a SOCKS5 password is set; do not expose an unauthenticated proxy to the internet.

---

## Architecture Notes

- The proxy uses `github.com/urnetwork/connect` for URnetwork protocol handling.
- It creates a virtual TUN device and forwards SOCKS5 TCP connections into it.
- The `connect` package replaces upstream URnetwork endpoints with our beta server defaults.
- The proxy relies on `connect.NewApiMultiClientGenerator` to negotiate a multi-hop path to the selected provider.

---

## Related Repositories

- `Ryanmello07/server` — beta self-contained API and connect server.
- `Ryanmello07/connect` — beta connect client and provider.
- `Ryanmello07/proxy` — this repository.
