# platform-iam

Declarative Keycloak configuration for the **Hubinity** ecosystem.
This repo is the **source of truth** for realms, clients, roles, scopes and dev users on both platforms:

- `hibit` realm — HiBit (loja / assistência técnica / caixa)
- `star-coffee` realm — Star Coffee (cafeteria, totem PWA)

Tested against **Keycloak 26.x** (modern realm export schema). Do **not** mix with legacy 12.x exports.

---

## Layout

```
platform-iam/
├── README.md                       ← this file
├── realms/
│   ├── hibit-realm.json            ← realm `hibit` (full export)
│   └── star-coffee-realm.json      ← realm `star-coffee` (full export)
└── clients/                        ← reserved for per-client patches/overlays (future)
```

A "full export" is everything Keycloak needs to recreate a realm on import: realm settings, roles, clients, client scopes, users (with dev credentials) and protocol mappers.

---

## Importing in dev (docker-compose / docker)

The whole `realms/` directory should be mounted into `/opt/keycloak/data/import` and Keycloak should be started with `--import-realm`.

### Minimal one-off run

```bash
docker run --rm \
  -p 8080:8080 \
  -e KEYCLOAK_ADMIN=admin \
  -e KEYCLOAK_ADMIN_PASSWORD=admin \
  -v "$(pwd)/realms:/opt/keycloak/data/import:ro" \
  quay.io/keycloak/keycloak:26.0 \
  start-dev --import-realm
```

After it boots:
- Admin console → http://localhost:8080 (admin / admin)
- Both realms `hibit` and `star-coffee` already populated.

### docker-compose snippet

```yaml
services:
  keycloak:
    image: quay.io/keycloak/keycloak:26.0
    command: ["start-dev", "--import-realm"]
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
    ports: ["8080:8080"]
    volumes:
      - ../platform-iam/realms:/opt/keycloak/data/import:ro
```

Note: `--import-realm` only imports realms that **do not already exist** in the underlying DB. To force re-import wipe the volume or use the override below.

To overwrite an existing realm during dev:

```bash
docker exec -it <kc-container> \
  /opt/keycloak/bin/kc.sh import \
  --dir /opt/keycloak/data/import \
  --override true
```

---

## Importing in prod (Railway / managed Postgres)

`--import-realm` is fine for first-boot, but for prod we prefer the **Admin REST API** so we can run it idempotently from CI.

Two options:

1. **Bake the JSON into the image / volume** and use the same `start --import-realm` (one-shot init).
2. **Push via Admin API** from CI: authenticate against `master` realm with admin creds, then `POST /admin/realms` with the JSON body — gives full control over partial vs full replacement.

In prod, **secrets in this repo are placeholders**. Real client secrets are injected via Railway env (`KC_DB_PASSWORD`, `HB_CATALOG_CLIENT_SECRET`, etc.) and rotated separately. The JSON `secret` fields exist only so dev works without extra setup.

---

## Exporting changes back from a running Keycloak

If you edit a realm in the Admin UI for prototyping, re-sync it into JSON like this:

```bash
docker exec -it <kc-container> \
  /opt/keycloak/bin/kc.sh export \
  --dir /opt/keycloak/data/import \
  --realm hibit \
  --users realm_file
```

Then copy the file out, diff against the version in this repo, and PR it.

For both realms at once: omit `--realm`. Keycloak writes `<realm>-realm.json` and (if users are split) `<realm>-users-0.json`. We keep users **inline** in the realm file (`--users realm_file`) to keep the diff readable.

---

## Versioning policy

- Every realm change goes through a **PR**.
- **Never edit the JSON by hand and ship** without first importing it into a local Keycloak and confirming it loads cleanly.
- **Never** rotate or change a `clientId`, role name or `realm` name without coordinating with the service teams — these are referenced by `application.yaml` files across the other 11 repos.
- Client `secret` values in this repo are dev-only placeholders. Rotating them is a no-op outside of dev.
- New scopes (`*:read`, `*:write`) must be added to the corresponding service's `defaultClientScopes` *and* the service must be updated to enforce them.

---

## Dev users — quick reference

### Realm `hibit` (http://localhost:8080/realms/hibit)

| Username      | Password   | Role             | Use it from                 |
| ------------- | ---------- | ---------------- | --------------------------- |
| `admin-hibit` | `admin123` | `admin`          | any HiBit web app           |
| `tech1`       | `tech123`  | `tecnico`        | `hb-support-web` (port 4201)|
| `atendente1`  | `atend123` | `atendente`      | `hb-support-web` (port 4201)|
| `caixa1`      | `caixa123` | `operador-caixa` | `hb-cashier-web` (port 4202)|

Backend service-accounts (client credentials grant — no human login):

| `clientId`           | `secret`             | Default scopes                       |
| -------------------- | -------------------- | ------------------------------------ |
| `hb-catalog-service` | `dev-catalog-secret` | `catalog:read`, `catalog:write`      |
| `hb-support-service` | `dev-support-secret` | `support:read`, `support:write`      |
| `hb-cashier-service` | `dev-cashier-secret` | `cashier:read`, `cashier:write`      |

Public SPAs (PKCE, no secret):

| `clientId`       | Redirect URI              | Web origin                |
| ---------------- | ------------------------- | ------------------------- |
| `hb-catalog-web` | `http://localhost:4200/*` | `http://localhost:4200`   |
| `hb-support-web` | `http://localhost:4201/*` | `http://localhost:4201`   |
| `hb-cashier-web` | `http://localhost:4202/*` | `http://localhost:4202`   |

### Realm `star-coffee` (http://localhost:8080/realms/star-coffee)

| Username   | Password   | Role                | Use it from                |
| ---------- | ---------- | ------------------- | -------------------------- |
| `admin-sc` | `admin123` | `admin`             | any Star Coffee surface    |
| `gerente1` | `ger123`   | `gerente-cafeteria` | gestão / fechamento totem  |

Backend & device clients:

| `clientId`         | Type            | `secret`                  | Notes                                                                       |
| ------------------ | --------------- | ------------------------- | --------------------------------------------------------------------------- |
| `sc-order-service` | confidential SA | `dev-order-secret`        | Has `catalog:read` default scope (cross-realm trust via API gateway later). |
| `sc-totem-web`     | public PKCE     | —                         | Kiosk PWA, port 4203.                                                       |
| `device-totem`     | confidential    | `dev-device-totem-secret` | Password grant + 30d offline session for kiosk bootstrap.                   |

---

## Cross-realm consideration: `catalog:read` in `star-coffee`

`sc-order-service` lives in the `star-coffee` realm but needs to read the HiBit product catalog. For now we model this as a `catalog:read` **scope present in the `star-coffee` realm** so tokens minted there carry the claim. The actual cross-realm trust (token exchange or relayed JWT) will be wired up at the API gateway layer in a later phase — see `platform-api-gateway/`. This file does *not* set up federated identity between realms yet.

---

## Security notes / DEV ONLY warning

> **All passwords and `secret` values in this repo are for local development only.**
> Do not reuse them in any environment that touches real data.
> Production deploys must:
> - Replace every `secret` field via env injection at import time, or by patching via Admin API right after first import.
> - Replace every test user (`admin-hibit`, `tech1`, `atendente1`, `caixa1`, `admin-sc`, `gerente1`) with real identities — these dev users should not be created in prod realms at all.
> - Set `bruteForceProtected: true` (already on), enforce a real password policy (we currently use `length(6)` for dev convenience), and require email verification.

The `KC_DB_*` env vars and the Keycloak admin bootstrap credentials are provisioned per-environment by `platform-infra/` and are *not* part of this repo.

---

## Schema compatibility

These files target the Keycloak **26.x partialImport / fullImport** JSON schema. Notable choices:

- `protocol: openid-connect` on every client; no SAML.
- PKCE enforced via the per-client attribute `pkce.code.challenge.method: S256`.
- `clientScopes` array is realm-level; per-client opt-in via `defaultClientScopes` and `optionalClientScopes`.
- The `device-totem` client uses per-client session attributes (`client.session.*`, `client.offline.session.*`) to override the realm defaults and reach ~30d offline lifespan, in line with kiosk requirements.

If a later Keycloak release rejects any of these, the fix is local to this repo: re-export, replace the JSON, ship a PR.
