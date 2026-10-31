# Remote domain (VPS) mode

`patches/0004-remote-domain.patch` adds an opt-in mode for running this image on a VPS behind a TLS-terminating reverse proxy such as Traefik or Cloudflare. It is **off by default**. While it is off, umbrelOS behaves exactly like the upstream image.

With the mode on and the default naming (`<base domain>` = `example.com`):

| What | URL |
| --- | --- |
| Dashboard | `https://umbrel.example.com` |
| App login | `https://umbrel-auth.example.com` |
| Each app | `https://<appId>-umbrel.example.com` (e.g. `https://nextcloud-umbrel.example.com`) |

IP, LAN (`umbrel.local`), Tor and Tailscale access is unchanged. Those still use `http://<host>:<app port>` and the login page on port 2000.

## Turning it on

After first boot, sign in as the owner and open **Settings → Advanced → Remote domain**:

1. Enter your base domain, for example `example.com`. If no base domain has been saved and you opened the dashboard on a domain name, the field is pre-filled with a suggestion (`umbrel.example.com` → `example.com`). Check it before saving. The suggestion never turns the mode on. The dashboard subdomain (`umbrel`), auth subdomain (`umbrel-auth`), app subdomain template (`{app}-umbrel`) and the Origin rewrite opt-out are under **Advanced**. The defaults are fine for most setups.
2. Turn on **Enable remote domain** and save.
3. Optional: allow individual apps that have Umbrel login turned off (see below).

Saving applies immediately. umbreld rebuilds its routing and no restart is needed. The same setting is available over tRPC as `lanIngress.setRemoteDomain` (owner only), for example `{enabled: true, baseDomain: "example.com"}`. `lanIngress.getRemoteDomain` can be read by any account. Only the owner also sees the per-app allow list and app status.

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

1. **Terminate TLS** for `umbrel.<domain>`, `umbrel-auth.<domain>` and `*-umbrel.<domain>`. Use a wildcard certificate for `*.<domain>`. The default names are all one level below the base domain, so a single `*.example.com` certificate (including Cloudflare's Universal SSL) covers them.
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
        Host(`umbrel.example.com`) || Host(`umbrel-auth.example.com`) ||
        HostRegexp(`^[a-z0-9-]+-umbrel\.example\.com$`)
      entryPoints: [websecure]
      service: umbrel
      tls:
        certResolver: letsencrypt   # DNS challenge, needed for a wildcard
        domains:
          - main: example.com
            sans: ['*.example.com']

  services:
    umbrel:
      loadBalancer:
        serversTransport: umbrel-insecure
        passHostHeader: true
        servers:
          - url: https://umbrel:443   # or https://<host-ip>:443
```

(`HostRegexp` with a plain regular expression is Traefik v3 syntax. On Traefik v2 use ``HostRegexp(`{app:[a-z0-9-]+}-umbrel.example.com`)``.)

### Cloudflare Tunnel example

```yaml
ingress:
  - hostname: umbrel.example.com
    service: https://localhost:443
    originRequest: {noTLSVerify: true}
  - hostname: umbrel-auth.example.com
    service: https://localhost:443
    originRequest: {noTLSVerify: true}
  - hostname: "*.example.com"
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
- Upstream apps see a LAN `Origin`/`Referer` but the remote `Host`. See "Origin and Referer rewrite" below.
- Home-screen shortcuts, widgets and the MCP `get_app_details` URL still use port URLs.
- umbreld does not trust `X-Forwarded-*` headers from the proxy in this version. The secure context comes from the proxy → :443 hop being HTTPS.
- The patch targets umbrelOS 2.0.0 and needs a rebase for each new Umbrel release (the Docker build fails loudly if it no longer applies).

## Origin and Referer rewrite

Some apps (for example Agent Zero on port 3015) only accept requests whose `Origin` matches the LAN URL they expect, and reject `https://agent-zero-umbrel.example.com`. For remote-domain requests that have passed Umbrel app login (or that come from an app you allowed without login), the app gateway:

- rewrites `Origin: https://<appId>-umbrel.<domain>` to `http://<umbrel hostname>:<app port>`. The hostname is the device hostname with `.local` added, `umbrel.local` by default. This applies to normal HTTP requests and WebSocket upgrades;
- rewrites a `Referer` on that same origin to the LAN origin, keeping its path and query;
- sets `X-Forwarded-Host: <appId>-umbrel.<domain>` and `X-Forwarded-Proto: https`.

Only an exact match on the app's own remote origin is rewritten. Foreign origins and other apps' origins reach the app unchanged, so the app's own cross-origin protection still works. Requests with no `Origin` are left alone. `Host` is **not** rewritten, so apps that build absolute URLs from `Host` keep producing remote URLs. LAN, IP and Tailscale requests are never rewritten.

**Per-app override.** If an app compares `Origin` with `Host` (some WebSocket servers do this by default), the rewrite makes them differ. In that case add the app ID under **Settings → Advanced → Remote domain → Advanced → Send the original Origin header to these apps**, or over tRPC: `lanIngress.setRemoteDomain({..., originRewriteDisabledApps: ["some-app"]})`. Those apps still get `X-Forwarded-Host`/`X-Forwarded-Proto`, but their `Origin` and `Referer` are passed through unchanged.

## Revision 2

1. **New and changed apps are routed without toggling the mode.** Before this revision, an app installed after the mode was turned on only got a remote host after the mode was turned off and on again. The cause was that the UI refetched the app list on `apps:state:change` before the app's gateway was listening, and nothing told it to fetch again. Now:
   - While the mode is on, umbreld refreshes routing (debounced, 250 ms) on every `apps:state:change` and `apps:settings:change`. That covers install, uninstall, update, start, stop and auth/allow-list changes.
   - Whenever the set of routed hosts or their auth state changes, umbreld emits a payload-free `lan-ingress:remote-routes:change` event. The UI then refetches the app list and the remote-domain settings, so the new `https://<app>-umbrel.<domain>` link appears by itself.
   - The host→app map and allow-list status are rebuilt even if part of a refresh fails.
   - An app whose gateway is still starting returns `503` with `Retry-After: 5` and `Refresh: 5` instead of `404`, so the browser retries on its own.
   - Uninstalled or stopped apps lose their gateway, drop out of the map on the next refresh, and return `404`.
2. **No real domain is used.** Code, tests and docs use `example.com`, `<app>-umbrel.example.com` and `umbrel-auth.example.com`. There is no built-in default base domain any more. It is empty until you set one.
3. **Simpler settings.** The base domain is the main field. Dashboard subdomain, auth subdomain, app template and the Origin rewrite opt-out are in a collapsed **Advanced** section, with a note that the defaults are fine for most setups.
4. **Base domain suggestion.** When nothing is saved yet, the field is pre-filled from the address you opened the dashboard on, with the first label dropped (`umbrel.example.com` → `example.com`). You confirm it by saving. The mode is never turned on automatically. No suggestion is made for IPv4/IPv6 addresses, `localhost`, `*.local`, Tailscale `*.ts.net`, `.onion`, other private-use suffixes (`.internal`, `.lan`, `.home`, `.test`), or single-label and two-label hosts.
5. **Origin/Referer rewrite in the app gateway** (generic, fixes Agent Zero). See "Origin and Referer rewrite" above, including the per-app `originRewriteDisabledApps` override and the `Origin`-vs-`Host` caveat.

Tests added: hostname suggestion (UI), lifecycle refresh, route-change event and uninstall removal (`lan-ingress-remote-lifecycle.unit.test.ts`), the 503 for starting apps, header rewrite helpers, and gateway integration tests. The integration tests send real HTTP and WebSocket upgrade requests to an echo upstream and cover: same-app origin rewritten, foreign/other-app origin untouched, referer path kept, no-origin untouched, LAN requests untouched, and the per-app opt-out.
