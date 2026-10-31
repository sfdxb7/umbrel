# Remote domain (VPS) mode

`patches/0004-remote-domain.patch` adds an opt-in mode for running this image on a VPS behind a TLS-terminating reverse proxy such as Traefik or Cloudflare. It is **off by default**. While it is off, umbrelOS behaves exactly like the upstream image.

With the mode on and the default naming (`<base domain>` = `alfalasi.io`):

| What | URL |
| --- | --- |
| Dashboard | `https://umbrel.alfalasi.io` |
| App login | `https://umbrel-auth.alfalasi.io` |
| Each app | `https://<appId>-umbrel.alfalasi.io` (e.g. `https://nextcloud-umbrel.alfalasi.io`) |

IP, LAN (`umbrel.local`), Tor and Tailscale access is unchanged. Those still use `http://<host>:<app port>` and the login page on port 2000.

## Turning it on

After first boot, sign in as the owner and open **Settings → Advanced → Remote domain**:

1. Check the base domain (pre-filled with `alfalasi.io`) and, if needed, the dashboard subdomain (`umbrel`), auth subdomain (`umbrel-auth`) and app subdomain template (`{app}-umbrel`).
2. Turn on **Enable remote domain** and save.
3. Optional: allow individual apps that have Umbrel login turned off (see below).

Saving applies immediately. umbreld rebuilds its routing and no restart is needed. The same setting is available over tRPC as `lanIngress.setRemoteDomain` (owner only), for example `{enabled: true, baseDomain: "alfalasi.io"}`. `lanIngress.getRemoteDomain` can be read by any account. Only the owner also sees the per-app allow list and app status.

The settings are stored in `umbrel.yaml` under `settings.remoteDomain`.

## How routing works

- umbreld's existing HTTPS listener on **port 443** looks at the `Host` header and **matches it exactly**:
  - the dashboard host serves the dashboard;
  - the auth host serves the app login page;
  - `<template>.<base domain>` serves that app through its app gateway.
- Any **other host under the base domain gets 404**. Hosts outside the base domain (IPs, `umbrel.local`, Tailscale names) go to the dashboard exactly as before.
- Auth and app hosts are only served over HTTPS (port 443). On port 80 they return `421`. This keeps the secure `__Host-` session cookies working.
- Login redirects, the post-login handoff URL and the login page's CSP `form-action` are built from the **configured domain** as exact `https://` origins. They are never taken from the request's `Host` header.
- Redirect paths are sanitised to same-origin paths. Schemes, `//host` and backslash tricks fall back to `/`.
- **Apps with Umbrel login turned off are blocked (403)** on the remote domain by default, because anyone on the internet could reach them. The owner can allow each such app in the settings panel.
- Only apps served through Umbrel's app gateway (normal web apps) get a remote host. Apps without a gateway and non-HTTP services (for example a Bitcoin node's P2P/RPC ports, or SMB) stay on LAN/Tailscale and return 404 on the remote domain.

## Reverse proxy requirements

The proxy must:

1. **Terminate TLS** for `umbrel.<domain>`, `umbrel-auth.<domain>` and `*-umbrel.<domain>`. Use a wildcard certificate for `*.<domain>`. The default names are all one level below the base domain, so a single `*.alfalasi.io` certificate (including Cloudflare's Universal SSL) covers them.
2. **Forward to umbreld's port 443 over HTTPS** and **skip upstream certificate verification**. umbreld presents its own local-CA certificate, which will not match the public name.
3. **Pass the original `Host` header through** (the default for Traefik and Cloudflare Tunnel).
4. Support WebSockets (the dashboard and many apps use them).

DNS: point a wildcard record `*.<domain>` (or `umbrel`, `umbrel-auth` plus one record per app) at the proxy. DNS cannot express `*-umbrel`, so a wildcard is the practical choice. Because unknown hosts under the base domain get a 404 from umbreld, a wildcard record is safe. Restrict the proxy rule to the hosts below if other services share the domain.

### Traefik example (file provider)

```yaml
http:
  serversTransports:
    umbrel-insecure:
      insecureSkipVerify: true   # umbreld uses its own local CA on :443

  routers:
    umbrel:
      rule: >-
        Host(`umbrel.alfalasi.io`) || Host(`umbrel-auth.alfalasi.io`) ||
        HostRegexp(`^[a-z0-9-]+-umbrel\.alfalasi\.io$`)
      entryPoints: [websecure]
      service: umbrel
      tls:
        certResolver: letsencrypt   # DNS challenge, needed for a wildcard
        domains:
          - main: alfalasi.io
            sans: ['*.alfalasi.io']

  services:
    umbrel:
      loadBalancer:
        serversTransport: umbrel-insecure
        passHostHeader: true
        servers:
          - url: https://umbrel:443   # or https://<host-ip>:443
```

(`HostRegexp` with a plain regular expression is Traefik v3 syntax. On Traefik v2 use ``HostRegexp(`{app:[a-z0-9-]+}-umbrel.alfalasi.io`)``.)

### Cloudflare Tunnel example

```yaml
ingress:
  - hostname: umbrel.alfalasi.io
    service: https://localhost:443
    originRequest: {noTLSVerify: true}
  - hostname: umbrel-auth.alfalasi.io
    service: https://localhost:443
    originRequest: {noTLSVerify: true}
  - hostname: "*.alfalasi.io"
    service: https://localhost:443
    originRequest: {noTLSVerify: true}
  - service: http_status:404
```

## Firewall: only expose 443 to the proxy

This mode trusts the proxy only to forward to port 443. You still need to lock down the VPS:

- Only the proxy should be able to reach umbreld's port 443. Do not publish 80, 443 or 2000 to the internet. For example, bind them to `127.0.0.1` or a private Docker network in `compose.yml`, or firewall them.
- **App ports are published directly by Docker** (e.g. `8085`) and bypass both this mode and many host firewalls (`ufw` does not filter Docker-published ports by default). Block them in the `DOCKER-USER` iptables chain or the VPS provider's firewall. Use LAN/Tailscale for port access.

## Login rate limiting (not built in)

umbreld does not rate-limit login attempts in this mode. Add it at the edge:

- Cloudflare: a WAF rate-limiting rule for `POST` requests to `/trpc/user.login*` on `umbrel.<domain>` (dashboard login) and to `/v1/account/login*` on `umbrel-auth.<domain>` (app login).
- Traefik: a `rateLimit` middleware on the router above (e.g. `average: 10`, `burst: 20`, per client IP).

Consider also turning on 2FA for every account.

## Known limits

- HSTS is not sent by umbreld (as before). Enable HSTS at Cloudflare/Traefik if you want it.
- Apps without an app gateway, and non-HTTP ports, are not routed on the remote domain.
- App paths that an app's manifest whitelists from Umbrel login stay public on its remote host, same as on the LAN.
- Home-screen shortcuts, widgets and the MCP `get_app_details` URL still use port URLs.
- umbreld does not trust `X-Forwarded-*` headers from the proxy in this version. The secure context comes from the proxy → :443 hop being HTTPS.
- The patch targets umbrelOS 2.0.0 and needs a rebase for each new Umbrel release (the Docker build fails loudly if it no longer applies).
