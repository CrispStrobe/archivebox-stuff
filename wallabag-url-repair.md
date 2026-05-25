# Repairing Broken URLs in Wallabag

When saving articles via browser extensions or mobile apps, Wallabag sometimes captures the wrong page — typically a redirect wrapper, login gate, or CAPTCHA page instead of the actual article. This guide documents common broken URL patterns and how to fix them.

## The Problem

Several services wrap shared URLs in redirect layers:

| Source | URL Pattern | What Wallabag Saves |
|---|---|---|
| LinkedIn | `linkedin.com/safety/go/?url=...` | LinkedIn's language-selector interstitial (2.7 KB, title "LinkedIn") |
| LinkedIn | `linkedin.com/uas/login?session_redirect=...` | Login page (useless) |
| archive.ph | `archive.ph/HASH` | CAPTCHA page (744 bytes, title "archive.ph") |

Wallabag doesn't follow these redirects because they require JavaScript execution or CAPTCHA solving, so it saves the wrapper page instead of the article.

## Detecting Broken Entries

Use the Wallabag API to find entries with suspiciously small content from known wrapper domains:

```bash
# Get OAuth token
TOKEN=$(curl -s -X POST "$WALLABAG_URL/oauth/v2/token" \
  -H "Content-Type: application/json" \
  -d "{
    \"grant_type\": \"password\",
    \"client_id\": \"$CLIENT_ID\",
    \"client_secret\": \"$CLIENT_SECRET\",
    \"username\": \"$USERNAME\",
    \"password\": \"$PASSWORD\"
  }" | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

# Find LinkedIn wrapper entries
curl -s -H "Authorization: Bearer $TOKEN" \
  "$WALLABAG_URL/api/entries.json?domain_name=www.linkedin.com&perPage=100" \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
for e in data.get('_embedded', {}).get('items', []):
    if 'safety/go' in e.get('url', ''):
        print(f'ID {e[\"id\"]}: {e[\"url\"][:80]}')
"
```

## Fixing LinkedIn Redirect URLs

LinkedIn's `safety/go` URLs encode the real destination in the `url` query parameter:

```
https://www.linkedin.com/safety/go/?url=https%3A%2F%2Fwww.example.com%2Farticle&urlhash=...
                                        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                                        Real URL (URL-encoded)
```

**Fix approach:**
1. Extract the real URL from the `url=` parameter
2. Delete the broken Wallabag entry via API
3. Create a new entry with the real URL (Wallabag will fetch it properly)

Note: Wallabag's API does not allow changing the URL of an existing entry via PATCH — the URL is immutable once saved. You must delete and re-create.

```python
import urllib.parse

def unwrap_linkedin_url(wrapper_url):
    """Extract the real URL from a LinkedIn safety/go wrapper."""
    parsed = urllib.parse.urlparse(wrapper_url)
    params = urllib.parse.parse_qs(parsed.query)
    return params.get("url", [None])[0]

# Example
real = unwrap_linkedin_url(
    "https://www.linkedin.com/safety/go/?url=https%3A%2F%2Fwww.zeit.de%2Farticle&urlhash=abc"
)
# Returns: "https://www.zeit.de/article"
```

## Preventing Future LinkedIn Issues

Two layers of defense:

### 1. Sync script unwrapping

Add URL unwrapping to any Wallabag → ArchiveBox sync script. In the URL extraction step, before adding to the archive queue:

```python
import re
from urllib.parse import unquote

# Unwrap LinkedIn redirect URLs
if "linkedin.com/safety/go" in entry_url:
    m = re.search(r'[?&]url=([^&]+)', entry_url)
    if m:
        entry_url = unquote(m.group(1))

# Skip login-gated pages entirely
if "linkedin.com/uas/login" in entry_url:
    continue
```

This ensures ArchiveBox archives the real URL, but Wallabag still has the broken entry.

### 2. Daily cleanup cron

Schedule the LinkedIn fix script to run daily and repair any new wrapper entries in Wallabag itself:

```bash
# In crontab (run after daily backup)
0 4 * * * python3 /path/to/fix_linkedin_urls.py >> /path/to/linkedin_fix.log 2>&1
```

The script uses `--dry-run` for safe previewing. It queries Wallabag for `domain_name=www.linkedin.com`, finds `safety/go` wrappers, extracts the real URL, deletes the broken entry, and re-adds with the correct URL. Wallabag then fetches the actual article content.

This doesn't fix the entry in Wallabag itself (it still stores the wrapper URL), but ensures ArchiveBox archives the correct page.

## Fixing archive.ph Entries

archive.ph blocks server-side fetching with a reCAPTCHA challenge. Wallabag gets back a CAPTCHA page instead of the archived article.

**Options:**
- **Save from your browser**: Use the Wallabag browser extension on your local machine (residential IP, no CAPTCHA) to save archive.ph pages — the extension sends the rendered page content directly
- **CAPTCHA solving service**: Use a service like 2captcha to programmatically solve the reCAPTCHA, then fetch the real content and push it into Wallabag via the API with the `content` field
- **Accept it**: The content lives on archive.ph and can always be read there manually

## Other Redirect Wrappers

The same pattern applies to other URL shorteners and redirect services. Common ones to watch for:

| Service | Pattern | Extractable? |
|---|---|---|
| LinkedIn safety | `linkedin.com/safety/go/?url=` | Yes — `url` param |
| Google redirect | `google.com/url?q=` | Yes — `q` param |
| Facebook redirect | `l.facebook.com/l.php?u=` | Yes — `u` param |
| Twitter/X shortlinks | `t.co/...` | Need to follow redirect |
| LinkedIn shortlinks | `lnkd.in/...` | Need to follow redirect |

For query-parameter redirects, extract the param. For shortlink services, you need to follow the HTTP redirect chain (which Wallabag usually handles correctly — it's only the JavaScript-based redirects that break).
