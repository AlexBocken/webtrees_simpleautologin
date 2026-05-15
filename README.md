[![License: GPL v3](https://img.shields.io/badge/License-GPL%20v3-blue.svg)](http://www.gnu.org/licenses/gpl-3.0)

# Simple Auto Login for Webtrees
This module provides a simple way to add a SSO auto login for webtrees in combination with a authentication proxy  (like [oauth2-proxy](https://github.com/oauth2-proxy/oauth2-proxy)).

## Installation
Requires webtrees 2.0.

### Using Git
If you are using ``git``, you could also clone the current main branch directly into your ``modules_v4`` directory 
by calling:

```
git clone https://github.com/fanningert/webtrees_simpleautologin.git modules_v4/webtrees_simpleautologin
```

### Manual installation
To manually install the module, perform the following steps:

1. Download the [latest release](https://github.com/fanningert/webtrees_simpleautologin/releases/latest).
1. Upload the downloaded file to your web server.
1. Unzip the package into your ``modules_v4`` directory.
1. Rename the folder to ``webtrees_simpleautologin``

## Enable
1. Visit the Control Panel
1. Click "All modules"
1. Scroll to "Simple Auto Login"
1. Check the checkbox for this module.
1. Scroll to the bottom.
1. Click the "save" button.
1. Add ``trusted_header_authenticated_user`` to the ``config.ini.php`` of webtrees
 
Known server parameter:
* oauth2-proxy: HTTP_X_FORWARDED_PREFERRED_USERNAME
* Apache mod_ssl: SSL_CLIENT_S_DN_CN
* general: REMOTE_USER
  
**Example:** trusted_header_authenticated_user="REMOTE_USER";

## Disable
1. Visit the Control Panel
1. Click "All modules"
1. Scroll to "Simple Auto Login"
1. Clear the checkbox for this module.
1. Scroll to the bottom.
1. Click the "save" button.

Alternatively, you can unload the module by renaming ``modules_v4/webtrees_simpleautologin/`` to ``modules_v4/webtrees_simpleautologin.disable/``

## Uninstall
It is safe to delete the ``webtrees_simpleautologin`` directory at any time.

# Landscape examples

## oauth2-proxy

In my installation, I have [Caddy](https://caddyserver.com/) as a first line reverse proxy. Behind this is a authentication proxy ([oauth2-proxy](https://github.com/oauth2-proxy/oauth2-proxy)) for the oauth authentication with [keycloak](https://www.keycloak.org/).

```
caddy -> oauth2-proxy -> webtrees
             |
             v
          Keycloak
```

### caddy configuration
```yaml
webtrees.example.com {
  reverse_proxy <oauth-proxy: https://x.x.x.x:port> {
    transport http {
      tls_insecure_skip_verify
    }
  }
}
```

### oauth2-proxy configuration
I am running oauth2-proxy as container (podman).
```bash
podman create --name "oauthproxy_core" --pod "oauthproxy" \
              -v "/etc/localtime:/etc/localtime:ro" \
              quay.io/oauth2-proxy/oauth2-proxy \
              --provider=oidc \
              --provider-display-name="Keycloak" \
              --client-id="app_webtrees" \
              --client-secret="<client-secret>" \
              --email-domain=* \
              --oidc-issuer-url="http(s)://<keycloak host>/auth/realms/<realm>" \
              --login-url="http(s)://<keycloak host>/auth/realms/<realm>/protocol/openid-connect/auth" \
              --redeem-url="http(s)://<keycloak host>/auth/realms/<realm>/protocol/openid-connect/token" \
              --validate-url="http(s)://<keycloak host>/auth/realms/<realm>/protocol/openid-connect/userinfo" \
              --allowed-group="<allowed_user_group>" \
              --whitelist-domain="<.example.com>" \
              --cookie-domain="<webtrees.example.com>" \
              --cookie-secure=true \
              --cookie-secret="${COOKIE_SECRET}" \
              --scope="openid profile email roles" \
              --http-address="127.0.0.1:4180" \
              --upstream="<webtrees url>" \
              --ssl-upstream-insecure-skip-verify="true" \
              --reverse-proxy="true" \
              --insecure-oidc-allow-unverified-email=true \
              --skip-provider-button=true

```
More information can be find [here](https://oauth2-proxy.github.io/oauth2-proxy/docs/configuration/oauth_provider#keycloak-auth-provider).

### Keycloak configuration

## nginx + authentik

This setup uses [authentik](https://goauthentik.io/) as the identity provider, fronted by nginx using `auth_request` to forward authentication to authentik's outpost. The authenticated username is passed to PHP via the `REMOTE_USER` fastcgi parameter, which webtrees consumes through `trusted_header_authenticated_user="REMOTE_USER"` in `config.ini.php`.

```
nginx (auth_request) -> authentik outpost -> webtrees (php-fpm)
```

### webtrees configuration

In `data/config.ini.php`:

```ini
trusted_header_authenticated_user="REMOTE_USER"
```

The username sent by authentik must match an existing webtrees user account. Create the user in the webtrees control panel first (the password does not matter, since login is delegated).

### authentik configuration

Create a **Proxy Provider** in the authentik admin UI:

- **Provider Name:** `Webtrees`
- **Authorization flow:** `default-provider-authorization-implicit-consent` (or explicit, your choice)
- **Mode:** `Forward auth (single application)`
- **External host:** `https://tree.example.com/` (the public URL of your webtrees instance)
- **Token validity:** `hours=24` (or whatever fits your policy)
- Under **Authentication settings**, enable **Intercept header authentication**.

Then create an **Application** that uses this provider:

- **Application Name:** `Webtrees`
- **Slug:** `webtrees`
- **Provider:** `Webtrees` (the proxy provider above)
- **Policy engine mode:** `ANY` (use group-binding policies to restrict access)

Finally, ensure the application is bound to your authentik **Outpost** (the embedded outpost listening on port 9000 by default), so `/outpost.goauthentik.io/` is served by it.

### nginx configuration

The trick with PHP applications is that `auth_request` must guard the `location ~ \.php$` block (where the actual processing happens), not just `location /`. Otherwise the rewrite from `/` to `/index.php` reaches the php-fpm block unauthenticated.

The `/outpost.goauthentik.io/` location must remain unauthenticated so the outpost can be reached for the auth handshake. The `@goauthentik_proxy_signin` named location handles the 401 redirect to start the SSO flow.

```nginx
server {
    root /usr/share/webapps/webtrees;

    client_max_body_size 50M;

    location /public {
        expires 365d;
        access_log off;
    }

    # Deny access to sensitive paths.
    location /.git    { deny all; }
    location /data    { deny all; }
    location /app     { deny all; }
    location /modules { deny all; }
    location /resources { deny all; }
    location /vendor  { deny all; }

    # Front-controller rewrite.
    location / {
        rewrite ^ /index.php last;
    }

    listen 443 ssl;
    http2 on;
    server_name tree.example.com;

    ssl_certificate     /etc/letsencrypt/live/tree.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/tree.example.com/privkey.pem;
    include             /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam         /etc/letsencrypt/ssl-dhparams.pem;

    # Larger buffers — authentik headers can be big.
    proxy_buffers       8 16k;
    proxy_buffer_size   32k;

    location ~ \.php$ {
        try_files $fastcgi_script_name =404;
        include fastcgi_params;

        fastcgi_pass         unix:/run/php-fpm/php-fpm.sock;
        fastcgi_index        index.php;
        fastcgi_buffers      8 16k;
        fastcgi_buffer_size  32k;

        fastcgi_param DOCUMENT_ROOT  $realpath_root;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;

        # --- authentik forward-auth ---
        auth_request     /outpost.goauthentik.io/auth/nginx;
        error_page       401 = @goauthentik_proxy_signin;
        auth_request_set $auth_cookie $upstream_http_set_cookie;
        add_header       Set-Cookie $auth_cookie;

        # Translate outpost response headers into variables.
        auth_request_set $authentik_username $upstream_http_x_authentik_username;
        auth_request_set $authentik_groups   $upstream_http_x_authentik_groups;
        auth_request_set $authentik_email    $upstream_http_x_authentik_email;
        auth_request_set $authentik_name     $upstream_http_x_authentik_name;
        auth_request_set $authentik_uid      $upstream_http_x_authentik_uid;

        # Hand them to PHP. REMOTE_USER is what webtrees reads.
        fastcgi_param X-authentik-username $authentik_username;
        fastcgi_param REMOTE_USER          $authentik_username;
        fastcgi_param X-authentik-groups   $authentik_groups;
        fastcgi_param X-authentik-email    $authentik_email;
        fastcgi_param X-authentik-name     $authentik_name;
        fastcgi_param X-authentik-uid      $authentik_uid;
    }

    # Outpost endpoint must be reachable without auth.
    location /outpost.goauthentik.io {
        proxy_pass              http://127.0.0.1:9000/outpost.goauthentik.io;
        proxy_set_header        Host $host;
        proxy_set_header        X-Original-URL $scheme://$http_host$request_uri;
        add_header              Set-Cookie $auth_cookie;
        auth_request_set        $auth_cookie $upstream_http_set_cookie;
        proxy_pass_request_body off;
        proxy_set_header        Content-Length "";
    }

    # Redirect 401 responses to the SSO start URL.
    location @goauthentik_proxy_signin {
        internal;
        add_header Set-Cookie $auth_cookie;
        return 302 /outpost.goauthentik.io/start?rd=$request_uri;
    }
}

server {
    if ($host = tree.example.com) {
        return 301 https://$host$request_uri;
    }
    listen 80;
    server_name tree.example.com;
    return 404;
}
```

Replace `tree.example.com` with your hostname, and adjust the outpost address (`127.0.0.1:9000`) if your authentik instance lives elsewhere. If authentik runs on a separate host, swap the single-application proxy mode for **Forward auth (domain level)** and use the domain-level redirect form documented in the authentik docs.

### Logout

Webtrees' logout link clears the local session, but the next request will be re-authenticated by authentik immediately. To fully sign out, redirect users to `/outpost.goauthentik.io/sign_out` (e.g. via a custom logout link or a webtrees customisation).
