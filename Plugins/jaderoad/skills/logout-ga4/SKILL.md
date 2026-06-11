---
name: logout-ga4
description: Use when the user wants to log out, sign out, or disconnect Claude Code from Google Analytics 4. Revokes the locally stored OAuth credentials and optionally guides the user through global token revocation in their Google account. Trigger phrases include "log out of GA4", "disconnect GA4", "remove GA4 access", "revoke GA4 credentials", "sign out of Google Analytics", "delete GA4 auth".
version: 1.0.0
---

# Logout from GA4

Revoke the OAuth Application Default Credentials used for GA4 access and (optionally) remove the refresh token globally from the user's Google account. Reversible only by re-running `/{ORG_ID}:login-ga4`.

**This is a destructive action.** It deletes a local secret file and (in the global mode) terminates a refresh token that cannot be recovered — only re-issued. Always confirm with the user before executing the revocation.

`gcloud` is NOT required — this skill talks to Google's revoke endpoint directly via Python.

## Usage

```
/{ORG_ID}:logout-ga4
```

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

### Step 1: Detect current credential state

```bash
python3 - <<'PY'
import json, os, sys, urllib.request
p = os.path.expanduser("~/.config/gcloud/application_default_credentials.json")
if not os.path.exists(p):
    print("STATE: NO_CREDENTIAL"); sys.exit(0)
try:
    from google.oauth2.credentials import Credentials
    from google.auth.transport.requests import Request
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
    print("Email:   ", info.get("email", "(none)"))
    print("Scope:   ", info.get("scope", "(none)"))
    print("Expires: ", info.get("expires_in", "?"), "seconds")
except Exception as e:
    print("STATE: CREDENTIAL_FILE_PRESENT_BUT_INVALID —", e)
PY
```

Report what was found:

| State | What to say to the user |
|-------|------------------------|
| `NO_CREDENTIAL` | "No GA4 credentials are currently configured on this machine. Nothing to log out of." Stop. |
| `CREDENTIAL_FILE_PRESENT_BUT_INVALID` | "An ADC file exists but the token cannot be refreshed. The credential is effectively dead already. I can clean up the file — confirm? (yes/no)" |
| `ACTIVE` | Show the `Email`, `Scope`, and `Expires` so the user can confirm this is the credential they want to revoke. |

### Step 2: Confirm revocation intent (mandatory)

This is a destructive action. Do NOT proceed without explicit confirmation. Ask:

```
Two revocation levels:

(a) LOCAL only — Deletes ~/.config/gcloud/application_default_credentials.json.
    The refresh token remains valid at Google. If you re-run /{ORG_ID}:login-ga4
    on this machine, you may not need to re-consent.

(b) LOCAL + GLOBAL — Deletes the local file AND revokes the refresh token at
    Google. This is the secure default if you suspect the token may have been
    exposed or if you're permanently rotating credentials.

Which? (a / b / cancel)
```

If the user says `cancel`, stop and report no change made.

### Step 3: Execute revocation

**For option (a) — LOCAL only:**

```bash
rm -f "$HOME/.config/gcloud/application_default_credentials.json"
[ ! -f "$HOME/.config/gcloud/application_default_credentials.json" ] && echo "LOCAL: removed" || echo "LOCAL: STILL PRESENT — manual cleanup needed"
```

**For option (b) — LOCAL + GLOBAL:**

```bash
python3 - <<'PY'
import json, os, sys, urllib.request, urllib.parse
p = os.path.expanduser("~/.config/gcloud/application_default_credentials.json")
if not os.path.exists(p):
    print("LOCAL: already absent"); sys.exit(0)
d = json.load(open(p))
token = d.get("refresh_token") or d.get("access_token")
if not token:
    print("WARN: no token found in ADC file — skipping global revoke")
else:
    try:
        body = urllib.parse.urlencode({"token": token}).encode()
        req = urllib.request.Request(
            "https://oauth2.googleapis.com/revoke",
            data=body,
            headers={"Content-Type": "application/x-www-form-urlencoded"},
        )
        urllib.request.urlopen(req, timeout=10).read()
        print("GLOBAL: revoke endpoint accepted token")
    except urllib.error.HTTPError as e:
        msg = e.read().decode(errors="replace")
        print(f"GLOBAL: revoke endpoint returned {e.code}: {msg}")
        print("       (already-revoked tokens return 400 — that's fine)")
    except Exception as e:
        print("GLOBAL: revoke call failed —", e)

os.remove(p)
print("LOCAL: removed")
PY
```

### Step 4: Verify revocation succeeded

```bash
ADC_PATH="$HOME/.config/gcloud/application_default_credentials.json"
[ ! -f "$ADC_PATH" ] && echo "FILE: removed" || echo "FILE: STILL PRESENT — revocation incomplete"
```

If anything reports `STILL PRESENT`, surface that to the user — do not silently swallow the failure.

### Step 5: Point to global account-level verification (always)

Tell the user where to verify and further revoke at the account level:

```
Verify global revocation in your Google account:

  https://myaccount.google.com/permissions

Look for the OAuth app you authorized (e.g., "Claude GA4"). If present after option (a),
the refresh token is still live at Google and option (b) (or manual revocation here)
is needed to fully sever access.

Recent API activity audit:

  https://myaccount.google.com/notifications
```

### Step 6: Report

```
GA4 access revoked.

  Local file:    <removed | still present>
  Global token:  <revoked | still live — manual action recommended>
  Account:       <email — for reference>

To re-authenticate: /{ORG_ID}:login-ga4
```

If anything failed or partial:
- Show what specifically failed (do not paper over)
- Tell the user the manual cleanup step
- Recommend they verify in https://myaccount.google.com/permissions before trusting the revocation

---

## Notes for the agent

- This skill is destructive (deletes a local credential, optionally terminates a refresh token). Per the project's destructive-action policy, **never proceed without explicit user confirmation** at Step 2.
- Do NOT log or echo the access token or refresh token to chat or to files. The metadata (email, scope, expiry) is fine to display so the user can confirm what they're revoking.
- This skill does NOT delete the OAuth client JSON (`~/.config/google/oauth-client-ga4.json`) — that's the user's own OAuth app, not a credential.
- After successful revocation, the user can immediately re-run `/{ORG_ID}:login-ga4` to set up fresh credentials if desired.
