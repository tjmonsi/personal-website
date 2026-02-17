````markdown
# Clarifications Batch 0013

**Date**: 2026-02-17
**Scope**: CLR-103 through CLR-104
**Status**: Awaiting answers

---

## CLR-103: `navigator.sendBeacon()` Incompatible with JWT `Authorization` Header on `POST /t` 🟡

**Files affected**: `specs/02-frontend-specifications.md` (FE-COMP-004), `specs/03-backend-api-specifications.md` (BE-API-009), `specs/06-security-specifications.md` (SEC-003A)

**Description**: FE-COMP-004 specifies two delivery mechanisms for `POST /t` tracking events:

- **Link clicks**: "fire-and-forget via `navigator.sendBeacon()` or a non-blocking `fetch`"
- **Time-on-page milestones**: "THE SYSTEM SHALL use `navigator.sendBeacon()` as the preferred method for sending milestone events, as it is reliable even during page unload."

However, BE-API-009 and SEC-003A require every `POST /t` request to include an `Authorization: Bearer <JWT>` header. **`navigator.sendBeacon()` does not support custom HTTP headers** — it can only set `Content-Type` (via `Blob`), not `Authorization`. These two requirements are mutually exclusive for the same request.

**Option A (Recommended)**: Replace `navigator.sendBeacon()` references with `fetch()` using `keepalive: true`, which supports custom headers and is also reliable during page unload. Update FE-COMP-004 to specify `fetch({ method: 'POST', keepalive: true, headers: { 'Authorization': 'Bearer ...' } })` as the delivery mechanism for all `POST /t` calls (link clicks, time-on-page milestones, error reports).

**Option B**: Allow `POST /t` to accept the JWT in the request body (e.g., a `token` field) instead of the `Authorization` header, enabling `sendBeacon()` with a JSON `Blob`. This requires changes to BE-API-009 and SEC-003A validation logic.

**Option C**: Use `fetch()` with `keepalive: true` as the primary mechanism (with `Authorization` header), and fall back to `sendBeacon()` without authentication only when `fetch` with `keepalive` is unavailable. Document the fallback behavior and accept that unauthenticated `sendBeacon` requests will be rejected by the backend (graceful data loss for very old browsers).

**Answer**:

---

## CLR-104: Frontend CSP `img-src` May Block External Images in Rendered Markdown Articles 🟢

**Files affected**: `specs/06-security-specifications.md` (SEC-005), `specs/02-frontend-specifications.md` (FE-PAGE-003, FE-PAGE-005)

**Description**: The frontend Content-Security-Policy in SEC-005 defines:
```
img-src 'self' data:
```

Articles are fetched as `text/markdown` and rendered client-side (FE-PAGE-003 for technical articles, FE-PAGE-005 for other articles). If any article's markdown body contains external image references (e.g., `![diagram](https://example.com/image.png)`), those images would be blocked by the CSP.

This may be intentional (articles are text-only) or an oversight.

**Option A (Recommended)**: Document the constraint explicitly — articles SHALL NOT contain external image references. Images in articles are limited to inline `data:` URIs or images hosted on `tjmonsi.com`. This is consistent with a minimal, text-focused personal website.

**Option B**: Broaden `img-src` to allow external HTTPS images: `img-src 'self' data: https:`. This permits markdown articles to reference any external image over HTTPS.

**Option C**: Broaden `img-src` to a specific allowlisted domain (e.g., a Cloud Storage bucket or GitHub raw content): `img-src 'self' data: https://storage.googleapis.com`. This is more restrictive than Option B while allowing hosted images.

**Answer**:

---

## Summary

| CLR | Severity | Category | Brief Description |
|-----|----------|----------|-------------------|
| CLR-103 | 🟡 IMPORTANT | API Incompatibility | `navigator.sendBeacon()` cannot send JWT `Authorization` header required by `POST /t` |
| CLR-104 | 🟢 SUGGESTION | Security / Content | Frontend CSP `img-src` may block external images in rendered markdown articles |

````
