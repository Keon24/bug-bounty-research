# Target B - Food Delivery Platform IDOR Assessment

## Overview
This one caught my attention because of the multi-role setup — consumer, driver, and merchant all on the same platform. Anytime you have multiple user types interacting with the same data, there's a good chance authorization isn't enforced consistently across all of them. That was the angle I went in with.

Program runs on HackerOne, paid bounty. Scope covers the web app and API. No automated scanners allowed so everything here was done manually through Burp.

---

## Recon

Nothing crazy here. Ran subfinder and assetfinder against the target, combined the lists, deduplicated with sort -u, then piped everything through httpx-toolkit to see what was actually alive.

```bash
subfinder -d [target] -o subs.txt
assetfinder --subs-only [target] >> subs.txt
sort -u subs.txt -o subs.txt
cat subs.txt | httpx-toolkit -sc -title -td -o alive.txt
```

Most subdomains came back with WAF responses — Cloudflare blocking direct access. The main web app and API were the only things worth focusing on. Went straight to manual testing from there.

---

## Account Setup

The program actually makes this easy — they support HackerOne plus-addressing so you can spin up multiple test accounts without needing separate emails.

```
Account 1: theappsecdev+1@wearehackerone.com
Account 2: theappsecdev+2@wearehackerone.com
```

Having two accounts was the whole point here. You can't properly test IDOR without two separate identities — otherwise you're just looking at your own data and calling it a finding.

Added the required research header to every request in Burp Match and Replace:
```
X-Bug-Bounty: theappsecdev
```

---

## Testing

### Cart Checkout IDOR

First interesting thing I found was cart IDs showing up in checkout URLs. Each cart had a UUID tied to it, and the URL looked like this:

```
GET /consumer/checkout/?order_cart_id=<account-1-cart-id>
```

Naturally the first thing I did was grab Account 1's cart ID, swap in Account 2's session cookie, and fire it off in Repeater.

```
Account 1 cart → Account 2 session → 200 OK
```

Page loaded and showed the cart contents — restaurant, items, subtotal. On the surface that looked like a confirmed IDOR. But digging deeper the 200 response was just Next.js rendering the frontend page. The actual cart data was being pulled by a background API call after the page loaded — I never confirmed the vulnerability at the API layer.

Marked it inconclusive. Would need to find and test that background API call directly to confirm.
- Status: ⚠ Inconclusive

---

### Consumer Profile IDOR

Found a profile endpoint while browsing:

```
GET /consumer/profile/<numeric-id>
```

Account 1 ID and Account 2 ID were sequential. Swapped Account 1's session into a request for Account 2's profile:

```
Account 1 session → Account 2 profile ID → 200 OK
```

Returned Account 2's data. But after looking at what actually came back — display name and contribution count — it was clear this was a public profile. No PII, no order history, nothing sensitive. This is by design, same as any review platform where profiles are public.

Not a finding.
- Status: ✗ Not vulnerable — public by design

---

### Consumer ID Parameter IDOR

This one had potential. Found a numeric consumer ID getting passed as a query parameter:

```
GET /unified-gateway/v1/get-latest-eligible-order?consumer_id=<id>
```

Tried incrementing and decrementing the ID:

```
consumer_id=<id-1>   → {"order_uuid": null}
consumer_id=<id+1>   → {"order_uuid": null}
consumer_id=<id+100> → {"order_uuid": null}
```

Everything came back null. My own ID returned real data so the endpoint was working — it was just validating on the backend before returning anything. Null response instead of an error is actually a sign of decent security practice here — no data leakage even when the auth check fails.
- Status: ✗ Not vulnerable

---

### Reservation IDOR

Created reservations on both accounts, grabbed both UUIDs from Burp history. Classic cross-account test:

**Account 2 session → Account 1 reservation:**
```
→ {"code":"RESERVATION_NOT_FOUND"}
```

**Account 1 session → Account 2 reservation:**
```
→ {"code":"RESERVATION_NOT_FOUND"}
```

Both directions blocked. What I liked about this was the consistent error — same response whether the reservation doesn't exist or belongs to someone else. Prevents enumeration. Properly done.
- Status: ✗ Not vulnerable

---

### GraphQL Store IDs

Found a GraphQL endpoint for store lookups with store IDs in the variables:

```json
{"storeIds": ["<store-id>"], "searchUseCase": "NearbyMap"}
```

Tried adjacent IDs, wildcards, arrays:

```
storeIds: ["<id-1>"] → empty
storeIds: ["<id+1>"] → empty
storeIds: ["*"]      → empty
```

Nothing came back. GraphQL endpoints that just return structured query results aren't usually a great XSS or IDOR surface — the server's using those values to look stuff up, not reflecting them back.
- Status: ✗ Not vulnerable

---

## Summary

Five IDOR vectors tested. Authorization was solid across the board. The cart checkout is the one I'd come back to — the frontend rendered another account's cart which is suspicious even if I couldn't confirm it at the API level. Still worth digging into.

The reservation endpoint stood out as a good example of proper implementation — same error for nonexistent vs unauthorized, no enumeration possible.

**What's next:**
- [ ] Find the background API call behind the checkout page and test directly
- [ ] Sign up as a driver and test cross-role access between consumer and driver endpoints
- [ ] Look at order history and payment method endpoints

## Tools
- subfinder, assetfinder, httpx-toolkit
- Burp Suite Community Edition v2026.2.3
- Chrome with FoxyProxy