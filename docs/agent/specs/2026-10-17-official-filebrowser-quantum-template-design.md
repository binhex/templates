# Design: `official-filebrowser-quantum.xml` unRAID template

Status: approved (design), awaiting implementation
Date: 2026-10-17
Repo: `/data/templates`, branch `main`
File under design: `unraid/binhex/templates/official-filebrowser-quantum.xml`

## Problem

The repo carries 11 `official-*` unRAID templates but none for FileBrowser Quantum
(`gtstef/filebrowser` on Docker Hub, upstream `gtsteffaniak/filebrowser`). A template is wanted that
matches the existing `official-*` house format and exposes the image's environment-variable surface.

The repo already contains `filebrowser.xml`, but that template targets a completely different product:
binhex's own `ghcr.io/binhex/arch-filebrowser` image with the container name `binhex-filebrowser` and
WebUI port `8125`. The new template therefore needs a name that cannot be confused with it.

The request was "expose all env vars for this image", the same phrasing used for the Filestash template.
That phrase again needs a precise meaning: the upstream documentation states that the preferred
configuration method is `config.yaml`, and that environment variables exist for **secrets, config
override, and developer purposes**. Completeness therefore means "every variable the shipped binary
reads that a user can meaningfully set", derived from source, with the remaining names recorded as
documented omissions.

## Decisions

| Topic | Decision |
| --- | --- |
| Image | `gtstef/filebrowser:stable` on Docker Hub. GHCR is rejected as the primary source: it carries no `stable` tag and no `2.x` tag at all (verified, see below). |
| Env scope | The 12 user-meaningful variables. Five names are deliberate omissions (four developer/test knobs plus a v1-only variable removed in v2.0); each is recorded with a reason. |
| Network | `bridge`, one port mapping host `8080` -> container `80`, `WebUI = http://[IP]:[PORT:80]`. |
| Volumes | Required `/home/filebrowser/data -> /mnt/user/appdata/filebrowser-quantum`; optional source `/srv -> /mnt/user`. |
| Icon | New `unraid/binhex/images/official-filebrowser-quantum-icon.png`, copied from upstream `frontend/public/img/icons/favicon.png` (already 256x256, the house size). |
| Credentials | Blank, `Mask="true"`, not required. Blank `FILEBROWSER_ADMIN_PASSWORD` preserves the upstream random-password first-login flow. |
| Category | `Productivity: Tools:`, matching the sibling `filebrowser.xml`. |
| Container port | Fixed at `80`; not exposed as a variable, because changing it also requires a custom healthcheck. |

## Source-verified environment surface

Derived from an upstream checkout (`git clone --depth 1 --filter=blob:none --sparse`, then
`git sparse-checkout set _docker backend frontend/public`) at revision
`91bc608f0ec6a3246c29e5fc87acdd46a5fadf44` ("Beta/v2.1.2 (#3142)", 2026-10-13), by grepping
`os.Getenv`, `LookupEnv` and `Getenv` under `backend/`. Cross-checked against the published
environment-variable reference at `https://filebrowserquantum.com/en/docs/reference/environment-variables/`.

The binary reads 16 `FILEBROWSER_*` names in four tiers:

| Tier | Names |
| --- | --- |
| User-facing secrets and auth | `FILEBROWSER_ADMIN_PASSWORD`, `FILEBROWSER_JWT_TOKEN_SECRET`, `FILEBROWSER_TOTP_SECRET`, `FILEBROWSER_RECAPTCHA_SECRET`, `FILEBROWSER_LDAP_USER_PASSWORD`, `FILEBROWSER_OIDC_CLIENT_ID`, `FILEBROWSER_OIDC_CLIENT_SECRET`, `FILEBROWSER_ONLYOFFICE_SECRET` |
| Paths and behaviour | `FILEBROWSER_CONFIG`, `FILEBROWSER_DATABASE_PATH`, `FILEBROWSER_FFMPEG_PATH`, `FILEBROWSER_DISABLE_AUTOMATIC_BACKUP` |
| Developer and test | `FILEBROWSER_DEVMODE`, `FILEBROWSER_SQL_WAL`, `FILEBROWSER_GENERATE_CONFIG`, `FILEBROWSER_PLAYWRIGHT_TEST` |
| Removed | `FILEBROWSER_DATABASE` (v1.x only; removed in v2.0.0, superseded by `FILEBROWSER_DATABASE_PATH`) |

Two further names appear in the grep but are not application settings: `KUBERNETES_SERVICE_HOST`
(in-cluster analytics detection) and `GOFFMPEG_FFMPEG_PATH` / `GOFFMPEG_SAMPLE_MP4` (Go test hooks).
They are filtered, not omissions.

### Registry verification

Queried through the registry APIs on 2026-10-17:

| Tag | Docker Hub `gtstef/filebrowser` | GHCR `ghcr.io/gtsteffaniak/filebrowser` |
| --- | --- | --- |
| `latest` | present | present |
| `stable` | present | **absent** |
| `stable-slim` | present | **absent** |
| `beta`, `beta-slim` | present | present |
| any `2.x` tag | present (`2.0-stable`, `2.0.1-stable`, ...) | **absent** (newest tags are `0.7.x-dev` / `0.7.x-beta`) |

Docker Hub carries 307 tags, GHCR 100. The upstream documentation's own compose example uses
`ghcr.io/gtsteffaniak/filebrowser:stable`, which does not exist; the template pins the Docker Hub tag
instead. The newest upstream stable release at design time was `v2.0.1-stable` (2026-10-13); `stable`
floats within the 2.0.x line.

### Image facts

Confirmed from `_docker/Dockerfile`:

- `EXPOSE 80` and `HEALTHCHECK ... CMD curl -f http://localhost:80/health || exit 1`.
- `USER filebrowser` (created as UID 1000, GID 1000), `WORKDIR /home/filebrowser`.
- `ENTRYPOINT ["./filebrowser"]` with no arguments.
- Image-set environment: `FILEBROWSER_FFMPEG_PATH=/usr/local/bin/`,
  `FILEBROWSER_DATABASE_PATH=/home/filebrowser/data/database.sqlite`,
  `FILEBROWSER_CONFIG=/home/filebrowser/data/config.yaml`.
- The shipped `backend/config.yaml` is copied to `/home/filebrowser/data/config.yaml` and to
  `/home/filebrowser/config.yaml` (the second location for compatibility). It sets `http.port: 80`,
  `http.baseURL: "/"`, `server.sources[].path: /srv`, and password auth enabled with `signup: false`.

HTTPS is supported through `http.tlsCert` / `http.tlsKey` on the same listener, and `http.socket` is
mutually exclusive with those (enforced by `ValidateHttpConfig`). Neither has an environment variable.

## Template structure

### Identity block

| Element | Value |
| --- | --- |
| `Name` | `binhex-official-filebrowser-quantum` |
| `Category` | `Productivity: Tools:` |
| `Repository` | `gtstef/filebrowser:stable` |
| `Registry` | `https://hub.docker.com/r/gtstef/filebrowser` |
| `Network` | `bridge` |
| `Privileged` / `PrivilegedReq` | `false` / `false` |
| `Shell` | `bash` |
| `ExtraParams` | empty |
| `GitHub` | `https://github.com/gtsteffaniak/filebrowser` |
| `Support` | `https://github.com/gtsteffaniak/filebrowser/issues` |
| `Project` | `https://github.com/gtsteffaniak/filebrowser` |
| `WebUI` | `http://[IP]:[PORT:80]` |
| `TemplateURL` | `https://raw.githubusercontent.com/binhex/templates/main/unraid/binhex/templates/official-filebrowser-quantum.xml` |
| `Icon` | `https://raw.githubusercontent.com/binhex/templates/main/unraid/binhex/images/official-filebrowser-quantum-icon.png` |
| `Date` / `Changes` | `1970-01-01` / empty |

`Overview` is a single paragraph describing FileBrowser Quantum as an actively maintained fork of the
original filebrowser with full-text search, previews and thumbnails, sharing links, multi-source
support, and optional OIDC, LDAP and proxy authentication. It must state that the primary configuration
is `config.yaml` inside the data volume while environment variables cover secrets and overrides, so a
reader is not misled into expecting every setting as a template field. Ampersands must be escaped as
`&amp;`.

The standard `DonateText` / `DonateLink` / `DonateImg` block is copied verbatim from another
`official-*` template. Indentation is four spaces throughout, matching `official-filestash.xml`;
the tab-indented `<Beta>` seen in some existing files is not replicated. `Beta` is `false`.

### Ports and volumes

| Name | Target | Default | Type | Display | Required |
| --- | --- | --- | --- | --- | --- |
| `Port: Web Interface` | `80` | `8080` | `Port` | `always` | `true` |
| `Path: /home/filebrowser/data` | `/home/filebrowser/data` | `/mnt/user/appdata/filebrowser-quantum` | `Path` | `always` | `true` |
| `Path: /srv` | `/srv` | `/mnt/user` | `Path` | `always` | `false` |

The container port stays `80` because that is what the image exposes, healthchecks and ships as
`http.port`; a different container port would also need a custom healthcheck. The host default `8080`
avoids unRAID's own port 80 and the `8125` already used by `binhex-filebrowser`. The port description
states that changing the container port requires matching `http.port` and healthcheck edits, so the
host-side field should be used instead.

`/home/filebrowser/data` is the only path that must persist: it holds `config.yaml`, `database.sqlite`
and the cache. Its description notes that the image runs as UID:GID `1000:1000`, that a different user
can be set through `--user` in extra parameters, and that the host directory then needs matching
ownership (the upstream documentation suggests `1001:1001` for unRAID installs).

`/srv` matches the shipped config's default source path, so files mounted there appear without editing
`config.yaml`. Its description tells the user to narrow the mapping to one share rather than all of
`/mnt/user`.

### Environment variables

`Display` is `always` for the single first-run essential and `advanced` for the rest. Credentials are
blank, masked and never required, because the image starts correctly with all of them unset. For
dropdown configs, the first option is the effective default.

| Target | Default | Display | Mask | Notes |
| --- | --- | --- | --- | --- |
| `FILEBROWSER_ADMIN_PASSWORD` | (blank) | `always` | true | Blank keeps the upstream behaviour: a random admin password is generated, logged once, and a reset is forced at first login |
| `FILEBROWSER_JWT_TOKEN_SECRET` | (blank) | `advanced` | true | Signs JWTs; changing it after first start invalidates existing sessions |
| `FILEBROWSER_TOTP_SECRET` | (blank) | `advanced` | true | TOTP encryption secret |
| `FILEBROWSER_RECAPTCHA_SECRET` | (blank) | `advanced` | true | reCAPTCHA secret for password auth |
| `FILEBROWSER_LDAP_USER_PASSWORD` | (blank) | `advanced` | true | LDAP service-account bind password |
| `FILEBROWSER_OIDC_CLIENT_ID` | (blank) | `advanced` | false | OIDC client id |
| `FILEBROWSER_OIDC_CLIENT_SECRET` | (blank) | `advanced` | true | OIDC client secret |
| `FILEBROWSER_ONLYOFFICE_SECRET` | (blank) | `advanced` | true | OnlyOffice integration JWT secret |
| `FILEBROWSER_CONFIG` | (blank) | `advanced` | false | Blank uses the image default `/home/filebrowser/data/config.yaml` |
| `FILEBROWSER_DATABASE_PATH` | (blank) | `advanced` | false | Blank uses the image default `/home/filebrowser/data/database.sqlite`; a path set in `config.yaml` takes precedence |
| `FILEBROWSER_FFMPEG_PATH` | (blank) | `advanced` | false | Blank uses the image default `/usr/local/bin/`; only the full image bundles ffmpeg |
| `FILEBROWSER_DISABLE_AUTOMATIC_BACKUP` | `false\|true` | `advanced` | false | Backups are on by default; `true` disables the automatic pre-upgrade backup |

Every `Config` uses `Name="<Kind>: <LABEL>"`, matching the existing templates. All twelve variables are
`Required="false"`; only the port and the data path are `Required="true"`. In the table above, `\|`
inside a `Default` cell is a literal pipe and denotes a dropdown whose options are that list.

## Omitted variables

| Variable | Why it is not exposed |
| --- | --- |
| `FILEBROWSER_DATABASE` | v1.x only. Removed in v2.0.0 and superseded by `FILEBROWSER_DATABASE_PATH`. |
| `FILEBROWSER_DEVMODE` | Developer switch that changes update-check and logging behaviour. Not a user setting. |
| `FILEBROWSER_SQL_WAL` | Toggles the SQLite WAL journal mode. An operational tuning detail of the database, not a user-facing option. |
| `FILEBROWSER_GENERATE_CONFIG` | Install-time helper that makes the binary emit a config file and exit. Meaningless under the image's entrypoint. |
| `FILEBROWSER_PLAYWRIGHT_TEST` | Test-harness switch used by the upstream Playwright suites. |

## Configuration outside the environment

The template cannot express most of the image's configuration, and the spec records this explicitly
rather than implying completeness:

- Everything except the twelve variables above is `config.yaml`-only: `http.port`, `http.baseURL`,
  `http.tlsCert` / `http.tlsKey`, `server.sources`, `server.cacheDir`, authentication methods,
  user defaults and integrations.
- `config.yaml` may reference environment variables in `${VAR}` / `$VAR` form; the loader expands
  string scalars after YAML decoding and coerces a whole-scalar reference to bool or number so typed
  fields keep working. This is the upstream-supported way to inject arbitrary config values, including
  through unRAID extra parameters.
- The documentation recommends omitting a root `/` source and excluding `/var`, and placing `cacheDir`
  under the data volume (`/home/filebrowser/data/tmp`) so it persists across restarts. The template's
  `/srv` default source and data-volume path are chosen to work with that guidance.

## Non-goals

- No committed drift-checking script or CI job; the repo has no template tooling today and the
  environment check is run ad hoc during implementation.
- No changes to any existing template, including `filebrowser.xml`, and no change to the pre-existing
  uncommitted `projects/common/.gitignore` modification, which stays untouched and uncommitted.
- No `PUID` / `PGID` variables: the image reads no such variables. Ownership is handled by the
  documented `--user` extra parameter and host-side `chown`.
- No TLS/HTTPS template fields: certificate configuration has no environment variable in this image.
- No OnlyOffice or Collabora companion template; the integration is opt-in through config and secrets.
- No separate template for the `stable-slim` image variant.
- No reverse-proxy template; `http.baseURL` and `trustProxyHeaders` are documented as `config.yaml`
  settings.

## Verification

Implementation is complete only when all of the following pass:

1. The XML parses (Python `xml.etree` or `xmllint --noout`).
2. Exactly 12 variable targets, 1 port and 2 paths are present, with no duplicate `Target` values across
   the file (15 `Config` entries in total).
3. Every pipe-separated `Default` begins with the documented effective default
   (`FILEBROWSER_DISABLE_AUTOMATIC_BACKUP` = `false|true`).
4. `Required` and `Mask` are limited to `true|false`, `Display` to `always|advanced`;
   `Required="true"` appears only on the port and the data path, and `Mask="true"` only on the seven
   credential variables (`ADMIN_PASSWORD`, `JWT_TOKEN_SECRET`, `TOTP_SECRET`, `RECAPTCHA_SECRET`,
   `LDAP_USER_PASSWORD`, `OIDC_CLIENT_SECRET`, `ONLYOFFICE_SECRET`).
5. The target list matches a grep of the upstream checkout (`backend/pkg/settings/config.go`,
   `backend/pkg/settings/structs.go`, `backend/pkg/settings/config_env.go`, `backend/cmd/cli.go`,
   `backend/cmd/kong.go`, `backend/cmd/user.go`), with the five omissions of this document accounted
   for and `KUBERNETES_SERVICE_HOST` / `GOFFMPEG_*` filtered as non-settings.
6. `TemplateURL` and `Icon` filenames match the committed paths, and
   `unraid/binhex/images/official-filebrowser-quantum-icon.png` exists, is a valid PNG, is 256x256, and
   is byte-identical to upstream `frontend/public/img/icons/favicon.png`.
7. The identity block is element-for-element identical in order to `official-qbit-guard.xml`
   (element order, Donate block, `Date` / `Changes`).
8. `markdownlint --fix` reports no issues for this design document.

## Deliverables

- `unraid/binhex/templates/official-filebrowser-quantum.xml`
- `unraid/binhex/images/official-filebrowser-quantum-icon.png` - the upstream favicon, which is already
  the 256x256 size the existing `official-*-icon.png` files use, so no conversion is required
- This design document

## Risks

- The `stable` tag floats within the 2.0.x line, so upstream patch releases reach users without a
  template change. Mitigated by the field descriptions documenting current behaviour and defaults.
- With `FILEBROWSER_ADMIN_PASSWORD` blank, a random password is generated and logged once, and the first
  login forces a reset. The field description states this, because a user who does not read the
  container log cannot log in.
- The sibling `filebrowser.xml` template is a different image (`ghcr.io/binhex/arch-filebrowser`). The
  `official-filebrowser-quantum` name and `Repository` value disambiguate them, and neither file is
  modified here.
- The image runs as UID:GID `1000:1000` while unRAID hosts commonly use `99:100`. Users must set
  `--user` and fix host ownership; the data-path description says so.
- The shipped source path is `/srv`, so the optional source mapping only works when files are mounted
  there. The template maps `/srv` by default and the description explains it.
- The host port differs from the container port. Changing `http.port` in `config.yaml` also requires the
  container port and the image healthcheck to be overridden, which the port description states.
