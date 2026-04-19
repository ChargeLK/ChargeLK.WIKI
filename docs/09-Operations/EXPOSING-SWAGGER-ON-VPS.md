# Exposing Swagger Documentation on the Production VPS

This guide explains how to safely expose Swagger UI for both `ChargeLK.OCPPServer` and `ChargeLK.UM` on a single VPS so that frontend developers can browse and try the APIs against the production-test environment.

Both services already have Swagger registered in code and `Swagger:Enabled = true` in their `appsettings.Production.json`. The remaining work is reverse proxy configuration, header forwarding, and access control.

---

## 1. Code-side prerequisites (already done)

Both `Program.cs` files now register `UseForwardedHeaders` immediately after `builder.Build()`, so ASP.NET sees the public scheme/host instead of the loopback hop. Without this, Swashbuckle generates `swagger.json` URLs against `http://127.0.0.1:<port>` and the "Try it out" button breaks.

```csharp
using Microsoft.AspNetCore.HttpOverrides;

// ...
app.UseForwardedHeaders(new ForwardedHeadersOptions
{
    ForwardedHeaders = ForwardedHeaders.XForwardedFor
                     | ForwardedHeaders.XForwardedProto
                     | ForwardedHeaders.XForwardedHost,
    KnownNetworks = { },
    KnownProxies = { }
});
```

`KnownNetworks` and `KnownProxies` are intentionally cleared. ASP.NET defaults only trust loopback, which technically works since Nginx forwards from `127.0.0.1`, but clearing them removes a footgun if Kestrel is ever moved off loopback. Kestrel must remain bound to localhost only — the public surface area is Nginx.

The Swagger gate in both services:

```csharp
if (app.Environment.IsDevelopment() || builder.Configuration.GetValue<bool>("Swagger:Enabled"))
{
    app.UseSwaggerWithUi();
}
```

So Swagger is exposed in production whenever `Swagger:Enabled = true` (already set in both production settings files).

---

## 2. Confirm the production environment is active

`appsettings.Production.json` is only loaded when `ASPNETCORE_ENVIRONMENT=Production`. Verify each systemd unit:

```bash
sudo systemctl cat chargelk-ocpp-imesh
sudo systemctl cat chargelk-um            # or whatever the UM service is named
```

Each unit (or its drop-in at `/etc/systemd/system/<service>.service.d/secrets.conf`) must contain:

```ini
Environment=ASPNETCORE_ENVIRONMENT=Production
```

If you'd rather not touch the JSON file, you can override the flag at the systemd level — env vars beat JSON:

```ini
Environment=Swagger__Enabled=true
```

After any change:

```bash
sudo systemctl daemon-reload
sudo systemctl restart chargelk-ocpp-imesh
sudo systemctl restart chargelk-um
```

---

## 3. Reverse proxy topology — both services on the same VPS

The two services run as separate systemd units bound to different localhost ports:

| Service | Localhost port (example) |
| ------- | ------------------------ |
| `chargelk-ocpp-imesh` (OCPPServer) | `127.0.0.1:5001` |
| `chargelk-um` (UM)                 | `127.0.0.1:5000` |

> Adjust the numbers below to match whatever `ASPNETCORE_URLS` (or `--urls`) your systemd units actually set. You can confirm with `sudo ss -tlnp | grep dotnet`.

Two ways to expose them through Nginx — pick one.

### Option A — Subdomains (recommended)

Each service gets its own hostname and TLS certificate. Swagger lives at `/swagger` on each, exactly as it does locally. **No code changes beyond `UseForwardedHeaders` are required.**

#### `/etc/nginx/sites-available/ocpp-api.conf`

```nginx
server {
    listen 443 ssl http2;
    server_name ocpp-api.example.com;

    ssl_certificate     /etc/letsencrypt/live/ocpp-api.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/ocpp-api.example.com/privkey.pem;

    location / {
        proxy_pass         http://127.0.0.1:5001;
        proxy_http_version 1.1;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_set_header   X-Forwarded-Host  $host;

        # OCPP charge points need WebSocket upgrade:
        proxy_set_header   Upgrade           $http_upgrade;
        proxy_set_header   Connection        "upgrade";
        proxy_read_timeout 3600s;
    }

    # Lock Swagger behind basic auth — see section 4.
    location /swagger {
        auth_basic           "ChargeLK API docs";
        auth_basic_user_file /etc/nginx/.htpasswd-chargelk;

        proxy_pass         http://127.0.0.1:5001;
        proxy_http_version 1.1;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_set_header   X-Forwarded-Host  $host;
    }
}
```

#### `/etc/nginx/sites-available/um-api.conf`

```nginx
server {
    listen 443 ssl http2;
    server_name um-api.example.com;

    ssl_certificate     /etc/letsencrypt/live/um-api.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/um-api.example.com/privkey.pem;

    location / {
        proxy_pass         http://127.0.0.1:5000;
        proxy_http_version 1.1;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_set_header   X-Forwarded-Host  $host;
        # UM has no WebSocket — Upgrade headers not needed.
    }

    location /swagger {
        auth_basic           "ChargeLK API docs";
        auth_basic_user_file /etc/nginx/.htpasswd-chargelk;

        proxy_pass         http://127.0.0.1:5000;
        proxy_http_version 1.1;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_set_header   X-Forwarded-Host  $host;
    }
}

# Redirect plain HTTP to HTTPS for both
server {
    listen 80;
    server_name ocpp-api.example.com um-api.example.com;
    return 301 https://$host$request_uri;
}
```

#### Enable the sites

```bash
sudo ln -s /etc/nginx/sites-available/ocpp-api.conf /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/um-api.conf   /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

#### TLS

```bash
sudo certbot --nginx -d ocpp-api.example.com -d um-api.example.com
```

#### DNS

Point both `ocpp-api` and `um-api` A records at the VPS IP.

#### Frontend devs get

- `https://ocpp-api.example.com/swagger`
- `https://um-api.example.com/swagger`

### Option B — Path prefixes on a single host

Both services share one hostname, e.g. `api.example.com/ocpp/...` and `api.example.com/um/...`. One certificate, one server block, but **path prefixes need additional code changes** because Swashbuckle does not natively know it lives under a subpath.

#### Nginx

```nginx
server {
    listen 443 ssl http2;
    server_name api.example.com;

    ssl_certificate     /etc/letsencrypt/live/api.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;

    # ---- OCPP ----
    location /ocpp/ {
        proxy_pass         http://127.0.0.1:5001/;   # trailing slash strips /ocpp
        proxy_http_version 1.1;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_set_header   X-Forwarded-Host  $host;
        proxy_set_header   X-Forwarded-Prefix /ocpp;     # important

        proxy_set_header   Upgrade           $http_upgrade;
        proxy_set_header   Connection        "upgrade";
        proxy_read_timeout 3600s;
    }

    # ---- UM ----
    location /um/ {
        proxy_pass         http://127.0.0.1:5000/;
        proxy_http_version 1.1;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_set_header   X-Forwarded-Host  $host;
        proxy_set_header   X-Forwarded-Prefix /um;
    }
}
```

#### Code change in both `Program.cs`

In addition to `UseForwardedHeaders`, both services need to honor the path base from the proxy:

```csharp
var forwardedPrefix = app.Configuration["ForwardedPathBase"];
if (!string.IsNullOrEmpty(forwardedPrefix))
{
    app.UsePathBase(forwardedPrefix);
}
```

And `UseSwaggerWithUi` (in `OcppServer.API/Extentions/ServiceCollectionExtensions.cs` and the equivalent in UM's `ChargeLK.API/Extensions/`) must inject the public server URL into `swagger.json`:

```csharp
internal static IApplicationBuilder UseSwaggerWithUi(this IApplicationBuilder app, string? pathBase = null)
{
    app.UseSwagger(c =>
    {
        if (!string.IsNullOrEmpty(pathBase))
        {
            c.PreSerializeFilters.Add((swagger, httpReq) =>
            {
                swagger.Servers = new List<OpenApiServer>
                {
                    new() { Url = $"{httpReq.Scheme}://{httpReq.Host.Value}{pathBase}" }
                };
            });
        }
    });
    app.UseSwaggerUI();
    return app;
}
```

Then in `Program.cs`:

```csharp
if (app.Environment.IsDevelopment() || builder.Configuration.GetValue<bool>("Swagger:Enabled"))
{
    app.UseSwaggerWithUi(builder.Configuration["ForwardedPathBase"]);
}
```

#### Set the prefix per service via the systemd drop-in

```ini
# /etc/systemd/system/chargelk-ocpp-imesh.service.d/secrets.conf
Environment=ForwardedPathBase=/ocpp
```

```ini
# /etc/systemd/system/chargelk-um.service.d/secrets.conf
Environment=ForwardedPathBase=/um
```

`daemon-reload` + restart both services.

#### Frontend devs get

- `https://api.example.com/ocpp/swagger`
- `https://api.example.com/um/swagger`

### Which option to pick?

| | Subdomains (A) | Path prefixes (B) |
| - | - | - |
| Code changes | `UseForwardedHeaders` only | `UseForwardedHeaders` + `UsePathBase` + Swagger server URL tweak |
| TLS certs | Two (or one wildcard) | One |
| DNS records | Two A records | One A record |
| Risk of subtle Swagger/route bugs | Low | Medium — path-rewriting is fiddly |
| OCPP WebSocket URL for charge points | `wss://ocpp-api.example.com/` | `wss://api.example.com/ocpp/` (every charge point needs reconfiguring) |

**Recommendation: Option A (subdomains).** It's the path of least resistance and avoids touching Swashbuckle internals.

---

## 4. Lock Swagger down — do not leave it fully public

`Swagger:Enabled = true` exposes the full API surface — including admin endpoints — to anyone who finds the URL. The JWT bearer scheme protects the *endpoints*, not the schema document, and the schema itself is sensitive (it leaks every route, parameter, and enum).

### Option A — HTTP basic auth at the Nginx layer (simplest)

Already shown in the Nginx blocks above via `auth_basic` / `auth_basic_user_file`. Create the password file:

```bash
sudo apt install apache2-utils    # if htpasswd isn't installed
sudo htpasswd -c /etc/nginx/.htpasswd-chargelk frontend
# enter a password — share it with frontend devs out-of-band
```

To add more users later, drop the `-c` flag:

```bash
sudo htpasswd /etc/nginx/.htpasswd-chargelk another-dev
```

### Option B — IP allow-list

If your frontend devs have stable IPs (e.g. office network), restrict by source:

```nginx
location /swagger {
    allow 203.0.113.0/24;   # office
    deny  all;

    proxy_pass         http://127.0.0.1:5001;
    proxy_set_header   Host              $host;
    proxy_set_header   X-Forwarded-Proto $scheme;
    proxy_set_header   X-Forwarded-Host  $host;
}
```

You can combine both — `allow` + `auth_basic` — for defense in depth.

---

## 5. CORS

CORS only matters if a frontend hosted on a *different* origin calls the API. If the dev opens Swagger UI directly at `https://ocpp-api.example.com/swagger`, the "Try it out" button calls the same origin and CORS does not apply.

If the dev runs a local frontend (e.g. `http://localhost:3000`) against the production-test API, that origin must be in `AllowedOrigins` in the relevant `appsettings.Production.json`:

- **OCPP** already lists `http://localhost:3000` and the VPS IP variants — fine for local React dev.
- **UM** already lists `http://localhost:3000`, `http://localhost:4200`, and `https://app.chargelk.com` — fine for typical setups.

Add new origins (with the exact scheme + port) and restart the service to pick them up.

---

## 6. Smoke test

From your machine, after the deploy lands:

```bash
# Schema should be 200 + JSON
curl -i https://ocpp-api.example.com/swagger/v1/swagger.json
curl -i https://um-api.example.com/swagger/v1/swagger.json

# Swagger UI should challenge with basic auth, then render
xdg-open https://ocpp-api.example.com/swagger
xdg-open https://um-api.example.com/swagger
```

Once the UI loads, click the **Authorize** button (the lock icon) and paste a JWT obtained from the UM auth endpoint. Both services already wire up JWT bearer via `AddSwaggerGenWithAuth()`.

If the OCPP `swagger.json` URL is `http://127.0.0.1:5001/...` instead of `https://ocpp-api.example.com/...`, `UseForwardedHeaders` is not running early enough — make sure it sits immediately after `builder.Build()` and before any other middleware.

If charge points start failing to connect after the Nginx change, check the WebSocket `Upgrade`/`Connection` headers in the OCPP `location /` block — those are mandatory for OCPP to work.

---

## 7. End-to-end action checklist

1. ✅ `UseForwardedHeaders(...)` is in both `Program.cs` files (committed in code).
2. Confirm `ASPNETCORE_ENVIRONMENT=Production` is set in both systemd units.
3. Add the Nginx server blocks from Option A to `/etc/nginx/sites-available/` and symlink into `sites-enabled/`.
4. `sudo nginx -t && sudo systemctl reload nginx`
5. Issue TLS certs with Certbot for both subdomains.
6. Create `/etc/nginx/.htpasswd-chargelk` with `htpasswd` and share credentials with frontend devs out-of-band.
7. Deploy the latest `Program.cs` via the existing GitHub Actions workflow.
8. Smoke test both `/swagger` URLs and the underlying `swagger.json`.
9. Hand the two URLs and the basic-auth credentials to your frontend devs.
