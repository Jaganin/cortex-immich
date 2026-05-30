# Immich — Authentication

Immich uses Authelia as its OIDC provider. There is no Authelia forward-auth in front of Immich: the route `photo.jaganin.duckdns.org` is bypassed in Authelia's access control, and Immich handles authentication natively via OAuth/OIDC.

## Architecture

```
User (browser or Android app)
  │
  ▼
Traefik → photo.jaganin.duckdns.org → Immich :2283
  │
  └─ (no Authelia middleware — Immich handles auth itself)

Immich OAuth flow:
  Immich → https://auth.jaganin.duckdns.org (Authelia OIDC)
```

## Authelia OIDC client (cortex-internet)

Defined in `/opt/cortex-internet/authelia/configuration.yml`:

```yaml
- client_id: immich
  client_name: Immich
  client_secret: '<argon2 hash>'   # plain value: AUTHELIA_OIDC_IMMICH_SECRET in cortex-internet/.env
  authorization_policy: two_factor
  consent_mode: implicit
  pre_configured_consent_duration: 1y
  redirect_uris:
    - https://photo.jaganin.duckdns.org/auth/login   # web browser callback
    - app.immich:///oauth-callback                    # Android/iOS deep link
  scopes:
    - openid
    - profile
    - email
  token_endpoint_auth_method: client_secret_post
```

## Immich OAuth configuration (admin UI)

These settings are configured in the Immich admin UI at  
**Administration → Settings → OAuth** — not via environment variables.

| Setting | Value |
|---|---|
| Enabled | ✓ |
| Issuer URL | `https://auth.jaganin.duckdns.org` |
| Client ID | `immich` |
| Client Secret | see `AUTHELIA_OIDC_IMMICH_SECRET` in `cortex-internet/.env` |
| Scope | `openid profile email` |
| Storage Label Claim | `preferred_username` |
| Auto-register users | ✓ |
| Mobile Redirect URI Override | `app.immich:///oauth-callback` |

If OAuth is not configured in the admin UI, users see only a password login form and the Android app cannot authenticate.

## Login flow

**Web browser**: go to `https://photo.jaganin.duckdns.org` → click "Sign in with OAuth" → Authelia login + 2FA → redirected back to Immich.

**Android app**: enter server URL `https://photo.jaganin.duckdns.org` → tap "Sign in with OAuth" → browser/WebView opens Authelia → 2FA → deep link `app.immich:///oauth-callback` returns to the app.

## User provisioning

With "Auto-register users" enabled, a new Immich account is created on first OAuth login. The account email comes from the `email` OIDC claim. Existing local accounts can be linked to OIDC by logging in with OAuth using the same email.

## Troubleshooting

- **No "Sign in with OAuth" button in the app**: OAuth is not enabled in Immich admin settings.
- **Redirect URI mismatch error**: check that `app.immich:///oauth-callback` is listed in the Authelia client's `redirect_uris`.
- **"Invalid client" error**: client secret mismatch between Immich admin settings and `AUTHELIA_OIDC_IMMICH_SECRET`.
- **2FA loop**: the Authelia session cookie domain is `jaganin.duckdns.org` — make sure the browser/WebView accepts it.
- **500 "Failed to finish oauth" / Authelia "Could not save the consent session"**: the SQLite database file is read-only for the Authelia container user. Fix:
  ```bash
  sudo chown 1000:1000 /opt/cortex-internet/authelia/db.sqlite3
  cd /opt/cortex-internet && docker compose restart authelia
  ```
  The file is created by root on first run but Authelia runs as uid 1000.
