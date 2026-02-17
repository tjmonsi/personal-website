## Clarifications Needed

The following item was identified during a consistency review of all spec documents after applying the answers from `clarifications-2026-02-17-0004`.

---

### CLR-049: JWT Issuer Identifier Uses Old `tjmonserrat` Name 🟢

**Context**: The domain was renamed from `tjmonserrat.com` to `tjmonsi.com` per the "Other Notes" in the previous clarifications round. However, the JWT issuer claim (`iss`) in SEC-003A and BE-API-009 still uses `"tjmonserrat-web"` as the static client identifier. This is an application-level identifier (not a domain reference), so it was not changed during the domain rename. It appears in:

- **Spec 03** (BE-API-009): `iss: Static client identifier (e.g., "tjmonserrat-web")`
- **Spec 06** (SEC-003A JWT Claims): `"iss": "tjmonserrat-web"`
- **Spec 06** (SEC-003A Implementation): `A static client ID (e.g., "tjmonserrat-web")`

Additionally, social media example URLs (e.g., `github.com/tjmonserrat`, `@tjmonserrat`) use the `tjmonserrat` username, which is separate from the website domain.

**Question**: Should the JWT issuer be updated to `"tjmonsi-web"` for consistency with the new domain, or should it remain as `"tjmonserrat-web"` since it's an internal identifier?

**Options**:

- **A**: Update to `"tjmonsi-web"` for consistency with the domain rename. *(Recommended)*
- **B**: Keep `"tjmonserrat-web"` — it's an internal identifier and doesn't need to match the domain.
- **C**: Other value.

**Answer**:

