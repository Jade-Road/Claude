---
name: login-ga4
description: Use when the user wants to authenticate Claude Code with Google Analytics 4 for read-only data access. Sets up OAuth credentials limited to the `analytics.readonly` scope only — no other Google services. Trigger phrases include "log in to GA4", "authenticate GA4", "set up GA4 access", "connect to Google Analytics", "GA4 auth", "give Claude GA4 access".
version: 1.0.0
---

# Login to GA4 (read-only)

Authenticate Claude Code with the user's Google account for read-only access to Google Analytics 4 properties. Writes an Application Default Credentials (ADC) file scoped to `analytics.readonly` and nothing else.

**Why this is safe by design:**

OAuth scopes are an absolute ceiling — the credentials cannot do anything outside the scopes granted at consent time, regardless of what the underlying Google account is permitted to do. With `analytics.readonly`, Claude can read GA4 data and absolutely nothing else: no GA4 modification, no Drive/Gmail/Calendar, no GCP resources, no Google Ads, no spend-eligible API.

**Why this skill does NOT use `gcloud auth application-default login`:**

Current `gcloud` versions reject any `--scopes` value that omits `https://www.googleapis.com/auth/cloud-platform` (error: *"cloud-platform scope is required but not requested"*). That defeats the narrow-scope goal of this skill. Instead, this skill runs the OAuth flow directly via Python's `google-auth-oauthlib` and writes the credential to the same ADC location and format that gcloud would have.

## Usage

```
/{ORG_ID}:login-ga4
```

## Prerequisites

- **Python 3.8+** on the user's machine
- **`google-auth-oauthlib`** Python package (the skill installs it if missing)
- A **Google account** that has GA4 access (any level — Viewer is sufficient) on at least one property the user wants to read
- An **OAuth 2.0 client of type "Desktop app"** belonging to the user (the skill walks the user through creating one if absent). `gcloud` is NOT required.

## Instructions

### Bootstrap: Load Org Configuration

**Run this before any other step.** Read the plugin configuration to bind org variables for this skill:

```bash
cat "{SKILL_BASE_DIR}/../../.claude-plugin/plugin.json"
```

Extract and bind:
- `{ORG_ID}` ← `name` field (e.g., `jaderoad`, `provaxus`, `acme`)
- `{ORG_NAME}` ← `org_name` field (e.g., `JadeRoad`, `Provaxus`, `Acme Corp`)
- `{GITHUB_ORG}` ← `github_org` field (e.g., `JadeRoad-AI`, `Provaxus-AI`, `Acme-Corp`)
- `{MARKETING_AGENT}` ← `marketing_agent` field if present; otherwise `{ORG_ID}-marketing-sme`

All `{ORG_ID}`, `{ORG_NAME}`, `{GITHUB_ORG}`, and `{MARKETING_AGENT}` tokens throughout this skill resolve to these values. The skill body contains no hard-coded organization names.

---

### Step 1: Verify Python + OAuth library

```bash
python3 -c "import sys; print(sys.version)" 2>&1 | head -1
python3 -c "import google_auth_oauthlib" 2>&1 && echo "oauthlib: OK" || echo "oauthlib: MISSING"
```

If Python 3 is missing, stop and tell the user to install it (`brew install python3` on macOS; package manager elsewhere) and re-run.

If `google_auth_oauthlib` is missing, install idempotently:

```bash
pip3 install --user --quiet google-auth-oauthlib 2>&1 || pip install --user --quiet google-auth-oauthlib
```

Re-check the import. If it still fails, surface the install error verbatim and stop.

### Step 2: Detect existing ADC credentials

```bash
ADC_PATH="$HOME/.config/gcloud/application_default_credentials.json"
if [ -f "$ADC_PATH" ]; then
  echo "Existing ADC found at $ADC_PATH"
  python3 - <<'PY'
import json, os, sys
try:
    d = json.load(open(os.path.expanduser("~/.config/gcloud/application_default_credentials.json")))
    print("Type:", d.get("type", "unknown"))
    print("Client ID:", d.get("client_id", "unknown"))
except Exception as e:
    print("Could not parse ADC file:", e)
PY
fi
```

Check whether the existing credential can mint a token, and what scopes it holds:

```bash
python3 - <<'PY'
import json, os, sys
try:
    from google.oauth2.credentials import Credentials
    from google.auth.transport.requests import Request
    import urllib.request
    p = os.path.expanduser("~/.config/gcloud/application_default_credentials.json")
    if not os.path.exists(p):
        print("STATE: NO_CREDENTIAL"); sys.exit(0)
    d = json.load(open(p))
    creds = Credentials(
        token=None,
        refresh_token=d.get("refresh_token"),
        client_id=d.get("client_id"),
        client_secret=d.get("client_secret"),
        token_uri="https://oauth2.googleapis.com/token",
    )
    creds.refresh(Request())
    info = json.loads(urllib.request.urlopen(
        f"https://oauth2.googleapis.com/tokeninfo?access_token={creds.token}"
    ).read().decode())
    print("STATE: ACTIVE")
    print("Email:", info.get("email", "(none)"))
    print("Scope:", info.get("scope", "(none)"))
    print("Expires in (s):", info.get("expires_in", "?"))
except Exception as e:
    print("STATE: INVALID_OR_EXPIRED —", e)
PY
```

Map the result:

| Existing scope | Action |
|----------------|--------|
| Contains `analytics.readonly` and nothing else beyond `openid email profile` | Tell user: "GA4 read-only credentials already in place. Nothing to do." Skip remaining steps. |
| Contains broader scopes (e.g., `cloud-platform`, multiple services) | **Surface this as a concern** — ask: "Existing credentials have broader scopes than `analytics.readonly`. Replace with narrow scope? (yes/no)" |
| Missing `analytics.readonly`, or `INVALID_OR_EXPIRED` | Ask: "Existing credentials don't include GA4 access (or can't refresh). Replace? (yes/no)" |

If user declines replacement, stop.

### Step 3: Locate or create the OAuth client JSON

Look for an existing client file:

```bash
for p in \
  "$HOME/.config/google/oauth-client-ga4.json" \
  "$HOME/.config/google/oauth-client-ga4-claude.json" \
  "$HOME/.config/google/client_secret_ga4.json"
do
  [ -f "$p" ] && echo "FOUND: $p" && python3 -c "import json; d=json.load(open('$p')); print('  keys:', list(d.keys())); k=list(d.keys())[0]; print('  redirect_uris:', d[k].get('redirect_uris'))" && break
done
```

Also scan `~/Downloads` for a recently-downloaded `client_secret_*.json`:

```bash
ls -t "$HOME/Downloads"/client_secret_*.json 2>/dev/null | head -3
```

**If a file is found:**

- Confirm with the user and verify it's `installed` type (Desktop app)

**If no file is found**, walk the user through creating one:

```
You need a per-user OAuth client (Desktop app) before continuing. One-time setup, ~3 minutes:

  1. Open https://console.cloud.google.com/apis/credentials
     (select or create a Google Cloud project — no billing required for analytics.readonly)

  2. Configure the OAuth consent screen:
       - User type: External
       - App name: "Claude GA4" (or anything memorable)
       - User support email + Developer email: your own email
       - Test users: add your own Google account

  3. Credentials → Create Credentials → OAuth client ID
       - Application type: Desktop app
       - Name: "Claude GA4 Desktop"
       - Create → DOWNLOAD JSON

  4. Move the downloaded file:
       mkdir -p ~/.config/google
       mv ~/Downloads/client_secret_*.json ~/.config/google/oauth-client-ga4.json
       chmod 600 ~/.config/google/oauth-client-ga4.json

When done, tell me and I'll continue.
```

### Step 4: Run the OAuth flow

Present this to the user (substitute `<CLIENT_JSON>` with the actual path):

```
Run this in your next prompt — type the `!` literally so the command executes in this
session and a browser tab opens locally for you to approve consent:

! python3 <<'PY'
import json, os, sys
from google_auth_oauthlib.flow import InstalledAppFlow

CLIENT = os.path.expanduser("<CLIENT_JSON>")
OUT    = os.path.expanduser("~/.config/gcloud/application_default_credentials.json")
SCOPES = ["https://www.googleapis.com/auth/analytics.readonly"]

flow = InstalledAppFlow.from_client_secrets_file(CLIENT, scopes=SCOPES)
try:
    creds = flow.run_local_server(port=0, open_browser=True)
except Exception:
    creds = flow.run_console()

with open(CLIENT) as f:
    c = json.load(f)["installed"]
adc = {
    "type": "authorized_user",
    "client_id": c["client_id"],
    "client_secret": c["client_secret"],
    "refresh_token": creds.refresh_token,
}
os.makedirs(os.path.dirname(OUT), exist_ok=True)
with open(OUT, "w") as f:
    json.dump(adc, f)
os.chmod(OUT, 0o600)
print("OK — ADC written to", OUT)
print("Granted scopes:", " ".join(creds.scopes))
PY
```

### Step 5: Verify the credential

After the user reports the OAuth flow completed, verify:

```bash
python3 - <<'PY'
import json, os, sys, urllib.request
from google.oauth2.credentials import Credentials
from google.auth.transport.requests import Request

p = os.path.expanduser("~/.config/gcloud/application_default_credentials.json")
if not os.path.exists(p):
    print("FAIL: ADC file missing"); sys.exit(1)
d = json.load(open(p))
creds = Credentials(
    token=None,
    refresh_token=d.get("refresh_token"),
    client_id=d.get("client_id"),
    client_secret=d.get("client_secret"),
    token_uri="https://oauth2.googleapis.com/token",
)
creds.refresh(Request())
info = json.loads(urllib.request.urlopen(
    f"https://oauth2.googleapis.com/tokeninfo?access_token={creds.token}"
).read().decode())
print("Email:    ", info.get("email", "(none)"))
print("Scope:    ", info.get("scope", "(none)"))
print("Expires:  ", info.get("expires_in", "?"), "seconds")
PY
```

Verify `Scope` contains `analytics.readonly` and ideally nothing else beyond `openid email profile`. If broader scopes were granted, flag this.

### Step 6: Permission and rotation hygiene

```bash
chmod 600 "$HOME/.config/gcloud/application_default_credentials.json" 2>/dev/null
```

Tell the user:
1. **Credential lives at** `~/.config/gcloud/application_default_credentials.json`
2. **Refresh tokens expire** if unused for 6 months (Google policy)
3. **Revocation:** `/{ORG_ID}:logout-ga4` or https://myaccount.google.com/permissions
4. **Audit trail:** https://myaccount.google.com/notifications

### Step 7: Smoke test (optional)

If the user provides a GA4 Property ID, run a quick read to confirm the pipeline works. Ask: "Want to run a smoke test against a specific GA4 property? If yes, paste the Property ID."

### Step 8: Report

```
GA4 read-only access configured.

  Account:       <email from tokeninfo>
  Scope:         analytics.readonly (read-only; cannot modify GA4 or access anything else)
  OAuth client:  <path to client JSON>
  Credential:    ~/.config/gcloud/application_default_credentials.json
  Status:        Active (token auto-refreshes)

  Smoke test:    <result if performed, or "skipped">

To revoke: /{ORG_ID}:logout-ga4
To audit:  https://myaccount.google.com/notifications
```

---

## Notes for the agent

- This skill produces no persistent files in the project. State lives entirely in `~/.config/gcloud/` (ADC) and `~/.config/google/` (per-user OAuth client).
- Do NOT save the access token, refresh token, OAuth client secret, or any credential content to memory files, project files, or chat output. Only output metadata (email, scope, status).
- **Do not fall back to `gcloud auth application-default login`** even if gcloud is present. gcloud's ADC login refuses to omit `cloud-platform`, which violates this skill's narrow-scope contract.
