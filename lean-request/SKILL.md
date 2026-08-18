---
name: lean-request
description: Strip HTTP requests (curl) to a programmatically replicable minimum — iteratively test-removing query params, then cookies, then non-essential headers. Use when the user provides a bloated curl command and wants a lean, reproducible equivalent.
---

# Lean Request

Strip a curl command to a programmatically replicable minimum. Most e-commerce API requests work without their complicated query parameters and cookies — this skill finds exactly which parts are essential while preserving the core headers needed for reliable, production-grade requests.

## Goal

**Programmatic replicability** — not "absolute minimum." Our output should produce a curl command that can be reliably used from code. Over-stripping headers to save a few bytes sacrifices reliability and makes requests look like bots. Be pragmatic.

## Core Headers — DO NOT REMOVE

These headers are almost always essential for API endpoints that expect browser-originated requests. Do not waste time testing their removal:

- `accept` — content negotiation; many endpoints 406 or return HTML instead of JSON without it
- `accept-language` — locale/encoding; can affect response content
- `user-agent` — bot detection; removing it risks 403/503 blocks
- `x-requested-with` — AJAX indicator; many endpoints require it to return JSON vs full HTML
- `referer` — origin validation; required by many CSRF-aware endpoints

Only attempt to strip headers that are clearly extraneous (e.g. `sec-ch-*`, `device-memory`, `dnt`, `downlink`, `ect`, `rtt`, `dpr`, `viewport-width`, `priority`, `x-amzn-flow-closure-id`, `sec-fetch-*`).

## Methodology

Work in strict order. Never skip ahead.

### Phase 1: Query Parameters

1. **Baseline first** — Run the original request as-is to confirm it works (200). Write response to a file so you can inspect structure.
2. **Strip one group at a time** — Remove related parameters together (e.g., all tracking params, all display params).
3. **Test after each removal** — If the response changes (different HTTP code, different size, different content), the removed group is essential — put it back.
4. **Keep going until only essential params remain.**

Common removable groups on e-commerce APIs:

- Timestamps (`qid`, `timestamp`, `_`)
- Search/rank positions (`sr`, `ref`, `ref_override`)
- Device info (`deviceOs`, `deviceType`, `viewport-*`)
- Feature flags (`twisterFeatureRefreshEnabled`, `isProductReel`)
- Session IDs in URL (`twsid`, `sessionId`)
- Referer in URL (`originalHttpReferer`)
- Version (`vs`, `v`)
- Product type metadata (`productTypeDefinition`, `productGroupId`, `storeId`)

### Phase 2: Cookies

1. **Keep all original headers + lean URL, strip cookies** — Cookies often have time-sensitive auth tokens and are the biggest source of bloat. Strip them first.
2. **Test with zero cookies** — Many public endpoints need no cookies at all.
3. **If cookies are essential**, remove them one at a time to find the minimum set. Often only `session-id` is needed.
4. **If cookies 404 without headers**, the original headers are part of the auth chain — keep core headers and retry.

### Phase 3: Headers

1. **Keep core headers** — Do NOT attempt to remove `accept`, `accept-language`, `user-agent`, `x-requested-with`, or `referer`.
2. **Strip extraneous headers only** — Remove all `sec-ch-*`, `sec-fetch-*`, client hint headers (`device-memory`, `dnt`, `downlink`, `dpr`, `ect`, `rtt`, `viewport-width`), and other clearly optional headers (`priority`, `x-amzn-flow-closure-id`).
3. **Test the removal.** These should always be safe to remove.

### Phase 4: Report

Output the final lean curl command and a summary like:

```
Removed: 15 query params, 8 cookies, 18 headers
Kept:   asin (essential), core headers, session-id (cookie)
```

## Rules

- **Always test** after every single change. Never assume a parameter is optional.
- **If 503 appears** — rate limiting. Wait 3-5 seconds and retry. If persistent, go back to the last working version.
- **Use `-o /dev/null -w "%{http_code}\n"`** for size-agnostic HTTP status checks. Add `%{size_download}` only after confirming 200.
- **Keep headers and cookies fixed** during Phase 1. Keep URL + headers fixed during Phase 2. Keep URL + cookies fixed during Phase 3.
- **Binary search for critical params** — if stripping a big group breaks, split it in half to isolate.
- **Write responses to files** when structure matters — for `&&&`-delimited or chunked responses, raw byte count can be misleading. Compare segment counts instead.
- **Don't over-strip** — a 3KB header difference on a 1MB response is meaningless. Focus on structural integrity (same segments, same content type), not exact byte matching.
- **Auth cookies expire** — if all tests suddenly start returning 404, the `session-token` or session cookies likely expired. Report what you found and note the cookie dependency.

## Example

Input:

```bash
curl 'https://api.example.com/search?q=shoes&page=1&size=20&sessionId=abc&deviceOs=ios&v=2&_=1720000000' \
  -H 'accept: application/json' \
  -H 'x-api-key: sk-123' \
  -H 'x-device: mobile' \
  -H 'x-session: abc' \
  -b 'tracking=xyz; session=abc; userPref=dark'
```

Result after lean-request:

```bash
curl 'https://api.example.com/search?q=shoes' \
  -H 'accept: application/json' \
  -H 'x-api-key: sk-123' \
  -b 'session=abc'
```
