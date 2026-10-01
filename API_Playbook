[OWASP Top Ten API](https://owasp.org/API-Security/editions/2023/en/0x11-t10/)
[API PortSwigger](https://portswigger.net/web-security/api-testing)
[[Access Control]]
[[Server Side Param Pollution SSPP]]
[API1:2023 Broken Object Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/)


[[digital forensics process playbook]]


# 🎯 API & Mobile Attack Playbook

> Bug Bounty Checklist — Obsidian Notes Based on: PortSwigger Web Security Academy + OWASP MAS Tags: #bugbounty #api #mobile #checklist


	You need to remember the path!!! The ONLY reason you know how to do any of the stuff above was from an accumulation of ~"random"~ skills. It was never random at all - it was all correctly placed. Developing apps gave you this experience. Writing code or debugging code gave you insight in the process and why. Forex internship showed you Linux, Python, R, and data analysis. Your IT jobs showed you how to analyze output, handle raw data, and create information from raw data!!! The CCENT studies you started at 18 were huge as well, it laid the groundwork for your understanding of the radiowave world. You are so much further along than you realize.

---

## 📋 Pre-Engagement

- [ ] Read the full scope document carefully
- [ ] Note all **in-scope domains** (wildcards, subdomains, mobile apps)
- [ ] Note all **explicitly out-of-scope** items (third-party SDKs, vendor infra)
- [ ] Check if third-party/vendor findings have a separate disclosure path
- [ ] Confirm rules of engagement (rate limits, account creation allowed, etc.)
- [ ] Set up Burp Suite proxy and confirm traffic is flowing
  > 💡 Burp listens on `127.0.0.1:8080` by default. Install the Burp CA cert on your device (Settings > Proxy > Import/export CA cert) so HTTPS traffic doesn't get blocked before you even start.
- [ ] For mobile: confirm SSL pinning bypass is working before investing time
  > 💡 Run `objection -g com.target.app explore` then `android sslpinning disable`. If Burp starts seeing HTTPS traffic, pinning is bypassed. Don't skip this — without it you're flying blind.

---

## 🔍 API Recon

### Documentation Discovery

- [ ] Check for public API docs (`/api`, `/swagger/index.html`, `/openapi.json`, `/docs`)
- [ ] If you find a resource endpoint like `/api/v1/users/123`, walk up the path:
  > 💡 APIs often leave parent routes exposed. `/api/v1/users/` with no ID may return all users. You're looking for object listings, schema leakage, or unintended access.
    - [ ] `/api/v1/users/`
    - [ ] `/api/v1/`
    - [ ] `/api/`
- [ ] Parse OpenAPI/Swagger JSON/YAML if found — import into Burp or Postman
  > 💡 In Burp: Proxy > HTTP history > right-click request > "Send to Repeater". For Swagger JSON, paste it into Postman (Import > Raw text) to auto-generate a request collection.
- [ ] Check `robots.txt` for hidden paths
- [ ] Check JS files for embedded API endpoints (use JS Link Finder BApp)
  > 💡 JS Link Finder works in Community. Alternatively: Proxy > HTTP history > filter by `.js` > open each file and Ctrl+F for `/api`, `/v1`, `fetch(`, `axios`. Tedious but effective.

### Endpoint Mapping

- [ ] ~~Crawl the app with Burp Scanner~~ ⚠️ **CE: Scanner is Pro only.** Use manual browsing through Burp's embedded browser + ffuf for active discovery.
- [ ] Browse app manually through Burp's browser — trigger all features
  > 💡 The goal is populating Proxy > HTTP history. Click every button, load every screen, trigger every state change. The don doesn't reveal himself unless you make him.
- [ ] Look for URL patterns: `/api/`, `/v1/`, `/internal/`, `/private/`
- [ ] Use Burp Intruder with common wordlists to brute-force hidden endpoints ⚠️ **CE: Intruder is rate-limited (1 req/sec).** Use ffuf instead:
  ```bash
  ffuf -w /usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt \
       -u https://TARGET/api/FUZZ \
       -mc 200,201,204,301,302,401,403 \
       -t 50
  ```
  > 💡 `-mc` filters by status code — 401/403 are goldmines, they confirm the endpoint exists even if you can't access it yet.
- [ ] For each endpoint, note the HTTP method it was observed with
- [ ] Test all other HTTP methods on each endpoint (GET, POST, PUT, PATCH, DELETE, OPTIONS)
  > 💡 In Burp Repeater: right-click the request > "Change request method", or manually edit the verb. A GET that returns 403 might accept a POST with no auth check — method-based ACL bypass is real.
- [ ] Use OPTIONS request to see what methods the server advertises
  > 💡 In Repeater, change method to OPTIONS, send. The `Allow:` response header lists what the server claims to support. Cross-reference against what you've already observed — gaps are interesting.

### Content Type Testing

- [ ] Identify what content type the endpoint expects (JSON, XML, form-data)
- [ ] Try switching `Content-Type` between JSON and XML — observe behavior
  > 💡 Some parsers handle the two differently, with different validation logic or error paths. A JSON endpoint that silently accepts XML may have a totally different (weaker) code path handling it.
- [ ] Use Content Type Converter BApp to automate format switching
  > 💡 Works in Community Edition. Install via Extensions > BApp Store.

---

## 🔓 Access Control Testing

### Vertical Privilege Escalation

- [ ] Check if admin/sensitive paths are accessible without auth (wordlist brute force)
  > 💡 ffuf with a focused admin wordlist: `common.txt`, `admin-panels.txt` from SecLists. A 200 response with no auth header = instant finding.
- [ ] Check `robots.txt` for disclosed sensitive paths
- [ ] Inspect JS source for hardcoded admin URLs or role-conditional logic
  > 💡 Browser DevTools > Sources tab > Ctrl+Shift+F to search all files. Search for `admin`, `role`, `isAdmin`, `superuser`. jadx does the same for APKs: File > Save All Sources, then `grep -r "admin" ./sources`.
- [ ] Look for role/admin flags in cookies, hidden fields, or query params:
    - [ ] `?admin=true`
    - [ ] `?role=1`
    - [ ] Modify these and observe if access changes
- [ ] Try `X-Original-URL` and `X-Rewrite-URL` headers to bypass platform-level ACLs:
  > 💡 Some reverse proxies (nginx, Cloudflare) enforce ACLs on the URL they see, but pass a different path to the backend via these headers. You send `POST /` (which is allowed) but the backend processes `/admin/deleteUser`. Add in Repeater manually.
    ```
    POST / HTTP/1.1
    X-Original-URL: /admin/deleteUser
    ```
- [ ] Try HTTP method switching on restricted endpoints (POST → GET, etc.)
- [ ] Try URL case variation (`/ADMIN/deleteUser`) on ACL-enforced paths
  > 💡 ACL rules are often case-sensitive string matches on the path. The backend may normalize to lowercase after the check, letting `/ADMIN/` slip through.
- [ ] Try trailing slash bypass (`/admin/deleteUser/`)
- [ ] If Spring app: try adding arbitrary file extension (`/admin/deleteUser.json`)
  > 💡 Spring MVC historically matched routes ignoring file extensions. The ACL rule checks `/admin/deleteUser` exactly — `.json` suffix makes it look different to the rule, identical to the route handler.

### Horizontal Privilege Escalation / IDOR

- [ ] Identify any numeric or sequential IDs in requests (`id=123`, `user_id=456`)
- [ ] Increment/decrement IDs and check if you get another user's data
  > 💡 In Repeater, right-click the ID value > "Send to Intruder" (CE: use ffuf with `-w` of a number range). You're looking for 200 responses with different user data. That's BOLA — the whole game.
- [ ] Look for GUIDs — even if not guessable, search the app for leaked GUIDs:
    - [ ] User messages, reviews, receipts, shared links
  > 💡 GUIDs aren't secret, they're just long. If the app shows you another user's GUID anywhere (shared link, notification, receipt), that's your key — plug it into the ID field.
- [ ] Check redirect responses — sensitive data may be in the body even on 302
  > 💡 Burp shows the full response including body even for redirects. Don't just check status codes — a 302 can carry the full user object in its body before redirecting to login.
- [ ] Test IDOR on **static files** (e.g., `/static/12144.txt` → try `12143.txt`)
- [ ] Target admin users via IDOR — if their account page leaks creds or allows password change → horizontal becomes vertical

### Multi-Step Process Bypass

- [ ] Map out any multi-step flows (checkout, account update, admin confirm)
- [ ] Try skipping directly to the final step with crafted parameters
  > 💡 Steps often have their own endpoints. If step 3 is `POST /admin/confirm`, hit it directly without going through steps 1-2. Each step may check auth independently — or it may just trust you got there.
- [ ] Check if each step independently validates auth — don't assume they all do

### Referer-Based Access Control

- [ ] Identify sub-pages that may check only the `Referer` header
  > 💡 Some apps check `Referer: https://target.com/admin` to verify you navigated from the right place rather than checking actual auth. It's a header you can freely forge.
- [ ] Forge `Referer: https://target.com/admin` on direct requests to sub-pages
- [ ] Test sensitive actions gated behind parent pages (e.g., `/admin/deleteUser`)

---

## 💉 Server-Side Parameter Pollution (SSPP)

[[Server Side Param Pollution SSPP]]
May need to turn cache off in DevTools in browser and also reset the Filter in Burp to SHOW ALL

> 💡 **Disable cache:** Browser DevTools (F12) > Network tab > check "Disable cache" checkbox. Keeps stale responses from masking your injections.
> 💡 **Reset Burp filter:** Proxy > HTTP history > click the Filter bar at the top > set "Show All" to avoid missing requests filtered by content type or MIME.

### Query String

- [ ] Inject `%23` (encoded `#`) to try truncating the server-side query:
    ```
    /search?name=test%23foo
    ```
  > 💡 The `#` becomes a fragment delimiter server-side, cutting off anything after it. If a hidden required parameter (like `&role=user`) lives after your input, you've just removed it from the internal call.
    - [ ] If server returns valid result → truncation worked, safety params may be bypassed
    - [ ] If "Invalid name" error → `foo` was treated as part of input, no truncation
- [ ] Inject `%26` (encoded `&`) to append extra parameters:
    ```
    /search?name=test%26foo=bar
    ```
    - [ ] Observe if response changes — unknown param may be silently accepted
- [ ] Inject valid hidden parameters discovered during recon:
    ```
    /search?name=test%26email=attacker@evil.com
    ```
- [ ] Override existing parameters by duplicating them:
    ```
    /search?name=test%26name=administrator
    ```
    - [ ] PHP → last value wins
    - [ ] ASP.NET → values combined (`test,administrator`)
    - [ ] Node/Express → first value wins

### SSPP via Password Reset — Account Takeover

> **What's happening:** The forgot-password endpoint passes your username to an internal API. That internal API has hidden parameters (like `field`) that control _what_ gets returned. By injecting `&field=reset_token` into your username input, you can force the API to return the reset token directly in the HTTP response instead of emailing it — letting you take over any account.

- [ ] Trigger a password reset for your own account — capture the `POST /forgot-password` request in Burp
- [ ] Send to Repeater. Confirm baseline response is consistent
- [ ] Test with invalid username (e.g., `administratorx`) — confirm you get `Invalid username` error (proves username validation is working, good baseline)
- [ ] Inject `%26x=y` after the username to probe for SSPP:
    ```
    username=administrator%26x=y
    ```
    - [ ] `Parameter is not supported` → internal API parsed `&x=y` as a separate param ✅ SSPP confirmed
- [ ] Inject `%23` to truncate the server-side query:
    ```
    username=administrator%23
    ```
    - [ ] `Field not specified` → a required `field` param was cut off by your `#` ✅ hidden param discovered
- [ ] Inject `&field=x#` to probe the field parameter:
    ```
    username=administrator%26field=x%23
    ```
    - [ ] `Invalid field` → the internal API recognizes `field` as a valid parameter name ✅
- [ ] Brute-force valid `field` values using **ffuf**:
    ```bash
    ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
         -u 'https://TARGET/forgot-password' \
         -X POST \
         -d 'username=administrator%26field=FUZZ%23' \
         -H 'Content-Type: application/x-www-form-urlencoded' \
         -mc 200 -fw <baseline_word_count>
    ```
  > 💡 Get `<baseline_word_count>` from your known-bad baseline response (the "Invalid field" one). ffuf filters out responses matching that word count, leaving only anomalies.
    - [ ] Look for `username` and `email` returning 200 — confirms those are valid field values
- [ ] Check JS files in Proxy HTTP history for the reset endpoint structure:
    - [ ] Look for patterns like `/forgot-password?reset_token=${resetToken}`
    - [ ] This tells you the internal token parameter name to target
- [ ] Change field to `reset_token`:
    ```
    username=administrator%26field=reset_token%23
    ```
    - [ ] Reset token returned directly in the API response ✅ Account takeover possible
- [ ] Navigate to `/forgot-password?reset_token=<token>` in browser
- [ ] Set new password and log in as the target user

**Impact:** Full account takeover without access to the victim's email. Critical severity.

### REST Path

- [ ] Identify endpoints where input maps to a REST path segment
- [ ] Inject path traversal via `%2f..%2f`:
    ```
    /edit?name=peter%2f..%2fadmin
    ```
  > 💡 If the backend constructs a path from your input (e.g., `/users/peter`), injecting `%2f..%2f` (URL-encoded `/../`) may walk up to `/users/../admin` → `/admin`. You're path-traversing within the URL routing layer, not the filesystem.
    - [ ] Check if normalized path resolves to privileged resource

### Structured Data (JSON/XML)

- [ ] If input lands inside server-side JSON, try breaking out of string context:
    ```
    name=peter","access_level":"administrator
    ```
  > 💡 If the server builds JSON by string concatenation (bad practice but common), your input closes the string and injects a new key. Check if the response or behavior reflects `access_level: administrator`.
- [ ] If client sends JSON, try escaping quotes:
    ```json
    {"name": "peter\",\"access_level\":\"administrator"}
    ```
- [ ] Check API responses for injection — stored data re-embedded in JSON responses
- [ ] Try XML equivalents if the API accepts XML

---

## 🔑 Mass Assignment

- [ ] Find a GET endpoint that returns a full user/object response
- [ ] Compare returned fields against what the PATCH/PUT endpoint accepts
  > 💡 The GET response is the source of truth for what fields exist on the object. The PATCH form only shows you what the app *wants* you to send — the binding may accept everything the object has.
- [ ] Look for fields in the GET response not present in the update form:
    - [ ] `isAdmin`, `role`, `access_level`, `verified`, `balance`
- [ ] Send a PATCH with an **invalid** value for the hidden field — observe if behavior changes (signals binding)
  > 💡 e.g., `"isAdmin": "notaboolean"`. If you get a type validation error vs. a "unknown field" error, the field is bound — the app knows about it and is processing it.
- [ ] Send a PATCH with `"isAdmin": true` (or equivalent) and check for privilege change
- [ ] Browse app after modification attempt to confirm whether privileges changed

---

## 📱 Mobile-Specific (Android)

### Setup

- [ ] SSL pinning bypassed (objection, Frida, or custom hook)
  > 💡 `adb shell` → confirm device is connected. Then: `objection -g com.target.app explore` → `android sslpinning disable`. If pinning survives objection, try a Frida script (`ssl-pinning-bypass.js` from https://codeshare.frida.re).
- [ ] All app traffic confirmed flowing through Burp
  > 💡 Set Wi-Fi proxy to `192.168.x.x:8080` (your machine's LAN IP). Browse to `http://burp` in the device browser to download and install Burp's CA cert. Then confirm HTTPS requests appear in Proxy > HTTP history.
- [ ] APK decompiled (jadx or apktool) for static analysis
  > 💡 `jadx -d output/ target.apk` — gives you readable Java. `apktool d target.apk` — gives you smali + resources. Use jadx for reading logic, apktool for patching/recompiling.

### Static Analysis

- [ ] Search decompiled code for hardcoded API keys, secrets, tokens
  > 💡 `grep -r "api_key\|secret\|token\|password\|Bearer" ./output/sources/` — cast a wide net first, then tighten.
- [ ] Look for base URLs, internal endpoints, staging/dev URLs
  > 💡 `grep -r "http\|https" ./output/sources/ | grep -v "schema\|xmlns\|google"` filters out XML boilerplate and leaves actual URLs.
- [ ] Check `AndroidManifest.xml` for exported activities/services/receivers
  > 💡 `android:exported="true"` with no permission check = accessible from other apps (including your own test app). Exported activities can sometimes be started directly to bypass login flows.
- [ ] Check for backup enabled (`android:allowBackup="true"`)
  > 💡 If true, `adb backup -apk com.target.app` may extract the app's data directory — tokens, cached credentials, local databases.
- [ ] Review `res/` and assets for config files, API docs, credentials

### Traffic Analysis

- [ ] Capture baseline traffic for all app features
- [ ] Identify all API endpoints the app calls
- [ ] Note authentication mechanism (Bearer token, API key, cookie, etc.)
- [ ] Check if tokens are long-lived or rotated
  > 💡 Use the app, log out, log back in — does the token change? Long-lived tokens with no expiry are a separate finding if you can leak one.
- [ ] Look for sensitive data in request/response headers
- [ ] Check if requests include device fingerprint or GUID that could be an IDOR vector
  > 💡 A device GUID that maps to a user account is just an IDOR with extra steps. Try substituting another user's GUID if you can find one (shared links, referral codes, etc.).

### API Testing (from mobile context)

- [ ] Run all above API/access control/SSPP checks against mobile endpoints
- [ ] Check if mobile endpoints have **weaker** auth than web equivalents
  > 💡 Mobile APIs are sometimes treated as "trusted" and skipped on auth checks. Hit the same resource at `/api/mobile/v1/` vs `/api/v1/` and compare — the accountant may have a back door.
- [ ] Try accessing mobile API endpoints directly from Burp (no app) — does auth still enforce?
- [ ] Check for different API versions (`/v1/` vs `/v2/`) — older versions may lack controls
- [ ] Test unauthenticated access to endpoints after removing auth header entirely
  > 💡 In Repeater, delete the `Authorization:` header line completely. A 200 back means no auth check — that's a critical finding.
- [ ] Check SDK/third-party endpoints — confirm in/out of scope before testing

---

## 🛠️ Tools Reference

| Task | Tool | CE Alternative |
|---|---|---|
| Proxy / intercept | Burp Suite | — |
| Active scanning | ~~Burp Scanner~~ (Pro only) | nikto, nuclei |
| SSL pinning bypass (Android) | objection / Frida | — |
| APK decompile | jadx / apktool | — |
| Find hidden params | Param Miner BApp | ffuf + param wordlist |
| Find hidden endpoints | ~~Burp Intruder~~ (rate-limited in CE) | **ffuf** |
| Fuzzing | **ffuf** (open source, CLI) | — |
| JS endpoint extraction | JS Link Finder BApp | grep on saved JS files |
| Content type switching | Content Type Converter BApp (works in CE) | manual in Repeater |
| SSPP detection | Backslash Powered Scanner BApp | manual in Repeater |
| OpenAPI parsing | OpenAPI Parser BApp (works in CE) | Postman import |
| Manual API testing | Burp Repeater / Postman | — |
| Android traffic | Burp + NetHunter on Pixel 6a | — |
| Recon wordlists | SecLists (`/usr/share/seclists/`) | — |

> ⚠️ **Burp Community Limitations:** No active scanner, Intruder throttled to 1 req/sec. Replace Intruder with ffuf everywhere. BApps (Param Miner, JS Link Finder, Content Type Converter, OpenAPI Parser, Backslash Powered Scanner) mostly work in CE — check each one's BApp Store listing to confirm.

---

## 📝 Finding Documentation Template

```
Title:
Endpoint:
Method:
Parameter:
Payload used:
Expected behavior:
Observed behavior:
Impact:
CVSS estimate:
In scope? (check program policy):
Evidence (screenshots/requests):
```

---

## ✅ Before Submitting a Report

- [ ] Confirmed endpoint is in-scope (no third-party/vendor exclusions)
- [ ] Impact is clearly demonstrated (not theoretical)
- [ ] Steps to reproduce are complete and reproducible
- [ ] Checked for duplicate signals (similar finding already reported?)
- [ ] Severity is accurate — don't over/undersell
- [ ] Sensitive data (other users' PII) handled responsibly — not retained
