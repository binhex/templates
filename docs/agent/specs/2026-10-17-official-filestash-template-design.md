# Design: `official-filestash.xml` unRAID template

Status: approved (design), awaiting implementation
Date: 2026-10-17
Repo: `/data/templates`, branch `main`
File under design: `unraid/binhex/templates/official-filestash.xml`

## Problem

The repo carries 10 `official-*` unRAID templates but none for Filestash
(`machines/filestash` on Docker Hub, upstream `mickael-kerjean/filestash`). A template is wanted that
matches the existing `official-*` house format and exposes the image's full environment-variable surface.

The request was "expose all env vars for this image". That phrase needs a precise meaning, because
Filestash reads only a small set of variables from the environment: storage backends, authentication
providers, themes and plugins are configured in `state/config/config.json` and edited through the
`/admin` UI. The upstream `docker/docker-compose.yml` sets just four variables. Completeness therefore
means "every variable the shipped binary reads", which was derived from the upstream source rather than
from documentation.

## Decisions

| Topic | Decision |
| --- | --- |
| Env scope | Expose only variables the binary actually reads (source-verified). The compose file's `CANARY` is excluded, as the current source never references it. |
| Network | `bridge`, one port mapping `8334 -> 8334`, `WebUI = http://[IP]:[PORT:8334]`. |
| Volumes | Required `/app/data -> /mnt/user/appdata/filestash`; optional advanced `/media -> /mnt/user`. |
| Office/WOPI | Single template. `OFFICE_*` and `ONLYOFFICE_URL` exposed but blank; no bundled Collabora container. |
| Icon | New `unraid/binhex/images/official-filestash-icon.png`, referenced by the standard convention URL. |
| Credentials | Blank, `Mask="true"`, not required. Blank `ADMIN_PASSWORD` preserves the upstream first-run wizard. |
| Port/path vars | Omit `FILESTASH_PORT` and `FILESTASH_PATH` (see Omitted variables). |
| Category | `Cloud: Tools:`. |

## Source-verified environment surface

Derived from an upstream checkout (`git clone --depth 1 --filter=blob:none --sparse`,
`git sparse-checkout set server docker`) by grepping `os.Getenv` and `defaultValue(..., "ENV")` under
`server/`. Twenty names are read; seventeen are exposed.

Image facts confirmed from `docker/Dockerfile`: `CMD ["/app/filestash"]`, `EXPOSE 8334`,
`USER filestash`, and all state rooted at `/app/data` (`state/config`, `state/certs`, `state/db`,
`state/search`, `state/plugins`, `state/log`, plus `cache`). Docker Hub publishes no semver tags -
`latest` is the only sensible pin.

## Template structure

### Identity block

| Element | Value |
| --- | --- |
| `Name` | `binhex-official-filestash` |
| `Category` | `Cloud: Tools:` |
| `Repository` | `machines/filestash:latest` |
| `Registry` | `https://hub.docker.com/r/machines/filestash/` |
| `Network` | `bridge` |
| `Privileged` / `PrivilegedReq` | `false` / `false` |
| `Shell` | `bash` |
| `ExtraParams` | empty |
| `GitHub` | `https://github.com/mickael-kerjean/filestash` |
| `Support` | `https://github.com/mickael-kerjean/filestash/issues` |
| `Project` | `https://github.com/mickael-kerjean/filestash` |
| `WebUI` | `http://[IP]:[PORT:8334]` |
| `TemplateURL` | `https://raw.githubusercontent.com/binhex/templates/main/unraid/binhex/templates/official-filestash.xml` |
| `Icon` | `https://raw.githubusercontent.com/binhex/templates/main/unraid/binhex/images/official-filestash-icon.png` |
| `Date` / `Changes` | `1970-01-01` / empty |

`Overview` is a single paragraph describing Filestash as a storage-agnostic web file manager supporting
SFTP, S3, SMB, WebDAV, FTP, NFS and IPFS, with office/web preview plugins and OIDC/SAML/LDAP auth.
Ampersands in it must be escaped as `&amp;`.

The standard `DonateText` / `DonateLink` / `DonateImg` block is copied verbatim from another
`official-*` template. Indentation is four spaces throughout; the tab-indented `<Beta>` seen in some
existing files is not replicated. `Beta` is `false`.

### Ports and volumes

| Name | Target | Default | Type | Display | Required |
| --- | --- | --- | --- | --- | --- |
| `Port: WebUI` | `8334` | `8334` | `Port` | `always` | `true` |
| `Path: Appdata` | `/app/data` | `/mnt/user/appdata/filestash` | `Path` | `always` | `true` |
| `Path: Media` | `/media` | `/mnt/user` | `Path` | `advanced` | `false` |

`/app/data` is the only path that must persist; it holds config, certificates, database, search index,
plugins, logs and cache. `/media` exists so the default "local storage" preset can see array data; its
description tells the user to narrow it to a specific share.

### Environment variables

`Display` is `always` for the instance essentials and `advanced` for the integration tail. For dropdown
configs (`Type="Variable"` with a pipe-separated `Default`), the first option is the effective default,
so ordering encodes the upstream default.

| Target | Default | Display | Mask | Notes |
| --- | --- | --- | --- | --- |
| `APPLICATION_URL` | (blank) | `always` | false | Public URL, e.g. `https://files.example.com`; required behind a reverse proxy |
| `ADMIN_PASSWORD` | (blank) | `always` | true | Blank keeps the upstream first-run wizard |
| `FILESTASH_BASE` | (blank) | `advanced` | false | URL base path for subfolder hosting, e.g. `/filestash` |
| `LOG_LEVEL` | `INFO\|DEBUG\|WARNING\|ERROR` | `always` | false | Upstream default `INFO` |
| `DEBUG` | `false\|true` | `advanced` | false | Verbose mode |
| `CONFIG_SECRET` | (blank) | `advanced` | true | Encryption key for `config.json`; blank derives one automatically |
| `CONFIG_ENCRYPT` | `true\|false` | `advanced` | false | Encrypt config at rest |
| `OFFICE_URL` | (blank) | `advanced` | false | WOPI server URL, e.g. `http://collabora:9980` |
| `OFFICE_FILESTASH_URL` | (blank) | `advanced` | false | URL the WOPI server uses to reach Filestash |
| `OFFICE_REWRITE_URL` | (blank) | `advanced` | false | URL rewrite used by the WOPI host |
| `ONLYOFFICE_URL` | (blank) | `advanced` | false | ONLYOFFICE document server URL |
| `SYNCTHING_URL` | (blank) | `advanced` | false | Syncthing widget endpoint |
| `GDRIVE_CLIENT_ID` | (blank) | `advanced` | false | Google Drive OAuth app client id |
| `GDRIVE_CLIENT_SECRET` | (blank) | `advanced` | true | Google Drive OAuth app client secret |
| `DROPBOX_CLIENT_ID` | (blank) | `advanced` | false | Dropbox OAuth app client id |
| `LOCAL_BACKEND_SECRET` | (blank) | `advanced` | true | Secret for the local-storage backend |
| `PLG_AUTHENTICATE_LOCAL_ENABLED` | `false\|true` | `advanced` | false | Enables the local authentication plugin |

Every `Config` uses `Name="<Kind>: <LABEL>"` (for example `Variable: APPLICATION_URL`) to match the
existing templates. Only the port and appdata path are `Required="true"`; every variable is
`Required="false"`, because the application starts correctly with all of them blank.

In the table above, `\|` inside a `Default` cell is a literal pipe and denotes a dropdown whose options
are that list. Nothing in this document contains an unescaped table pipe.

## Omitted variables

| Variable | Why it is not exposed |
| --- | --- |
| `FILESTASH_PORT` | Changing it desynchronises the unRAID port mapping and the `WebUI` port link. The port is fixed at 8334 in the port config instead. |
| `FILESTASH_PATH` | Relocates all state outside the mapped `/app/data` volume, so data silently fails to persist. |
| `LICENSE` | Read-only display constant in the admin UI, not a user setting. |
| `CANARY` | Present in the upstream compose file but unreferenced by the current source. Excluded as a no-op. |

## Non-goals

- No `official-collabora.xml` companion template (office editing is opt-in through `OFFICE_*`).
- No bundled SFTP/S3 gateway port configs; those are plugin features enabled in `/admin`.
- No committed drift-checking script or CI job - the repo has no template tooling today.
- No changes to any existing template, including the pre-existing uncommitted
  `projects/common/.gitignore` modification, which stays untouched and uncommitted.

## Verification

Implementation is complete only when all of the following pass:

1. The XML parses (Python `xml.etree` or `xmllint --noout`).
2. All 17 variable targets are present exactly once, with no duplicate `Target` values across the file.
3. Every pipe-separated `Default` begins with the documented upstream default.
4. `Required`, `Mask` and `Display` values are limited to `true|false` and `always|advanced`.
5. The target list matches a grep of the upstream source checkout (`server/pkg/config/config.go`,
   `server/pkg/config/config_state.go`, `server/pkg/env/constants.go`, `server/plugin/plg_editor_wopi`,
   `server/plugin/plg_editor_onlyoffice`, `server/plugin/plg_handler_syncthing`,
   `server/plugin/plg_backend_gdrive`, `server/plugin/plg_backend_dropbox`,
   `server/plugin/plg_backend_local`, `server/plugin/plg_authenticate_local`, `server/plugin/plg_license`),
   with `FILESTASH_PORT`, `FILESTASH_PATH` and `LICENSE` accounted for as the three deliberate omissions.
   `CANARY` is not one of them: the current source never reads it.
6. `TemplateURL` and `Icon` filenames match the committed paths, and `unraid/binhex/images/official-filestash-icon.png`
   exists and is a valid PNG.
7. The template is byte-identical in structure to the identity block conventions of
   `official-qbit-guard.xml` (element order, Donate block, `Date`/`Changes`).
8. `markdownlint --fix` reports no issues for this design document.

## Deliverables

- `unraid/binhex/templates/official-filestash.xml`
- `unraid/binhex/images/official-filestash-icon.png` - the upstream project logo converted to PNG,
  matching the naming and rough dimensions of the existing `official-*-icon.png` files
- This design document

## Risks

- Upstream has no versioned image tags, so the template tracks `latest`; a future upstream change can
  alter behaviour without the template changing. Mitigated by the description fields documenting defaults.
- `OFFICE_*` defaults are blank, so office editing appears broken until the user supplies a WOPI server.
  The field descriptions state that explicitly rather than shipping a default that cannot work.
- The exposed `CONFIG_SECRET` interacts with existing state: setting it after first run can make an
  already-encrypted `config.json` unreadable. The field description warns that it must be set before
  first start.
