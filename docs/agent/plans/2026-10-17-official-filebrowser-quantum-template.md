# official-filebrowser-quantum unRAID Template Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use sub-agents (recommended) to implement this plan task-by-task.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship `unraid/binhex/templates/official-filebrowser-quantum.xml` and its 256x256 icon, exposing the
12 user-meaningful environment variables of the `gtstef/filebrowser:stable` image in the existing `official-*`
house format.

**Architecture:** A single declarative unRAID template file plus a copied PNG icon. No application code, no build
step, no runtime component. Correctness is verified by an ad-hoc Python validator kept in `/tmp` (never committed)
and by the approved design spec's own 8-point checklist.

**Tech Stack:** XML (unRAID Docker template schema), Python 3 stdlib for validation, `xmllint` and `markdownlint`
as available CLI gates.

---

## Scope check

Single subsystem: one template, one icon. The approved design spec already exists and is committed, so this plan
contains no spec-editing task. No decomposition required.

## Repo facts (verified during planning)

| Fact | Value |
| --- | --- |
| Repo root | `/data/templates` (branch `main`, HEAD `4ec700f`) |
| Working CWD | `/data/templates/unraid/binhex/templates` |
| Approved spec | `docs/agent/specs/2026-10-17-official-filebrowser-quantum-template-design.md` (committed `4ec700f`) |
| Upstream revision inspected | `91bc608f0ec6a3246c29e5fc87acdd46a5fadf44` (`gtsteffaniak/filebrowser`, 2026-10-13) |
| Reference template | `unraid/binhex/templates/official-qbit-guard.xml` (identity-block conventions) |
| Icon source | upstream `frontend/public/img/icons/favicon.png`, already 256x256 |
| Image | `gtstef/filebrowser:stable` (Docker Hub). GHCR has **no** `stable` tag and no `2.x` tag |
| Gates available | `xmllint` (`/usr/sbin/xmllint`), `markdownlint` (repo ships no config; project standard is 120) |

## CRITICAL: the index already holds unrelated staged files

`git status` at planning time:

```text
M  docs/agent/specs/2026-10-17-official-filestash-template-design.md
 M projects/common/.gitignore
AM unraid/binhex/images/official-filestash-icon.png
A  unraid/binhex/templates/official-filestash.xml
```

Those filestash entries are **staged in the index** and belong to a different, unfinished chain. A plain
`git commit` would sweep them into this plan's commits. Every commit step below therefore:

1. stages only its own file with an explicit path, and
2. commits with `git commit --only <path>`, which commits that path and leaves every other staged entry alone.

This technique was already proven in this repository: `git add <newspec> && git commit --only <newspec>` produced
a one-file commit (`4ec700f`) while the filestash entries stayed staged.

**Never** use `git add -A`, `git add .`, or a bare `git commit` in this plan. `projects/common/.gitignore` is a
pre-existing unrelated modification and must stay untouched and unstaged.

## House-convention decisions

| Element | Value | Why |
| --- | --- | --- |
| Port label | `Port: Web Interface` | matches `official-metube.xml` and `official-pihole.xml` |
| Path labels | `Path: /home/filebrowser/data`, `Path: /srv` | all existing `Path` configs name the container target |
| `WebUI` | `http://[IP]:[PORT:80]` | no trailing slash, matching `official-filestash.xml` and the `8000`/`8006` house examples |
| Indentation | four spaces, no tabs | the tab-indented `<Beta>` in some existing files is an inconsistency, not a convention |
| `Category` | `Productivity: Tools:` | matches the sibling `filebrowser.xml` |

---

### Task 1: Re-verify the upstream env surface and registry facts

**Files:**

- Create: none (evidence only)
- Modify: none

- [ ] **Step 1: Fetch an upstream checkout with only the needed subtrees**

```bash
cd /tmp
rm -rf fbq-src
git clone --depth 1 --filter=blob:none --sparse https://github.com/gtsteffaniak/filebrowser fbq-src
cd /tmp/fbq-src
git sparse-checkout set _docker backend frontend/public
git log -1 --format='%H %ad %s'
```

Expected: the clone succeeds and prints the revision. Planning observed
`91bc608f0ec6a3246c29e5fc87acdd46a5fadf44` ("Beta/v2.1.2 (#3142)", 2026-10-13). A different hash is acceptable
(upstream moves), but record it in the commit body, because the spec cites the surface as source-verified.

- [ ] **Step 2: Re-derive the env read sites**

```bash
cd /tmp/fbq-src
grep -rhoE '(os\.Getenv|LookupEnv|Getenv)\("[A-Za-z_0-9]+"\)' --include='*.go' backend/ \
  | sed 's/.*("//;s/")//' | sort | uniq -c | sort -rn
```

Expected: these `FILEBROWSER_*` names, plus two non-settings (each shown with planning's observed counts):

```text
FILEBROWSER_GENERATE_CONFIG
FILEBROWSER_DEVMODE
FILEBROWSER_DATABASE_PATH
FILEBROWSER_CONFIG
FILEBROWSER_SQL_WAL
FILEBROWSER_ADMIN_PASSWORD
FILEBROWSER_DISABLE_AUTOMATIC_BACKUP
FILEBROWSER_DATABASE            <-- v1.x only, removed in v2.0
FILEBROWSER_FFMPEG_PATH
FILEBROWSER_JWT_TOKEN_SECRET
FILEBROWSER_LDAP_USER_PASSWORD
FILEBROWSER_OIDC_CLIENT_ID
FILEBROWSER_OIDC_CLIENT_SECRET
FILEBROWSER_ONLYOFFICE_SECRET
FILEBROWSER_PLAYWRIGHT_TEST
FILEBROWSER_RECAPTCHA_SECRET
FILEBROWSER_TOTP_SECRET
KUBERNETES_SERVICE_HOST         <-- in-cluster analytics detection, not a setting
GOFFMPEG_FFMPEG_PATH            <-- Go test hook
GOFFMPEG_SAMPLE_MP4             <-- Go test hook
```

- [ ] **Step 3: Assert the partition adds up**

```bash
cd /tmp/fbq-src
python3 - <<'PY'
found = {"FILEBROWSER_ADMIN_PASSWORD","FILEBROWSER_JWT_TOKEN_SECRET","FILEBROWSER_TOTP_SECRET",
"FILEBROWSER_RECAPTCHA_SECRET","FILEBROWSER_LDAP_USER_PASSWORD","FILEBROWSER_OIDC_CLIENT_ID",
"FILEBROWSER_OIDC_CLIENT_SECRET","FILEBROWSER_ONLYOFFICE_SECRET","FILEBROWSER_CONFIG",
"FILEBROWSER_DATABASE_PATH","FILEBROWSER_FFMPEG_PATH","FILEBROWSER_DISABLE_AUTOMATIC_BACKUP",
"FILEBROWSER_DEVMODE","FILEBROWSER_SQL_WAL","FILEBROWSER_GENERATE_CONFIG","FILEBROWSER_PLAYWRIGHT_TEST",
"FILEBROWSER_DATABASE"}
exposed = {n for n in found if n not in {"FILEBROWSER_DEVMODE","FILEBROWSER_SQL_WAL",
"FILEBROWSER_GENERATE_CONFIG","FILEBROWSER_PLAYWRIGHT_TEST","FILEBROWSER_DATABASE"}}
print("read:", len(found), "| exposed:", len(exposed), "| omissions:", len(found - exposed))
assert len(exposed) == 12, sorted(exposed)
print("partition OK: 12 exposed + 5 documented omissions")
PY
```

Expected: `read: 17 | exposed: 12 | omissions: 5` and `partition OK`.

- [ ] **Step 4: Re-confirm the image facts the template depends on**

```bash
cd /tmp/fbq-src
grep -nE 'EXPOSE|HEALTHCHECK|USER|ENTRYPOINT|^ENV' _docker/Dockerfile
grep -nE '^  port:|sources:|^    - path:' backend/config.yaml
python3 -c "
import struct; d=open('frontend/public/img/icons/favicon.png','rb').read(33)
assert d[:8]==b'\x89PNG\r\n\x1a\n'; w,h=struct.unpack('>II',d[16:24]); print('favicon', f'{w}x{h}')"
```

Expected: `EXPOSE 80`; healthcheck on `localhost:80/health`; `USER filebrowser`; entrypoint `./filebrowser`;
`ENV` lines setting `FILEBROWSER_FFMPEG_PATH=/usr/local/bin/`,
`FILEBROWSER_DATABASE_PATH=/home/filebrowser/data/database.sqlite`,
`FILEBROWSER_CONFIG=/home/filebrowser/data/config.yaml`; `port: 80`; `sources:` with `- path: /srv`;
`favicon 256x256`.

- [ ] **Step 5: Re-confirm the registry claim behind the image choice**

```bash
TOKEN=$(curl -s --max-time 25 "https://auth.docker.io/token?service=registry.docker.io&scope=repository:gtstef/filebrowser:pull" | python3 -c "import json,sys;print(json.load(sys.stdin)['token'])")
curl -s --max-time 30 -H "Authorization: Bearer $TOKEN" "https://registry-1.docker.io/v2/gtstef/filebrowser/tags/list" \
  | python3 -c "import json,sys; t=set(json.load(sys.stdin).get('tags') or []); print('DH stable:', 'stable' in t, '| DH stable-slim:', 'stable-slim' in t)"

GTOKEN=$(curl -s --max-time 25 "https://ghcr.io/token?scope=repository:gtsteffaniak/filebrowser:pull" | python3 -c "import json,sys;print(json.load(sys.stdin)['token'])")
curl -s --max-time 30 -H "Authorization: Bearer $GTOKEN" "https://ghcr.io/v2/gtsteffaniak/filebrowser/tags/list" \
  | python3 -c "import json,sys; t=set(json.load(sys.stdin).get('tags') or []); print('GHCR stable:', 'stable' in t, '| GHCR 2.x tags:', sorted(x for x in t if x.startswith('2.'))[:5])"
```

Expected: `DH stable: True | DH stable-slim: True` and `GHCR stable: False | GHCR 2.x tags: []`.

- [ ] **Step 6: Record the evidence in the task notes**

No repo file is written in this task. Paste the command outputs into the sub-agent report; the commit bodies of
Tasks 3-4 cite them. If any name outside the expected set appears, or the partition fails, STOP and report — the
12-variable set would need re-approval before Task 3.

---

### Task 2: Write the validator and watch it fail

**Files:**

- Create: `/tmp/verify-fbq-template.py` (temp only, never committed)
- Test: itself

- [ ] **Step 1: Write the validator**

```python
#!/usr/bin/env python3
"""Ad-hoc validator for the official-filebrowser-quantum unRAID template. Temp file; never commit."""

import struct
import sys
import xml.etree.ElementTree as ET
from pathlib import Path

ROOT = Path("/data/templates")
TPL = ROOT / "unraid/binhex/templates/official-filebrowser-quantum.xml"
REF = ROOT / "unraid/binhex/templates/official-qbit-guard.xml"
ICON = ROOT / "unraid/binhex/images/official-filebrowser-quantum-icon.png"

EXPECTED_VARIABLES = [
    "FILEBROWSER_ADMIN_PASSWORD",
    "FILEBROWSER_JWT_TOKEN_SECRET",
    "FILEBROWSER_TOTP_SECRET",
    "FILEBROWSER_RECAPTCHA_SECRET",
    "FILEBROWSER_LDAP_USER_PASSWORD",
    "FILEBROWSER_OIDC_CLIENT_ID",
    "FILEBROWSER_OIDC_CLIENT_SECRET",
    "FILEBROWSER_ONLYOFFICE_SECRET",
    "FILEBROWSER_CONFIG",
    "FILEBROWSER_DATABASE_PATH",
    "FILEBROWSER_FFMPEG_PATH",
    "FILEBROWSER_DISABLE_AUTOMATIC_BACKUP",
]
EXPECTED_PORTS = ["80"]
EXPECTED_PATHS = ["/home/filebrowser/data", "/srv"]
EXPECTED_ALL = set(EXPECTED_VARIABLES) | set(EXPECTED_PORTS) | set(EXPECTED_PATHS)

DROPDOWNS = {"FILEBROWSER_DISABLE_AUTOMATIC_BACKUP": "false|true"}
MASKED = {
    "FILEBROWSER_ADMIN_PASSWORD",
    "FILEBROWSER_JWT_TOKEN_SECRET",
    "FILEBROWSER_TOTP_SECRET",
    "FILEBROWSER_RECAPTCHA_SECRET",
    "FILEBROWSER_LDAP_USER_PASSWORD",
    "FILEBROWSER_OIDC_CLIENT_SECRET",
    "FILEBROWSER_ONLYOFFICE_SECRET",
}

TEMPLATE_URL = (
    "https://raw.githubusercontent.com/binhex/templates/main/unraid/binhex/templates/"
    "official-filebrowser-quantum.xml"
)
ICON_URL = "https://raw.githubusercontent.com/binhex/templates/main/unraid/binhex/images/official-filebrowser-quantum-icon.png"

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
        check((width, height) == (256, 256), f"icon is 256x256 (got {width}x{height})")


def main():
    skip_icon = "--skip-icon" in sys.argv
    if not TPL.is_file():
        print(f"FAIL  template exists: {TPL}")
        return 1
    check(True, f"template exists: {TPL}")

    raw = TPL.read_text(encoding="utf-8")
    check("\t" not in raw, "file uses spaces only (no tab characters)")

    root = ET.fromstring(raw)
    check(root.tag == "Container", "root element is <Container>")
    check(root.get("version") == "2", 'Container version="2"')

    ref_root = ET.parse(REF).getroot()
    ref_order = [c.tag for c in ref_root if c.tag != "Config"]
    our_order = [c.tag for c in root if c.tag != "Config"]
    check(our_order == ref_order, f"identity-block element order matches official-qbit-guard.xml ({ref_order})")

    texts = {c.tag: (c.text or "") for c in root if c.tag != "Config"}
    check(texts.get("Name") == "binhex-official-filebrowser-quantum", "Name is binhex-official-filebrowser-quantum")
    check(texts.get("Category") == "Productivity: Tools:", "Category is Productivity: Tools:")
    check(texts.get("Repository") == "gtstef/filebrowser:stable", "Repository pins gtstef/filebrowser:stable")
    check(texts.get("Registry") == "https://hub.docker.com/r/gtstef/filebrowser", "Registry points at Docker Hub")
    check(texts.get("Network") == "bridge", "Network is bridge")
    check(texts.get("WebUI") == "http://[IP]:[PORT:80]", "WebUI points at [PORT:80]")
    check(texts.get("TemplateURL") == TEMPLATE_URL, "TemplateURL matches the committed path")
    check(texts.get("Icon") == ICON_URL, "Icon matches the committed path")
    check(texts.get("Date") == "1970-01-01", "Date is 1970-01-01")
    check(texts.get("Beta") == "false", "Beta is false")
    check(texts.get("GitHub") == "https://github.com/gtsteffaniak/filebrowser", "GitHub link is upstream")
    check(texts.get("Support") == "https://github.com/gtsteffaniak/filebrowser/issues", "Support link is upstream issues")
    check("config.yaml" in (texts.get("Overview") or ""), "Overview mentions config.yaml as the primary config")

    configs = root.findall("Config")
    targets = [c.get("Target") for c in configs]
    check(len(targets) == len(set(targets)), "no duplicate Target values")
    check(len(targets) == 15, f"exactly 15 Config entries (got {len(targets)})")
    check(set(targets) == EXPECTED_ALL, "target set is exactly the expected 15 entries")

    kinds = {}
    for c in configs:
        kind = (c.get("Name") or "").split(":")[0]
        kinds.setdefault(kind, []).append(c)

    var_targets = [c.get("Target") for c in kinds.get("Variable", [])]
    check(sorted(var_targets) == sorted(EXPECTED_VARIABLES), "all 12 variables present exactly once")
    check([c.get("Target") for c in kinds.get("Port", [])] == EXPECTED_PORTS, "exactly one port config (80)")
    check([c.get("Target") for c in kinds.get("Path", [])] == EXPECTED_PATHS, "exactly two path configs, in order")

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
    check(dropdown_targets == set(DROPDOWNS), "only the one documented dropdown uses a pipe-separated Default")
    for c in kinds.get("Variable", []):
        target = c.get("Target")
        if target in DROPDOWNS:
            check(c.get("Default") == DROPDOWNS[target],
                  f"{target}: Default is {DROPDOWNS[target]} (first option is the effective default)")

    always = [c.get("Target") for c in kinds.get("Variable", []) if c.get("Display") == "always"]
    check(always == ["FILEBROWSER_ADMIN_PASSWORD"], f"only the admin password is Display=always (got {always})")

    required_true = {c.get("Target") for c in configs if c.get("Required") == "true"}
    check(required_true == {"80", "/home/filebrowser/data"}, "only the port and data path are Required=true")

    masked_true = {c.get("Target") for c in configs if c.get("Mask") == "true"}
    check(masked_true == MASKED, f"exactly the seven credential variables are masked (got {sorted(masked_true)})")

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

Run: `python3 /tmp/verify-fbq-template.py`

Expected: exit code 1 and a single line
`FAIL  template exists: /data/templates/unraid/binhex/templates/official-filebrowser-quantum.xml` — the template
does not exist yet.

- [ ] **Step 3: Commit**

Nothing to commit. The validator is a temp file and stays out of git (Global rule 2 and the spec's
"no committed drift-checking script" non-goal). Confirm with `git status --short` that no validator file appears
and that the filestash entries are exactly as they were.

---

### Task 3: Write the template

**Files:**

- Create: `unraid/binhex/templates/official-filebrowser-quantum.xml`
- Test: `/tmp/verify-fbq-template.py --skip-icon`

- [ ] **Step 1: Write the file**

Write exactly this content to
`/data/templates/unraid/binhex/templates/official-filebrowser-quantum.xml`:

```xml
<?xml version="1.0"?>
<Container version="2">
    <Name>binhex-official-filebrowser-quantum</Name>
    <Beta>false</Beta>
    <Category>Productivity: Tools:</Category>
    <Repository>gtstef/filebrowser:stable</Repository>
    <Registry>https://hub.docker.com/r/gtstef/filebrowser</Registry>
    <Network>bridge</Network>
    <Privileged>false</Privileged>
    <PrivilegedReq>false</PrivilegedReq>
    <ExtraParams></ExtraParams>
    <Shell>bash</Shell>
    <GitHub>https://github.com/gtsteffaniak/filebrowser</GitHub>
    <Support>https://github.com/gtsteffaniak/filebrowser/issues</Support>
    <Project>https://github.com/gtsteffaniak/filebrowser</Project>
    <Overview>FileBrowser Quantum is an actively maintained fork of the original filebrowser with full-text search, previews and thumbnails, sharing links, multi-source support and optional OIDC, LDAP and proxy authentication. Its primary configuration is config.yaml inside the data volume, so most settings are edited there rather than exposed here; the environment variables below cover secrets and overrides.</Overview>
    <WebUI>http://[IP]:[PORT:80]</WebUI>
    <TemplateURL>https://raw.githubusercontent.com/binhex/templates/main/unraid/binhex/templates/official-filebrowser-quantum.xml</TemplateURL>
    <Icon>https://raw.githubusercontent.com/binhex/templates/main/unraid/binhex/images/official-filebrowser-quantum-icon.png</Icon>
    <DonateText>If you appreciate my work, then please consider buying me a beer :D</DonateText>
    <DonateLink>https://www.paypal.com/cgi-bin/webscr?cmd=_s-xclick&amp;hosted_button_id=MM5E27UX6AUU4</DonateLink>
    <DonateImg>https://www.paypal.com/en_US/i/btn/btn_donate_SM.gif</DonateImg>
    <Date>1970-01-01</Date>
    <Changes></Changes>

    <Config Name="Port: Web Interface" Target="80" Default="8080" Mode="tcp" Description="This is the TCP port for the FileBrowser Quantum WebUI. The container listens on port 80 because the image healthcheck and the shipped config require it, so change the host port here rather than http.port in config.yaml." Type="Port" Display="always" Required="true" Mask="false"/>

    <Config Name="Path: /home/filebrowser/data" Target="/home/filebrowser/data" Default="/mnt/user/appdata/filebrowser-quantum" Mode="rw" Description="This is the container path to your configuration file, database and cache, so it must persist. The image runs as UID:GID 1000:1000; to run it as a different user set --user in extra parameters and give this host directory matching ownership." Type="Path" Display="always" Required="true" Mask="false"/>
    <Config Name="Path: /srv" Target="/srv" Default="/mnt/user" Mode="rw" Description="This is the path FileBrowser indexes as its default source. Narrow this to a specific share rather than all of /mnt/user, and add further sources in config.yaml." Type="Path" Display="always" Required="false" Mask="false"/>

    <Config Name="Variable: FILEBROWSER_ADMIN_PASSWORD" Target="FILEBROWSER_ADMIN_PASSWORD" Default="" Description="Admin password. Leave blank to keep the upstream behaviour: a random admin password is generated, written to the container log once, and a reset is forced at first login." Type="Variable" Display="always" Required="false" Mask="true"/>

    <Config Name="Variable: FILEBROWSER_JWT_TOKEN_SECRET" Target="FILEBROWSER_JWT_TOKEN_SECRET" Default="" Description="Secret used to sign JWT authentication tokens. Changing it after first start invalidates all existing sessions." Type="Variable" Display="advanced" Required="false" Mask="true"/>
    <Config Name="Variable: FILEBROWSER_TOTP_SECRET" Target="FILEBROWSER_TOTP_SECRET" Default="" Description="Encryption secret for stored TOTP two-factor secrets. Set it before enrolling users in TOTP." Type="Variable" Display="advanced" Required="false" Mask="true"/>
    <Config Name="Variable: FILEBROWSER_RECAPTCHA_SECRET" Target="FILEBROWSER_RECAPTCHA_SECRET" Default="" Description="reCAPTCHA secret key used by password authentication when reCAPTCHA is enabled in config.yaml." Type="Variable" Display="advanced" Required="false" Mask="true"/>
    <Config Name="Variable: FILEBROWSER_LDAP_USER_PASSWORD" Target="FILEBROWSER_LDAP_USER_PASSWORD" Default="" Description="Password of the LDAP bind account used to look up users." Type="Variable" Display="advanced" Required="false" Mask="true"/>

    <Config Name="Variable: FILEBROWSER_OIDC_CLIENT_ID" Target="FILEBROWSER_OIDC_CLIENT_ID" Default="" Description="OIDC client id for single sign-on, for example from Authentik, Keycloak or Pocket ID." Type="Variable" Display="advanced" Required="false" Mask="false"/>
    <Config Name="Variable: FILEBROWSER_OIDC_CLIENT_SECRET" Target="FILEBROWSER_OIDC_CLIENT_SECRET" Default="" Description="OIDC client secret for single sign-on." Type="Variable" Display="advanced" Required="false" Mask="true"/>
    <Config Name="Variable: FILEBROWSER_ONLYOFFICE_SECRET" Target="FILEBROWSER_ONLYOFFICE_SECRET" Default="" Description="JWT secret shared with your OnlyOffice document server." Type="Variable" Display="advanced" Required="false" Mask="true"/>

    <Config Name="Variable: FILEBROWSER_CONFIG" Target="FILEBROWSER_CONFIG" Default="" Description="Path to the configuration file. Leave blank to use the image default of /home/filebrowser/data/config.yaml inside the data volume." Type="Variable" Display="advanced" Required="false" Mask="false"/>
    <Config Name="Variable: FILEBROWSER_DATABASE_PATH" Target="FILEBROWSER_DATABASE_PATH" Default="" Description="Path to the SQLite database. Leave blank to use the image default of /home/filebrowser/data/database.sqlite. A path set in config.yaml takes precedence over this variable." Type="Variable" Display="advanced" Required="false" Mask="false"/>
    <Config Name="Variable: FILEBROWSER_FFMPEG_PATH" Target="FILEBROWSER_FFMPEG_PATH" Default="" Description="Directory holding the ffmpeg and ffprobe binaries. Leave blank to use the image default of /usr/local/bin. Only the full image bundles ffmpeg, not the slim variant." Type="Variable" Display="advanced" Required="false" Mask="false"/>
    <Config Name="Variable: FILEBROWSER_DISABLE_AUTOMATIC_BACKUP" Target="FILEBROWSER_DISABLE_AUTOMATIC_BACKUP" Default="false|true" Description="Automatic backups are on by default. Set this to true to disable the backup taken automatically before an upgrade." Type="Variable" Display="advanced" Required="false" Mask="false"/>
</Container>
```

- [ ] **Step 2: Parse-check with an independent tool**

Run: `xmllint --noout /data/templates/unraid/binhex/templates/official-filebrowser-quantum.xml`

Expected: no output, exit code 0.

- [ ] **Step 3: Run the validator (icon deferred)**

Run: `python3 /tmp/verify-fbq-template.py --skip-icon`

Expected: every line `PASS  ...`, one line `SKIP  icon checks (--skip-icon)`, and finally `ALL CHECKS PASSED`,
exit code 0 — including `all 12 variables present exactly once`, `exactly 15 Config entries`,
`only the admin password is Display=always`, `exactly the seven credential variables are masked`, and
`identity-block element order matches official-qbit-guard.xml`. If a check fails, fix the XML — never edit the
validator to match the file.

- [ ] **Step 4: Commit only this file**

```bash
cd /data/templates
git add unraid/binhex/templates/official-filebrowser-quantum.xml
git commit --only unraid/binhex/templates/official-filebrowser-quantum.xml -m "Add official-filebrowser-quantum unraid template

12 source-verified env vars from gtsteffaniak/filebrowser at 91bc608, five
documented omissions (v1-only FILEBROWSER_DATABASE plus four developer/test
knobs). Docker Hub gtstef/filebrowser:stable because GHCR carries no stable
tag. Port 80 with host default 8080, data + /srv volumes."
```

Expected: the commit lists exactly one file. `git show --stat --oneline HEAD` must show
`1 file changed` and only the template path. `git status --short` must still show the filestash entries as
staged (`A`, `AM`, `M`) and `M projects/common/.gitignore` unstaged.

---

### Task 4: Add the template icon

**Files:**

- Create: `unraid/binhex/images/official-filebrowser-quantum-icon.png`
- Test: `/tmp/verify-fbq-template.py` (full run, icon checks enabled)

- [ ] **Step 1: Fetch the upstream favicon**

```bash
mkdir -p /tmp/fbq-icon
curl -fsSL -o /tmp/fbq-icon/official-filebrowser-quantum-icon.png \
  https://raw.githubusercontent.com/gtsteffaniak/filebrowser/main/frontend/public/img/icons/favicon.png
ls -l /tmp/fbq-icon/official-filebrowser-quantum-icon.png
file /tmp/fbq-icon/official-filebrowser-quantum-icon.png
```

Expected: a PNG of about 9 KB reported as `PNG image data, 256 x 256` (measured 9 334 bytes; the sparse-checkout
copy at `frontend/public/img/icons/favicon.png` is byte-identical to the raw URL, confirmed by `cmp`).

If that URL 404s, fall back to the pinned revision
`https://raw.githubusercontent.com/gtsteffaniak/filebrowser/91bc608f0ec6a3246c29e5fc87acdd46a5fadf44/frontend/public/img/icons/favicon.png`,
or copy from the local checkout (`cp /tmp/fbq-src/frontend/public/img/icons/favicon.png /tmp/fbq-icon/...`).
If none of the three works, STOP and report; do not substitute a different asset and do not draw an icon.

- [ ] **Step 2: Verify the downloaded bytes before copying anything**

```bash
python3 - <<'PY'
import struct
p = '/tmp/fbq-icon/official-filebrowser-quantum-icon.png'
d = open(p, 'rb').read(33)
assert d[:8] == b'\x89PNG\r\n\x1a\n', 'not a PNG'
w, h = struct.unpack('>II', d[16:24])
assert (w, h) == (256, 256), f'unexpected size {w}x{h}'
print('PNG OK', f'{w}x{h}')
PY
cmp -s /tmp/fbq-icon/official-filebrowser-quantum-icon.png /tmp/fbq-src/frontend/public/img/icons/favicon.png \
  && echo "IDENTICAL to upstream favicon" || echo "WARNING: differs from upstream favicon in /tmp/fbq-src"
```

Expected: `PNG OK 256x256`. The `cmp` should print `IDENTICAL to upstream favicon` when upstream main still matches
the checkout; a `WARNING` line means upstream moved — record it, and continue only if the file is still a valid
256x256 PNG of the logo (open it and confirm it is the FileBrowser logo, not an unrelated image).

- [ ] **Step 3: Copy into the repo**

```bash
cp /tmp/fbq-icon/official-filebrowser-quantum-icon.png \
   /data/templates/unraid/binhex/images/official-filebrowser-quantum-icon.png
ls -l /data/templates/unraid/binhex/images/official-filebrowser-quantum-icon.png
cmp -s /tmp/fbq-icon/official-filebrowser-quantum-icon.png \
       /data/templates/unraid/binhex/images/official-filebrowser-quantum-icon.png \
  && echo "copy verified byte-identical"
```

Expected: the copy exists and is byte-identical to the verified download.

- [ ] **Step 4: Run the full validator**

Run: `python3 /tmp/verify-fbq-template.py`

Expected: every line `PASS  ...` including `PASS  icon file exists:`, `PASS  icon is a PNG (magic bytes)` and
`PASS  icon is 256x256 (got 256x256)`, ending in `ALL CHECKS PASSED`, exit code 0.

- [ ] **Step 5: Commit only this file**

```bash
cd /data/templates
git add unraid/binhex/images/official-filebrowser-quantum-icon.png
git commit --only unraid/binhex/images/official-filebrowser-quantum-icon.png -m "Add icon for official-filebrowser-quantum template

Copied from upstream frontend/public/img/icons/favicon.png, already 256x256
which matches the existing official-*-icon.png files, so no conversion."
```

Expected: one file in the commit; the filestash entries remain staged and untouched afterwards.

---

### Task 5: Consolidated gate against the spec's own checklist

**Files:**

- Test: `/tmp/verify-fbq-template.py`, `xmllint`, `markdownlint`, `git status`

- [ ] **Step 1: Run every spec verification item**

```bash
cd /data/templates
echo "--- item 1: XML parses ---"
python3 -c "import xml.etree.ElementTree as ET; ET.parse('unraid/binhex/templates/official-filebrowser-quantum.xml'); print('parsed')"
xmllint --noout unraid/binhex/templates/official-filebrowser-quantum.xml && echo "xmllint clean"
echo "--- items 2,3,4,6,7: validator ---"
python3 /tmp/verify-fbq-template.py
echo "--- item 5: target list vs the upstream surface ---"
grep -oE 'Target="[^"]+"' unraid/binhex/templates/official-filebrowser-quantum.xml | sed 's/Target="//;s/"//' | sort
echo "--- item 6b: icon byte-identical to upstream ---"
cmp -s unraid/binhex/images/official-filebrowser-quantum-icon.png /tmp/fbq-src/frontend/public/img/icons/favicon.png \
  && echo "icon IDENTICAL to upstream favicon" || echo "icon differs from upstream checkout"
echo "--- item 8: spec markdown ---"
markdownlint -c /tmp/mdlint.json docs/agent/specs/2026-10-17-official-filebrowser-quantum-template-design.md \
  && echo "markdownlint clean"
```

Expected: `parsed`; `xmllint clean`; `ALL CHECKS PASSED`; the 15 targets sorted (the 12 `FILEBROWSER_*` names plus
`80`, `/home/filebrowser/data`, `/srv`); `icon IDENTICAL to upstream favicon`; `markdownlint clean`.

`markdownlint` needs a config because the repo ships none and the default 80-character limit does not match this
project's 120-character standard. If `/tmp/mdlint.json` is missing, create it first:

```bash
cat > /tmp/mdlint.json <<'JSON'
{"MD013": {"line_length": 120, "code_blocks": false, "tables": false, "headings": false}}
JSON
```

- [ ] **Step 2: Confirm no collateral changes and that the filestash work is untouched**

```bash
cd /data/templates
git status --short
git log --oneline -3
git show --stat --oneline HEAD~1 | head -4
git show --stat --oneline HEAD | head -4
```

Expected: the two new commits each touch exactly one file; `git status --short` still lists the four filestash/
gitignore lines from the "CRITICAL" section above and nothing else.

- [ ] **Step 3: Report**

Summarise: template path, icon path, validator output, the 8/8 spec checklist items with their evidence, and the
two commit hashes. State explicitly that the filestash artifacts remain staged and untouched, because the earlier
chain's review is still unfinished and its files must be committed or excluded before this chain's review step.
Do not claim completion without the Step 1 output in hand.

No commit in this task unless a fix was needed; if one was, use the same pattern:

```bash
git add <exact path>
git commit --only <exact path> -m "Fix official-filebrowser-quantum <what>"
```
