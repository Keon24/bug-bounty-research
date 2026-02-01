# Target A - Fintech Payment Platform Assessment

## What is Target A?
A fintech payment infrastructure platform discovered through HackerOne. Scope included their web application and REST API, both with a test mode available for safe testing.

## Recon Phase

Started with passive subdomain enumeration using subfinder, amass, and assetfinder. Combined and deduplicated results to get a clean list. Found 28 subdomains total, but only 2 were actually in scope - the main web app and the API endpoint. Just because you find subdomains doesn't mean you can test them. Always check the program's scope first.

## IDOR Testing

### Phase 1: Redirect Parameter Testing

The login page uses a redirect parameter to send users back to where they were trying to go. Naturally, I started testing it.

**Test 1: Internal redirect manipulation**
- Endpoint: `https://[target]/login?redirect=https://[target]/transactions`
- Result: Redirected to login page, forced re-authentication
- Status: ✗ Not vulnerable

**Test 2: External redirect attempts**
- Parameters tested: `?redirect=`, `?next=`, `?url=`, `?return=`, `?redir=`
- Payloads: External URLs (google.com, github.com, etc.)
- Result: All properly validated, no open redirect
- Status: ✗ Not vulnerable

### Phase 2: Google Dorking Reconnaissance

Tried finding hidden endpoints and parameters through Google dorking.

**Dorks attempted:**
- `inurl:= site:[target].com`
- `inurl:id site:[target].com`
- `inurl:user site:[target].com`
- `inurl:account site:[target].com`
- `inurl:api site:[target].com`

**Results:**
- No hidden endpoints discovered
- No additional parameters found

### Phase 3: Referer-Based Open Redirects

Checked if the app redirects based on the HTTP Referer header.

- Method: Looked for endpoints that redirect based on where the request came from
- Result: No referer-based redirects found
- Status: ✗ Not found

### Phase 4: Parameter-Based Open Redirects

Tried identifying all redirect parameters through multiple discovery methods.

**Discovery Methods:**
- Manual browsing and URL observation
- Google dorking (inurl operators)
- Parameter fuzzing attempts

**Parameters discovered:**
- `?redirect=` (login page)

**Parameters NOT found:**
- No additional redirect parameters discovered
- No hidden redirect functionality identified

**Conclusion:** Application uses minimal redirect functionality, properly validated.

### Phase 5: Bypassing Open-Redirect Protection

Basic redirect testing blocked all external URLs. Moved to advanced bypass techniques - these exploit mismatches between how the URL validator and the browser interpret URLs. If the validator and browser disagree on where a URL points, you can trick the app into redirecting offsite.

**Browser Autocorrect Bypasses:**
Browsers autocorrect malformed URLs. If the validator sees a broken URL and rejects it, but the browser fixes it and follows it anyway, that's a bypass.
- `https:[target].com`
- `https;[target].com`
- `https:\/\/[attacker].com`
- `https:/\/[attacker].com`
- ✗ Blocked

**Backslash Manipulation:**
Validators and browsers handle backslashes differently. Some validators treat `\` as a path separator, but browsers convert it to `/`, changing where the URL points.
- `https://[attacker].com\@[target].com`
- ✗ Blocked

**Flawed Validator Logic:**
Many validators just check if the target domain appears anywhere in the URL. These payloads trick that check by including the target domain as a subdomain or path while actually redirecting to an attacker site.
- `https://[target].com.[attacker].com`
- `https://[attacker].com/[target].com`
- `https://[target].com.[attacker].com/[target].com`
- `https://[target].com@[attacker].com/[target].com`
- ✗ Blocked

**Data URL Bypass:**
Data URLs let you embed content directly in a URL using the `data:` scheme. If the validator doesn't block the data: scheme, you can encode a JavaScript redirect inside it.
- `data:text/html;base64,[base64 encoded redirect]`
- ✗ Blocked

**URL Encoding:**
Exploits mismatches in how the validator and browser decode special characters. If the validator only decodes once but the browser decodes multiple times, the real destination stays hidden during validation.
- Single: `https://[target].com%2f@[attacker].com`
- Double: `https://[target].com%252f@[attacker].com`
- Triple: `https://[target].com%25252f@[attacker].com`
- ✗ Blocked

**Non-ASCII Characters:**
Non-ASCII characters like `%ff` can confuse validators. Browsers sometimes convert these into different characters like `?`, which changes the URL structure entirely.
- `https://[attacker].com%ff.[target].com`
- ✗ Blocked

**Unicode/Cyrillic Bypass:**
Cyrillic characters look identical to ASCII but are different characters. Validators might not catch them, but browsers still follow the URL.
- ✗ Blocked

**Slash Look-alike Characters:**
Unicode characters that look like `/` but aren't. Browsers convert them to real slashes, changing where the URL points.
- Uses `%E2%95%B1` and similar look-alike characters
- ✗ Blocked

**Combined Techniques:**
Layered multiple bypass techniques together. Harder for validators to catch when exploits are chained.
- `https://[target].com%252f@[attacker].com/[target].com`
- ✗ Blocked

### Phase 6: Escalation Potential
If an open redirect was found, these are the attack chains it could enable:
- Phishing - send victims to fake login via legitimate-looking URL
- SSRF - bypass URL allowlists by redirecting through the target
- OAuth token theft - leaked via Referer header during redirect
- Bug chaining - combine with other vulnerabilities for higher impact
## Summary

Tested 15+ variations across redirect parameters. No vulnerabilities found. The application has solid input validation on redirect functionality.

**Key findings:**
- All redirect parameters properly validate destination URLs
- External redirects are blocked
- Referer-based redirects not implemented
- Minimal attack surface for redirect-based attacks

## What's Next
- [ ] Set up Burp Suite proxy
- [ ] Capture API requests with IDs
- [ ] Test for IDOR in API endpoints
- [ ] Review API documentation for testable endpoints

## Tools Used
- subfinder, amass, assetfinder
- Google dorking
- Firefox with FoxyProxy