# O2 Migration Plan (Movistar → O2 Cloud Spain)

> Status: **planned, not implemented**. This document records the research, decisions and
> step-by-step plan for turning this fork of
> [uniextra/MovistarWebDAVProxy](https://github.com/uniextra/MovistarWebDAVProxy) into an
> O2 Cloud (Spain) WebDAV proxy. Base commit: `cd7609a` (v2.0.1).

## 1. Goal

- Replace every Movistar Cloud URL/host with the O2 Cloud equivalent.
- O2-only (no dual-provider support).
- Respect O2's **10 TB** storage limit (Movistar is unlimited).
- Full rebrand, keeping backward compatibility for existing token files.
- Reference project for O2: [garanda21/o2cloud_gateway_webdav](https://github.com/garanda21/o2cloud_gateway_webdav).

## 2. Decisions taken

| Topic | Decision |
|---|---|
| Scope | **O2-only**. Drop Movistar URLs entirely. |
| OAuth source | Inspect O2 live and use best-guess values, every one overridable via env. |
| Login flow | **SAPI server-side PKCE broker (`platform=web`) + env fallback** to the legacy direct PKCE flow (only active if `O2_OAUTH_CLIENT_SECRET` is set). |
| 10 TB limit | Report quota **and** reject uploads that do not fit (HTTP 507). |
| Rename depth | **Full rename, with token compatibility** (read legacy `MOVISTAR_*` keys). |
| Docker image | Placeholder `<your-dockerhub-user>/o2-webdav-proxy`. |
| Version | Bump to **v3.0.0** (breaking: new provider + new login flow). |
| UI language | Keep Spanish. |

## 3. Research findings

### 3.1 Reference project (garanda21/o2cloud_gateway_webdav)

- Python app; logs in by driving **headless Chromium/noVNC** and sniffing cookies + OAuth bundle.
  It therefore contains **no** native OAuth values (authorize/token URL, client_id/secret,
  redirect URI, desktop User-Agent).
- Useful values (`src/o2gateway/settings.py`):
  - API: `https://cloud.o2online.es/sapi/`
  - Upload: `https://upload.cloud.o2online.es/sapi/`
  - Web login: `https://cloud.o2online.es/` (human login path `/acceso`)
- Same SAPI protocol as Movistar: `login/oauth?action=login&responsetime=true` with
  `Authorization: oauth <base64 bundle>` and in-band token rotation in the response header.
- Quota: `media?action=get-storage-space&softdeleted=true`; fallback total `10 * 1024**4` (10 TiB).
- Note in `o2/login.py`: O2 binds **web** sessions to the browser's `X-deviceid`
  (cookies/validationKey replayed with another device id → 401).

### 3.2 O2 public system information

`GET https://cloud.o2online.es/sapi/system/information?action=get` (unauthenticated) exposes the
OAuth configuration used by the web SPA:

| | Movistar (current fork) | **O2** |
|---|---|---|
| Authorize | `https://apiseg.telefonica.es/openid/connect/cus/col/auth/oauth/v2/authorize` | `https://apiseg.telefonica.es/openid/connect/auth/oauth/v2/o2/cus/authorize` |
| Token / refresh | `https://apiseg.telefonica.es/openid/connect/cus/col/auth/oauth/v2/token` | `https://apiseg.telefonica.es/openid/connect/auth/oauth/v2/o2/cus/token` |
| Portal | `https://micloud.movistar.es` | `https://cloud.o2online.es` |
| SAPI | `https://micloud.movistar.es/sapi` | `https://cloud.o2online.es/sapi` |
| Upload (`sapi.upload.endpoint`) | `https://upload.micloud.movistar.es` | `https://upload.cloud.o2online.es` |
| Download (`downloadurl`) | `https://micloud.movistar.es/d` | `https://cloud.o2online.es/d` |
| OAuth error page | `https://micloud.movistar.es/ui/html/oauth.html` | `https://cloud.o2online.es/ui/html/oauth.html` |
| Web client id | `6ff3ebc4-4980-43e0-9b18-265e6a1ad260` | `7f8afae4-30e7-4591-b4a8-4fc08d545d1d` |
| Web redirect | `https://micloud.movistar.es/sapi/login/oauth` | `https://cloud.o2online.es/sapi/login/oauth` |
| IdP login page | `t3.movistar.es/segu-loginapp` | `t3.o2online.es/acceso/#/accessUserPassO2` (`client_name=O2CLOUD_WEB`) |
| `sapi.upload.max-size-in-mb` | 4096 | 4096 |
| `sapiversion` | 31.0.0.33 | 31.0.0.33 |
| Login fields | phonenumber, useremail | phonenumber, useremail |
| Features | … `transcoding`, `versioning` … | no `transcoding`, no `versioning` |

`versioning` missing on O2 is harmless: the fork already deletes the old file before overwriting.

### 3.3 OAuth probes (2026-10-29)

| Probe | Result |
|---|---|
| O2 authorize + Movistar **Windows** client_id `b99e5095-…` + redirect `cloud.o2online.es/ui/html/clientoauth.html` | **400** (same as a bogus client id) |
| O2 authorize + bogus client id | 400 |
| O2 authorize + O2 web client id + redirect `cloud.o2online.es/ui/html/clientoauth.html` | 400 (redirect not registered) |
| O2 authorize + O2 web client id + redirect `cloud.o2online.es/sapi/login/oauth` | **302** → `t3.o2online.es/acceso/...` |
| `GET cloud.o2online.es/sapi/oauth/pkce/authorize?platform=web&deviceid=web-x` | **302** → O2 authorize URL with server-generated `state`, `nonce`, `code_challenge` (S256), `redirect_uri=https://cloud.o2online.es/sapi/login/oauth`, `acr_values=2`; sets `JSESSIONID` |
| Same broker with `platform=windows` / `platform=android` | 400 |
| `cloud.o2online.es/ui/html/clientoauth.html` | 200 (page exists) |

Conclusions:

- The fork's current approach (direct PKCE + direct token refresh with the Movistar Windows
  `client_id`/`client_secret` from `brand.dll`) **cannot work on O2**. O2's desktop client
  credentials are unknown.
- The SAPI **server-side PKCE broker** keeps the verifier and client secret on O2's side and is
  the most promising path.
- Since SAPI ≥ 31.0.0.1 the web SPA no longer keeps the OAuth bundle in `localStorage`
  (`storeAuthToken` feature is `serverVersion: "<31.0.0.1"`). So "copy the token from devtools"
  is not a reliable option.
- The web SPA uses `X-deviceid: web-<fingerprint>`.

Reproduce:

```bash
curl -s "https://cloud.o2online.es/sapi/system/information?action=get" | python3 -m json.tool | less
curl -s -D - -o /dev/null "https://cloud.o2online.es/sapi/oauth/pkce/authorize?platform=web&deviceid=web-test"
```

## 4. Current Movistar references in the fork

| File | What |
|---|---|
| `internal/auth/oauth.go:20-27` | `OAuthAuthorizeURL`, `OAuthTokenURL`, `OAuthClientID`, `OAuthClientSecret`, `OAuthRedirectURI`, `ClientPlatform` (`windows`), `DesktopUserAgent` (`omh windows client 32.0.7.1`), `SAPIBaseURL` |
| `internal/sapi/client.go:269` | hardcoded upload host `https://upload.micloud.movistar.es` |
| `internal/sapi/client.go:275,282,290` | hardcoded cookie URL / domain `micloud.movistar.es` |
| `internal/sapi/client.go:422-424,1150` | log text `apiseg.telefonica.es`; download prefix `https://micloud.movistar.es` |
| `internal/webui/server.go:151,243,265` | `"Unlimited"`, log/error texts |
| `internal/webui/templates/index.html:5,18,71,115,133,148,161` | titles, login instructions, redirect URL examples |
| `internal/webdav/server.go:131` | "10 TB virtual for Unlimited" quota fallback |
| `internal/webdav/fs.go` | `MovistarFS` type + log texts |
| `cmd/server/main.go:18,22,120` | `AppVersion`, log texts |
| `internal/config/env.go:111-174` | `MOVISTAR_*` keys in persisted token file |
| `go.mod` | module `movistar-webdav-proxy` (imported everywhere) |
| `Dockerfile`, `docker-compose.yml`, `.github/workflows/docker-publish.yml` | binary, service, volumes, image `uniextra/movistar-webdav-proxy` |
| `README.md`, `LICENSE`, `.gitignore` | docs, copyright, `.movistar-cloud-device-id` |

## 5. Implementation plan

### A. Hosts and URLs (`internal/auth/oauth.go`, `internal/sapi/client.go`)

1. Centralise constants in `auth`:
   - `PortalBaseURL = "https://cloud.o2online.es"`
   - `SAPIBaseURL = PortalBaseURL + "/sapi"`
   - `UploadBaseURL = "https://upload.cloud.o2online.es"`
   - `CookieDomain = "cloud.o2online.es"`
   - `PKCEBrokerURL = SAPIBaseURL + "/oauth/pkce/authorize"`
2. OAuth values with O2 defaults, each overridable by env:

   | Env var | Default |
   |---|---|
   | `O2_OAUTH_AUTHORIZE_URL` | `https://apiseg.telefonica.es/openid/connect/auth/oauth/v2/o2/cus/authorize` |
   | `O2_OAUTH_TOKEN_URL` | `https://apiseg.telefonica.es/openid/connect/auth/oauth/v2/o2/cus/token` |
   | `O2_OAUTH_CLIENT_ID` | `7f8afae4-30e7-4591-b4a8-4fc08d545d1d` (web) |
   | `O2_OAUTH_CLIENT_SECRET` | *(empty → direct PKCE fallback disabled)* |
   | `O2_OAUTH_REDIRECT_URI` | `https://cloud.o2online.es/sapi/login/oauth` |
   | `O2_OAUTH_PLATFORM` | `web` |
   | `O2_OAUTH_USER_AGENT` | browser-like UA (TBD) |

3. Replace the hardcoded hosts in `client.go` (upload, cookie domain ×2, download prefix) with the constants.

### B. Login flow

Main flow: **SAPI PKCE broker**.

1. `/start_login`:
   - The proxy generates `X-deviceid = web-<random>` and persists it.
   - It calls `GET {PKCEBrokerURL}?platform=web&deviceid=<id>` with its own cookie jar and does
     **not** follow redirects.
   - It stores the `Location` (IdP URL) and keeps `JSESSIONID` in the jar.
   - The UI shows the button "🌐 Abrir Login de O2" pointing to that URL.
2. The user logs in at `t3.o2online.es` (O2 user/password or phone + SMS) and ends on
   `https://cloud.o2online.es/sapi/login/oauth?code=…&state=…`. The user copies that full URL
   and pastes it into the proxy.
3. `/complete_login`:
   - The proxy replays the callback URL with the same cookie jar (same `JSESSIONID`) and does
     not follow redirects.
   - It captures `validationkey`, `JSESSIONID`, the rotated `Authorization: oauth <bundle>`
     header, and the JSON body (`data.access_token`, `data.validationkey`, `encryption-token`).
   - Then it validates with `media?action=get-storage-space`.
4. Session renewal relies **only on in-band SAPI rotation**:
   - `login/oauth?action=login&responsetime=true` with `oauth <bundle>`, `platform=web`.
   - Persist each rotated bundle atomically (existing behaviour).
   - Skip direct token-endpoint refresh unless `O2_OAUTH_CLIENT_SECRET` is set.
5. `BuildOAuthHeader` uses the configured platform (`web`). The persisted bundle may be stored
   as received (opaque) rather than rebuilt; decide during implementation after seeing the real
   response.

Fallback flow: the existing direct PKCE code (`BuildOAuthLoginURL`, `ExchangeCodeForTokens`,
`RefreshOAuthToken`), now driven by env values. It is enabled only when
`O2_OAUTH_CLIENT_SECRET` is set, e.g. if O2 desktop client credentials are ever extracted from
the O2 Windows client.

⚠️ **Main risk (untested, no O2 account available during planning):**

- The user's browser also requests the callback URL. If O2 binds the verifier to `state` only
  (not to the broker's `JSESSIONID`), the browser consumes the code first and the proxy replay
  fails.
- Things to try if that happens:
  - Ask the user to copy the URL from devtools *before* it completes (unlikely to work).
  - Add an advanced manual box to paste `validationkey` / `JSESSIONID` / bundle +
    `X-deviceid` captured from devtools.
  - Fall back to a Chromium-based capture like the reference project.

### C. 10 TB limit

1. `GetStorageSpace` (`internal/sapi/client.go`): if the API reports `nolimit`, a missing quota
   or an absurd quota (> 1e18), use `TotalBytes = 10 * 1024^4` (10 TiB, same as the reference)
   and `NoLimit = false`. A real quota returned by the API always wins.
   `FreeBytes = max(0, Total - Used)`.
2. `internal/webdav/server.go` RFC 4331 quota injection: use the real/10 TiB numbers and remove
   the "virtual unlimited" branch.
3. `internal/webui/server.go` + `index.html`: drop `"Unlimited"`; always show `used / total`
   and add a usage bar.
4. Upload guard in `WebDAVFileWriter.Close` (`internal/webdav/fs.go`), where the size is known:
   - If `size > FreeBytes` (60 s cache), abort before uploading.
   - Return an error mapped to **HTTP 507 Insufficient Storage**. This may need a custom error
     check in the `webdav.Handler` wrapper in `server.go`, because `x/net/webdav` maps unknown
     errors to 500.
   - Log `[WebDAV] ❌ Espacio insuficiente …`.
   - Invalidate the storage cache after each successful upload/delete (already done for deletes;
     verify for uploads).
5. Optionally map O2 quota error codes from the API (e.g. `FOL-1035`, "maximum folder count") to
   507 as well.

### D. Rename (full, token-compatible)

- `go.mod`: module `movistar-webdav-proxy` → `o2-webdav-proxy`; update all imports.
- `MovistarFS` → `O2FS`, `NewMovistarFS` → `NewO2FS`.
- `internal/config/env.go`: write `O2_*` keys (`O2_PHONE`, `O2_JSESSIONID`, `O2_VALIDATIONKEY`,
  `O2_DEVICE_ID`, `O2_DEVICE_NAME`, `O2_REFRESH_TOKEN`, `O2_ACCESS_TOKEN`,
  `O2_LAST_REFRESH_DATE`, `O2_EXPIRES_IN`, plus the bundle if stored opaque). When reading, fall
  back to the legacy `MOVISTAR_*` keys.
- Dockerfile: binary `/o2-proxy`.
- docker-compose: service and container `o2-webdav-proxy`, volumes `o2_webdav_tokens` /
  `o2_webdav_data`, image `<your-dockerhub-user>/o2-webdav-proxy:latest`.
- Workflow: `images: <your-dockerhub-user>/o2-webdav-proxy`.
- README:
  - O2 title and badges.
  - New login steps (broker flow, `t3.o2online.es`, paste the `cloud.o2online.es/sapi/login/oauth?code=…` URL).
  - 10 TB limit note and 507 behaviour.
  - Env var table (section A).
  - Credit to upstream and to the O2 reference project.
  - Remove the Movistar-only claims (unlimited storage, Windows client emulation).
- LICENSE: keep the original copyright line and add the fork's.
- `.gitignore`: `.movistar-cloud-device-id` → `.o2-cloud-device-id` (keep the old entry too).
- UI (`index.html`) and log texts: Movistar → O2, Telefónica login → O2 login; keep Spanish.
- `cmd/server/main.go`: `AppVersion = "v3.0.0"`.

### E. Verification

1. `go build ./...` and `go vet ./...`.
2. `docker build .`
3. `grep -rniE "movistar|micloud|cus/col" --exclude-dir=.git .` should only return intentional
   hits (legacy env fallback, credits, this document).
4. Manual test with a real O2 account:
   - Login via the broker.
   - Storage info shows used / 10 TB.
   - PROPFIND quota props.
   - Upload, download (range), rename, delete.
   - Token rotation survives restart.
   - Keep-alive over more than 4 h.
   - Oversized upload → 507.

## 6. Open questions / to verify live

- Does replaying the broker callback from the proxy work (see risk in B)?
- What exactly does `sapi/login/oauth?code=…` return: JSON, cookies, `Authorization` header,
  or a redirect to the portal?
- Does O2 reject an `X-deviceid` that differs from the one sent to the broker?
- Does O2 report a real `quota` in `get-storage-space`, or `nolimit`?
- Which User-Agent does SAPI accept for `platform=web`?
- Can O2 desktop client credentials be obtained (O2 Cloud Windows client `brand.dll`) to enable
  the direct PKCE fallback?
