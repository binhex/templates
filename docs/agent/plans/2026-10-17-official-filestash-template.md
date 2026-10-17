# official-filestash unRAID Template Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use sub-agents (recommended) to implement this plan task-by-task.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship `unraid/binhex/templates/official-filestash.xml` (plus its icon) exposing all 18 environment
variables the `machines/filestash` binary actually reads, and correct the approved design spec to match.

**Architecture:** A single declarative unRAID template file following the existing `official-*` house format
(`<Container version="2">`, `Config` entries of type `Variable`/`Path`/`Port`). No application code, no build
step, no runtime component. Correctness is verified by an ad-hoc Python validator kept in `/tmp` (never
committed) plus the spec's own 8-point checklist.

**Tech Stack:** XML (unRAID Docker template schema), Python 3 stdlib for validation, `xmllint` and
`markdownlint` as available CLI gates.

---

## Scope check

Single subsystem. The spec covers exactly one artifact pair (one template + one icon) plus a spec correction.
No split into separate plans is warranted.

## Repo facts (verified during planning)

| Fact | Value |
| --- | --- |
| Repo root | `/data/templates` (branch `main`, HEAD `99a6ef9`) |
| Working CWD | `/data/templates/unraid/binhex/templates` |
| Spec | `docs/agent/specs/2026-10-17-official-filestash-template-design.md` (approved, 173 lines) |
| Upstream revision inspected | `6feaa2127a495880641ed2b70aa7abeae9e8367f` (`mickael-kerjean/filestash`, 2026-10-17) |
| Reference template | `unraid/binhex/templates/official-qbit-guard.xml` (identity-block conventions) |
| Unrelated dirty file | `projects/common/.gitignore` — **must stay untouched and never staged** |
| Icon tooling | none (no ImageMagick, no `ffmpeg`, no PIL, no `sharp`) → icon is copied verbatim, not resized |
| Available gates | `xmllint` (`/usr/sbin/xmllint`), `markdownlint` (no `.markdownlint*` config exists) |

## Decisions taken during planning

1. **`FEATURE_TRANSCODING_VIDEO_BLACKLIST` is exposed as an 18th variable** (user-approved). The approved spec
   listed 17 and missed this one because its file enumeration omitted
   `server/plugin/plg_video_transcoder/config.go`, where line 39 reads
   `f.Default = os.Getenv("FEATURE_TRANSCODING_VIDEO_BLACKLIST")`. That plugin is compiled into the default
   binary (`server/plugin/index.go:44`, blank import), so the shipped image reads it.
2. **`HOME` and `PATH` are filtered as system variables**, not listed as template omissions; the spec gains a
   sentence saying so.
3. **Icon is `public/assets/logo/android-chrome-512x512.png` copied verbatim** (512x512 palette PNG, 18 564
   bytes). The repo already ships a 512x512 icon (`resilio-sync-icon.png`), so this is within the existing
   dimension range and needs no image tooling.

## House-convention decisions (intentional deviations from the spec's cosmetic naming)

The user's instruction was "follow the same format as other `official-*` unraid templates". Where the spec's
config *labels* differ from the house pattern, the house pattern wins. Reviewers should treat these as
deliberate, not as defects:

| Element | Spec table says | Plan uses | Why |
| --- | --- | --- | --- |
| Port label | `Port: WebUI` | `Port: Web Interface` | `official-metube.xml` and `official-pihole.xml` (the other web-GUI templates) use this exact label |
| Path labels | `Path: Appdata`, `Path: Media` | `Path: /app/data`, `Path: /media` | All 11 existing `Path` configs across the 10 `official-*` files name the container target, never a descriptive label |
| `WebUI` value | `http://[IP]:[PORT:8334]` | `http://[IP]:[PORT:8334]` | follows the spec and the `8000`/`8006` house examples (no trailing slash) |
| `<Beta>` indentation | four spaces | four spaces | the tab-indented `<Beta>` in some existing files is an inconsistency, not a convention |

## File structure

| Action | Path | Responsibility |
| --- | --- | --- |
| Create | `unraid/binhex/templates/official-filestash.xml` | the unRAID template: identity block, 1 port, 2 paths, 18 variables |
| Create | `unraid/binhex/images/official-filestash-icon.png` | template icon, referenced by `TemplateURL`/`Icon` |
| Modify | `docs/agent/specs/2026-10-17-official-filestash-template-design.md` | correct var count 17 → 18, add the new row, extend verification checklist |
| Temp only (never committed) | `/tmp/verify-filestash-template.py` | ad-hoc validator, honours the spec non-goal "no committed drift-checking script" |
| Temp only (never committed) | `/tmp/fs-logo/` | staging directory for the fetched upstream icon |

All paths above are relative to `/data/templates` unless absolute.

---

### Task 1: Re-verify the environment-variable surface from upstream source

**Files:**

- Create: none (evidence only)
- Modify: none

- [ ] **Step 1: Fetch an upstream checkout with only the needed subtrees**

```bash
cd /tmp
rm -rf fs-src
git clone --depth 1 --filter=blob:none --sparse https://github.com/mickael-kerjean/filestash fs-src
cd /tmp/fs-src
git sparse-checkout set server docker
git log -1 --format='%H %ad %s'
```

Expected: the clone succeeds and prints the revision hash. Planning observed
`6feaa2127a495880641ed2b70aa7abeae9e8367f`. A different hash is acceptable (upstream moves), but note it in
the commit body because the spec cites the surface as source-verified.

- [ ] **Step 2: Re-derive the env read sites**

```bash
cd /tmp/fs-src
grep -rhoE '(os\.Getenv|LookupEnv|Getenv)\("[A-Za-z_0-9]+"\)' --include='*.go' server/ \
  | sed 's/.*("//;s/")//' | sort -u
echo '--- defaultValue defaults ---'
grep -rhoE 'defaultValue\([^,]+, *"[A-Za-z_0-9]+"' --include='*.go' server/ \
  | sed -E 's/.*"(.*)"/\1/' | sort -u
```

Expected, from the first command (20 names, sorted):

```text
ADMIN_PASSWORD
APPLICATION_URL
CONFIG_SECRET
DEBUG
DROPBOX_CLIENT_ID
FEATURE_TRANSCODING_VIDEO_BLACKLIST
FILESTASH_BASE
FILESTASH_PATH
GDRIVE_CLIENT_ID
GDRIVE_CLIENT_SECRET
HOME
LICENSE
LOCAL_BACKEND_SECRET
OFFICE_FILESTASH_URL
OFFICE_REWRITE_URL
OFFICE_URL
ONLYOFFICE_URL
PATH
PLG_AUTHENTICATE_LOCAL_ENABLED
SYNCTHING_URL
```

Expected, from the second command (4 names, sorted):

```text
CONFIG_ENCRYPT
CONFIG_SECRET
FILESTASH_PORT
LOG_LEVEL
```

Union: 23 unique names (`CONFIG_SECRET` appears in both).

- [ ] **Step 3: Assert the partition adds up**

```bash
cd /tmp/fs-src
grep -n 'plg_video_transcoder' server/plugin/index.go
grep -n 'FEATURE_TRANSCODING_VIDEO_BLACKLIST' server/plugin/plg_video_transcoder/config.go
grep -n '"CANARY"' docker/docker-compose.yml || echo "CANARY absent from source"
```

Expected: `server/plugin/index.go:44` blank-imports `plg_video_transcoder`;
`server/plugin/plg_video_transcoder/config.go:39` reads `os.Getenv("FEATURE_TRANSCODING_VIDEO_BLACKLIST")`;
the `CANARY` grep prints `CANARY absent from source`.

Partition to confirm before continuing: 23 read = 18 exposed + 3 deliberate omissions (`FILESTASH_PORT`,
`FILESTASH_PATH`, `LICENSE`) + 2 system variables (`HOME`, `PATH`). `CANARY` is not read at all.

- [ ] **Step 4: Record the evidence in the task notes**

No file is written to the repo in this task. Paste the three command outputs into the sub-agent report; Task 3's
commit body summarises them. If any name outside the expected 23 appears, STOP and report — the plan's
18-variable set would need re-approval before Task 3.

---

### Task 2: Write the validator and watch it fail

**Files:**

- Create: `/tmp/verify-filestash-template.py` (temp only, never committed)
- Test: itself

- [ ] **Step 1: Write the validator**

```python
#!/usr/bin/env python3
"""Ad-hoc validator for the official-filestash unRAID template. Temp file; never commit."""

import struct
import sys
import xml.etree.ElementTree as ET
from pathlib import Path

ROOT = Path("/data/templates")
TPL = ROOT / "unraid/binhex/templates/official-filestash.xml"
REF = ROOT / "unraid/binhex/templates/official-qbit-guard.xml"
ICON = ROOT / "unraid/binhex/images/official-filestash-icon.png"

EXPECTED_VARIABLES = [
    "APPLICATION_URL",
    "ADMIN_PASSWORD",
    "FILESTASH_BASE",
    "LOG_LEVEL",
    "DEBUG",
    "CONFIG_SECRET",
    "CONFIG_ENCRYPT",
    "OFFICE_URL",
    "OFFICE_FILESTASH_URL",
    "OFFICE_REWRITE_URL",
    "ONLYOFFICE_URL",
    "SYNCTHING_URL",
    "GDRIVE_CLIENT_ID",
    "GDRIVE_CLIENT_SECRET",
    "DROPBOX_CLIENT_ID",
    "LOCAL_BACKEND_SECRET",
    "PLG_AUTHENTICATE_LOCAL_ENABLED",
    "FEATURE_TRANSCODING_VIDEO_BLACKLIST",
]
EXPECTED_PORTS = ["8334"]
EXPECTED_PATHS = ["/app/data", "/media"]
EXPECTED_ALL = set(EXPECTED_VARIABLES) | set(EXPECTED_PORTS) | set(EXPECTED_PATHS)

DROPDOWNS = {
    "LOG_LEVEL": "INFO|DEBUG|WARNING|ERROR",
    "DEBUG": "false|true",
    "CONFIG_ENCRYPT": "true|false",
    "PLG_AUTHENTICATE_LOCAL_ENABLED": "false|true",
}
MASKED = {"ADMIN_PASSWORD", "CONFIG_SECRET", "GDRIVE_CLIENT_SECRET", "LOCAL_BACKEND_SECRET"}

TEMPLATE_URL = (
    "https://raw.githubusercontent.com/binhex/templates/main/unraid/binhex/templates/official-filestash.xml"
)
ICON_URL = "https://raw.githubusercontent.com/binhex/templates/main/unraid/binhex/images/official-filestash-icon.png"

FAILS = []


def check(ok, message):
    print(("PASS  " if ok else "FAIL  ") + message)
    if not ok:
        FAILS.append(message)


def check_icon():
    if not ICON.is_file():
        check(False, f"icon file exists: {ICON}")
        return
    check(True, f"icon file exists: {ICON}")
    head = ICON.read_bytes()[:33]
    is_png = head[:8] == b"\x89PNG\r\n\x1a\n"
    check(is_png, "icon is a PNG (magic bytes)")
    if is_png:
        width, height = struct.unpack(">II", head[16:24])
        check((width, height) == (512, 512), f"icon is 512x512 (got {width}x{height})")


def main():
    skip_icon = "--skip-icon" in sys.argv
    if not TPL.is_file():
        print(f"FAIL  template exists: {TPL}")
        return 1
    check(True, f"template exists: {TPL}")

    raw = TPL.read_text(encoding="utf-8")
    check("\t" not in raw, "file uses spaces only (no tab characters)")

    root = ET.fromstring(raw)
    check(root.tag == "Container", 'root element is <Container>')
    check(root.get("version") == "2", 'Container version="2"')

    ref_root = ET.parse(REF).getroot()
    ref_order = [c.tag for c in ref_root if c.tag != "Config"]
    our_order = [c.tag for c in root if c.tag != "Config"]
    check(our_order == ref_order, f"identity-block element order matches official-qbit-guard.xml ({ref_order})")

    texts = {c.tag: (c.text or "") for c in root if c.tag != "Config"}
    check(texts.get("Name") == "binhex-official-filestash", "Name is binhex-official-filestash")
    check(texts.get("Category") == "Cloud: Tools:", "Category is Cloud: Tools:")
    check(texts.get("Repository") == "machines/filestash:latest", "Repository pins machines/filestash:latest")
    check(texts.get("Network") == "bridge", "Network is bridge")
    check(texts.get("WebUI") == "http://[IP]:[PORT:8334]", "WebUI points at [PORT:8334]")
    check(texts.get("TemplateURL") == TEMPLATE_URL, "TemplateURL matches the committed path")
    check(texts.get("Icon") == ICON_URL, "Icon matches the committed path")
    check(texts.get("Date") == "1970-01-01", "Date is 1970-01-01")
    check(texts.get("Beta") == "false", "Beta is false")

    configs = root.findall("Config")
    targets = [c.get("Target") for c in configs]
    check(len(targets) == len(set(targets)), "no duplicate Target values")
    check(set(targets) == EXPECTED_ALL, f"target set is exactly the expected {len(EXPECTED_ALL)} entries")

    kinds = {}
    for c in configs:
        kind = (c.get("Name") or "").split(":")[0]
        kinds.setdefault(kind, []).append(c)

    var_targets = [c.get("Target") for c in kinds.get("Variable", [])]
    check(sorted(var_targets) == sorted(EXPECTED_VARIABLES), "all 18 variables present exactly once")
    check([c.get("Target") for c in kinds.get("Port", [])] == EXPECTED_PORTS, "exactly one port config (8334)")
    check([c.get("Target") for c in kinds.get("Path", [])] == EXPECTED_PATHS, "exactly two path configs")

    for c in configs:
        target = c.get("Target")
        check(c.get("Type") in {"Variable", "Path", "Port"}, f"{target}: Type is Variable/Path/Port")
        check(c.get("Display") in {"always", "advanced"}, f"{target}: Display is always/advanced")
        for attr in ("Required", "Mask"):
            check(c.get(attr) in {"true", "false"}, f"{target}: {attr} is true/false")
        check(bool(c.get("Description")), f"{target}: has a non-empty Description")

    for c in kinds.get("Variable", []):
        target = c.get("Target")
        check(c.get("Required") == "false", f"{target}: Required is false")
        check(c.get("Mask") == ("true" if target in MASKED else "false"), f"{target}: Mask value correct")

    dropdown_targets = {c.get("Target") for c in kinds.get("Variable", []) if "|" in (c.get("Default") or "")}
    check(dropdown_targets == set(DROPDOWNS), "only the four documented dropdowns use pipe-separated Defaults")
    for c in kinds.get("Variable", []):
        target = c.get("Target")
        if target in DROPDOWNS:
            default = c.get("Default")
            check(default == DROPDOWNS[target], f"{target}: Default is {DROPDOWNS[target]} (first option is the default)")

    required_true = {c.get("Target") for c in configs if c.get("Required") == "true"}
    check(required_true == {"8334", "/app/data"}, "only the port and appdata path are Required=true")

    if skip_icon:
        print("SKIP  icon checks (--skip-icon)")
    else:
        check_icon()

    print()
    if FAILS:
        print(f"{len(FAILS)} check(s) FAILED")
        return 1
    print("ALL CHECKS PASSED")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

- [ ] **Step 2: Run it to verify it fails**

Run: `python3 /tmp/verify-filestash-template.py`

Expected: exit code 1 and a single line
`FAIL  template exists: /data/templates/unraid/binhex/templates/official-filestash.xml` — the template does not
exist yet.

- [ ] **Step 3: Commit**

Nothing to commit in this task. The validator is a temp file and stays out of git (Global rule 2 and the
spec's "no committed drift-checking script" non-goal). Confirm with `git status --short` that only the
pre-existing `M projects/common/.gitignore` line is present.

---

### Task 3: Write the template

**Files:**

- Create: `unraid/binhex/templates/official-filestash.xml`
- Test: `/tmp/verify-filestash-template.py --skip-icon`

- [ ] **Step 1: Write the file**

Write exactly this content to `/data/templates/unraid/binhex/templates/official-filestash.xml`:

```xml
<?xml version="1.0"?>
<Container version="2">
    <Name>binhex-official-filestash</Name>
    <Beta>false</Beta>
    <Category>Cloud: Tools:</Category>
    <Repository>machines/filestash:latest</Repository>
    <Registry>https://hub.docker.com/r/machines/filestash/</Registry>
    <Network>bridge</Network>
    <Privileged>false</Privileged>
    <PrivilegedReq>false</PrivilegedReq>
    <ExtraParams></ExtraParams>
    <Shell>bash</Shell>
    <GitHub>https://github.com/mickael-kerjean/filestash</GitHub>
    <Support>https://github.com/mickael-kerjean/filestash/issues</Support>
    <Project>https://github.com/mickael-kerjean/filestash</Project>
    <Overview>Filestash is a storage-agnostic web file manager that gives you a single UI over SFTP, S3, SMB, WebDAV, FTP, NFS and IPFS backends, with office and media preview plugins and optional OIDC, SAML and LDAP authentication. Storage backends, themes and plugins are configured in the admin UI rather than through environment variables.</Overview>
    <WebUI>http://[IP]:[PORT:8334]</WebUI>
    <TemplateURL>https://raw.githubusercontent.com/binhex/templates/main/unraid/binhex/templates/official-filestash.xml</TemplateURL>
    <Icon>https://raw.githubusercontent.com/binhex/templates/main/unraid/binhex/images/official-filestash-icon.png</Icon>
    <DonateText>If you appreciate my work, then please consider buying me a beer :D</DonateText>
    <DonateLink>https://www.paypal.com/cgi-bin/webscr?cmd=_s-xclick&amp;hosted_button_id=MM5E27UX6AUU4</DonateLink>
    <DonateImg>https://www.paypal.com/en_US/i/btn/btn_donate_SM.gif</DonateImg>
    <Date>1970-01-01</Date>
    <Changes></Changes>

    <Config Name="Port: Web Interface" Target="8334" Default="8334" Mode="tcp" Description="This is the TCP port for the Filestash WebUI" Type="Port" Display="always" Required="true" Mask="false"/>

    <Config Name="Path: /app/data" Target="/app/data" Default="/mnt/user/appdata/filestash" Mode="rw" Description="This is the container path to the Filestash data directory. It holds the config, certificates, database, search index, plugins, logs and cache, so it must persist." Type="Path" Display="always" Required="true" Mask="false"/>
    <Config Name="Path: /media" Target="/media" Default="/mnt/user" Mode="rw" Description="Optional path exposed to the local storage backend so it can browse array data. Narrow this to a specific share rather than all of /mnt/user." Type="Path" Display="advanced" Required="false" Mask="false"/>

    <Config Name="Variable: APPLICATION_URL" Target="APPLICATION_URL" Default="" Description="Public base URL, for example https://files.example.com. Set this when running Filestash behind a reverse proxy." Type="Variable" Display="always" Required="false" Mask="false"/>
    <Config Name="Variable: ADMIN_PASSWORD" Target="ADMIN_PASSWORD" Default="" Description="Pre-set admin password. Leave blank to keep the upstream first-run setup wizard." Type="Variable" Display="always" Required="false" Mask="true"/>
    <Config Name="Variable: FILESTASH_BASE" Target="FILESTASH_BASE" Default="" Description="URL base path when serving Filestash from a subfolder, for example /filestash" Type="Variable" Display="advanced" Required="false" Mask="false"/>
    <Config Name="Variable: LOG_LEVEL" Target="LOG_LEVEL" Default="INFO|DEBUG|WARNING|ERROR" Description="Logging verbosity, default INFO" Type="Variable" Display="always" Required="false" Mask="false"/>
    <Config Name="Variable: DEBUG" Target="DEBUG" Default="false|true" Description="Verbose debug mode" Type="Variable" Display="advanced" Required="false" Mask="false"/>

    <Config Name="Variable: CONFIG_SECRET" Target="CONFIG_SECRET" Default="" Description="Encryption key for config.json. It must be set before first start: setting it afterwards can make an already encrypted config unreadable. Leave blank to derive one automatically." Type="Variable" Display="advanced" Required="false" Mask="true"/>
    <Config Name="Variable: CONFIG_ENCRYPT" Target="CONFIG_ENCRYPT" Default="true|false" Description="Encrypt the config at rest, default true" Type="Variable" Display="advanced" Required="false" Mask="false"/>

    <Config Name="Variable: OFFICE_URL" Target="OFFICE_URL" Default="" Description="WOPI server URL for office editing, for example http://collabora:9980. Office editing stays disabled until a WOPI server is supplied." Type="Variable" Display="advanced" Required="false" Mask="false"/>
    <Config Name="Variable: OFFICE_FILESTASH_URL" Target="OFFICE_FILESTASH_URL" Default="" Description="URL the WOPI server uses to reach Filestash" Type="Variable" Display="advanced" Required="false" Mask="false"/>
    <Config Name="Variable: OFFICE_REWRITE_URL" Target="OFFICE_REWRITE_URL" Default="" Description="URL rewrite used by the WOPI host" Type="Variable" Display="advanced" Required="false" Mask="false"/>
    <Config Name="Variable: ONLYOFFICE_URL" Target="ONLYOFFICE_URL" Default="" Description="ONLYOFFICE document server URL for office editing" Type="Variable" Display="advanced" Required="false" Mask="false"/>

    <Config Name="Variable: SYNCTHING_URL" Target="SYNCTHING_URL" Default="" Description="Syncthing widget endpoint" Type="Variable" Display="advanced" Required="false" Mask="false"/>
    <Config Name="Variable: GDRIVE_CLIENT_ID" Target="GDRIVE_CLIENT_ID" Default="" Description="Google Drive OAuth app client id" Type="Variable" Display="advanced" Required="false" Mask="false"/>
    <Config Name="Variable: GDRIVE_CLIENT_SECRET" Target="GDRIVE_CLIENT_SECRET" Default="" Description="Google Drive OAuth app client secret" Type="Variable" Display="advanced" Required="false" Mask="true"/>
    <Config Name="Variable: DROPBOX_CLIENT_ID" Target="DROPBOX_CLIENT_ID" Default="" Description="Dropbox OAuth app client id" Type="Variable" Display="advanced" Required="false" Mask="false"/>
    <Config Name="Variable: LOCAL_BACKEND_SECRET" Target="LOCAL_BACKEND_SECRET" Default="" Description="Secret for the local storage backend" Type="Variable" Display="advanced" Required="false" Mask="true"/>
    <Config Name="Variable: PLG_AUTHENTICATE_LOCAL_ENABLED" Target="PLG_AUTHENTICATE_LOCAL_ENABLED" Default="false|true" Description="Enable the local authentication plugin" Type="Variable" Display="advanced" Required="false" Mask="false"/>

    <Config Name="Variable: FEATURE_TRANSCODING_VIDEO_BLACKLIST" Target="FEATURE_TRANSCODING_VIDEO_BLACKLIST" Default="" Description="Video formats that will not be transcoded. Feature flag default for the on-demand transcoder; normally set in the admin UI." Type="Variable" Display="advanced" Required="false" Mask="false"/>
</Container>
```

- [ ] **Step 2: Parse-check with an independent tool**

Run: `xmllint --noout /data/templates/unraid/binhex/templates/official-filestash.xml`

Expected: no output, exit code 0.

- [ ] **Step 3: Run the validator (icon deferred)**

Run: `python3 /tmp/verify-filestash-template.py --skip-icon`

Expected: every line `PASS  ...`, one line `SKIP  icon checks (--skip-icon)`, and finally
`ALL CHECKS PASSED`, exit code 0. If any check fails, fix the XML — do not edit the validator to match the
file; the validator encodes the spec.

- [ ] **Step 4: Commit**

```bash
cd /data/templates
git add unraid/binhex/templates/official-filestash.xml
git status --short
git commit -m "Add official-filestash unraid template

18 source-verified env vars from mickael-kerjean/filestash at 6feaa21,
1 port (8334) and 2 paths (/app/data required, /media optional).
FILESTASH_PORT, FILESTASH_PATH and LICENSE deliberately omitted; HOME and
PATH are system variables, not settings."
```

Expected: `git status --short` shows the new file as `A` and still `M projects/common/.gitignore` (never
staged).

---

### Task 4: Add the template icon

**Files:**

- Create: `unraid/binhex/images/official-filestash-icon.png`
- Test: `/tmp/verify-filestash-template.py` (full run, icon checks enabled)

- [ ] **Step 1: Fetch the upstream logo**

```bash
mkdir -p /tmp/fs-logo
curl -fsSL -o /tmp/fs-logo/official-filestash-icon.png \
  https://raw.githubusercontent.com/mickael-kerjean/filestash/master/public/assets/logo/android-chrome-512x512.png
ls -l /tmp/fs-logo/official-filestash-icon.png
file /tmp/fs-logo/official-filestash-icon.png
```

Expected: 18 564 bytes (upstream may differ slightly); `file` reports
`PNG image data, 512 x 512, 8-bit colormap, non-interlaced`.

If the URL 404s, fall back to the pinned revision
`https://raw.githubusercontent.com/mickael-kerjean/filestash/6feaa2127a495880641ed2b70aa7abeae9e8367f/public/assets/logo/android-chrome-512x512.png`.
Do not substitute a different asset or hand-draw an icon; if both URLs fail, STOP and report.

- [ ] **Step 2: Verify it is a real PNG before copying**

```bash
python3 - <<'PY'
import struct
d = open('/tmp/fs-logo/official-filestash-icon.png', 'rb').read(33)
assert d[:8] == b'\x89PNG\r\n\x1a\n', 'not a PNG'
w, h = struct.unpack('>II', d[16:24])
assert (w, h) == (512, 512), f'unexpected size {w}x{h}'
print('PNG OK', f'{w}x{h}')
PY
```

Expected: `PNG OK 512x512`.

- [ ] **Step 3: Copy into the repo**

```bash
cp /tmp/fs-logo/official-filestash-icon.png /data/templates/unraid/binhex/images/official-filestash-icon.png
ls -l /data/templates/unraid/binhex/images/official-filestash-icon.png
```

Expected: the file exists with the same size as the downloaded copy.

- [ ] **Step 4: Run the full validator**

Run: `python3 /tmp/verify-filestash-template.py`

Expected: every line `PASS  ...` including `PASS  icon file exists:`, `PASS  icon is a PNG (magic bytes)` and
`PASS  icon is 512x512 (got 512x512)`, ending in `ALL CHECKS PASSED`, exit code 0.

- [ ] **Step 5: Commit**

```bash
cd /data/templates
git add unraid/binhex/images/official-filestash-icon.png
git commit -m "Add icon for official-filestash template

Upstream public/assets/logo/android-chrome-512x512.png copied verbatim;
no image tooling available in this environment."
```

Expected: commit succeeds, `git status --short` still shows only `M projects/common/.gitignore`.

---

### Task 5: Correct the design spec to 18 variables

**Files:**

- Modify: `docs/agent/specs/2026-10-17-official-filestash-template-design.md` (lines 38, 111-115, 120-127, 142, 145-151)

- [ ] **Step 1: Correct the source-surface paragraph (line 38)**

Replace:

```markdown
`server/`. Twenty names are read; seventeen are exposed.
```

with:

```markdown
`server/`. Twenty-three names are read; eighteen are exposed. Three are deliberate omissions
(`FILESTASH_PORT`, `FILESTASH_PATH`, `LICENSE`) and two (`HOME`, `PATH`) are environment fundamentals rather
than application settings, so they are filtered out.
```

- [ ] **Step 2: Add the missing variable row to the environment table**

Immediately after the `PLG_AUTHENTICATE_LOCAL_ENABLED` row (line 111), add:

```markdown
| `FEATURE_TRANSCODING_VIDEO_BLACKLIST` | (blank) | `advanced` | false | Video formats that will not be transcoded; feature flag default for the on-demand transcoder, normally set in `/admin` |
```

- [ ] **Step 3: Extend the omission note (after line 127)**

After the `CANARY` row of the "Omitted variables" table and its blank line, insert:

```markdown
`HOME` and `PATH` are also read by the binary but are environment fundamentals, not user settings, so they are
not template entries. `FEATURE_TRANSCODING_VIDEO_BLACKLIST` is **not** in this table: it is read by
`server/plugin/plg_video_transcoder/config.go` (via `os.Getenv`) and is exposed as the eighteenth variable.
```

- [ ] **Step 4: Update the verification checklist**

Replace line 142:

```markdown
2. All 17 variable targets are present exactly once, with no duplicate `Target` values across the file.
```

with:

```markdown
2. All 18 variable targets are present exactly once, with no duplicate `Target` values across the file.
```

Then replace lines 145-151 (the item 5 source list) with:

```markdown
5. The target list matches a grep of the upstream source checkout (`server/pkg/config/config.go`,
   `server/pkg/config/config_state.go`, `server/pkg/env/constants.go`, `server/plugin/plg_editor_wopi`,
   `server/plugin/plg_editor_onlyoffice`, `server/plugin/plg_handler_syncthing`,
   `server/plugin/plg_video_transcoder`, `server/plugin/plg_backend_gdrive`,
   `server/plugin/plg_backend_dropbox`, `server/plugin/plg_backend_local`,
   `server/plugin/plg_authenticate_local`, `server/plugin/plg_license`),
   with `FILESTASH_PORT`, `FILESTASH_PATH` and `LICENSE` accounted for as the three deliberate omissions and
   `HOME` / `PATH` filtered as environment fundamentals.
   `CANARY` is not one of them: the current source never reads it.
```

- [ ] **Step 5: Lint the markdown**

Run: `markdownlint --fix docs/agent/specs/2026-10-17-official-filestash-template-design.md`

Expected: no remaining violations (re-run without `--fix` if the tool prints any and confirm zero output).

- [ ] **Step 6: Commit**

```bash
cd /data/templates
git add docs/agent/specs/2026-10-17-official-filestash-template-design.md
git commit -m "Update filestash spec for transcoding env var

The surface is 23 names read, 18 exposed: FEATURE_TRANSCODING_VIDEO_BLACKLIST
was missed because plg_video_transcoder was absent from the source list. HOME
and PATH documented as filtered environment fundamentals."
```

---

### Task 6: Consolidated gate against the spec's own checklist

**Files:**

- Test: `/tmp/verify-filestash-template.py`, `xmllint`, `markdownlint`, `git status`

- [ ] **Step 1: Run every spec verification item**

```bash
cd /data/templates
echo "--- item 1: XML parses ---"
python3 -c "import xml.etree.ElementTree as ET; ET.parse('unraid/binhex/templates/official-filestash.xml'); print('parsed')"
xmllint --noout unraid/binhex/templates/official-filestash.xml && echo "xmllint clean"
echo "--- items 2,3,4,6,7: validator ---"
python3 /tmp/verify-filestash-template.py
echo "--- item 5: target list vs source surface ---"
grep -oE 'Target="[^"]+"' unraid/binhex/templates/official-filestash.xml | sed 's/Target="//;s/"//' | sort
echo "--- item 8: spec markdown ---"
markdownlint docs/agent/specs/2026-10-17-official-filestash-template-design.md && echo "markdownlint clean"
```

Expected: `parsed`, `xmllint clean`, `ALL CHECKS PASSED`, the 21 targets sorted (`/app/data`, `/media`, `8334`
plus the 18 variable names), and `markdownlint clean`.

- [ ] **Step 2: Confirm no collateral changes**

```bash
cd /data/templates
git status --short
git log --oneline -3
```

Expected: `git status --short` shows exactly one line, `M projects/common/.gitignore` (pre-existing, unrelated,
left unstaged). The log shows the three new commits from Tasks 3-5.

- [ ] **Step 3: Report**

Summarise: the template path, the icon path, the spec correction, the validator output, and the 8/8 spec
checklist items with their evidence. Do not claim completion without the Step 1 output in hand.

No commit in this task unless a fix was needed; if a fix was needed, commit it with
`git add <exact path> && git commit -m "Fix official-filestash <what>"`.
