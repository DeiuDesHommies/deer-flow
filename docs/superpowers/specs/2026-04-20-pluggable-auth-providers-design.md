# Pluggable Auth Providers with Request-Level Hook

**Date**: 2026-04-20
**Status**: Draft for external RFC publication
**Target upstream**: `bytedance/deer-flow` `release/2.0-rc`
**Reference local counterpart**: `TrustedHeaderAuthProvider`

---

## 1. Motivation

DeerFlow 2.0-rc already ships a solid authentication foundation:

- `AuthProvider` abstraction for credential-based auth
- `AuthMiddleware` for request gating
- `runtime.user_context` for request-scoped owner isolation

However, one common production deployment mode is still missing a first-class extension point:

> A trusted reverse proxy or upstream gateway authenticates the user first, then forwards the user identity to DeerFlow via request headers.

Today, DeerFlow's `AuthProvider` only supports:

- `authenticate(credentials)`
- `get_user(user_id)`

There is no request-level hook such as `authenticate_request(request)`. This makes header-based SSO / gateway-auth integration awkward, forcing deployers to patch middleware instead of extending the provider system.

This RFC proposes a small, backwards-compatible extension:

- add an optional request-level hook to `AuthProvider`
- provide a `TrustedHeaderAuthProvider` reference implementation
- wire `AuthMiddleware` to try the request-level hook before cookie-based auth

---

## 2. Goals

1. Add a request-level authentication hook without breaking existing local/JWT flows.
2. Support reverse-proxy / gateway-injected identity in a safe, explicit way.
3. Keep the default behavior unchanged for all existing deployments.
4. Reuse the current `AuthProvider` abstraction instead of inventing a parallel plugin system.

## 3. Non-Goals

1. Replace the current JWT/cookie auth flow.
2. Add a full enterprise SSO framework.
3. Standardize all possible upstream headers used by every company.
4. Solve authorization policy beyond current `AuthMiddleware` + `authz.py` responsibilities.

---

## 4. Proposal

### 4.1 Extend `AuthProvider`

Add an optional hook:

```python
async def authenticate_request(self, request: Request) -> User | None:
    return None
```

Semantics:

- return `User` → request is authenticated by this provider
- return `None` → provider declines, middleware falls back to existing auth path
- raise exception → middleware treats it as auth failure and does not silently downgrade

Default implementation returns `None`, preserving current behavior.

### 4.2 `TrustedHeaderAuthProvider`

Reference implementation for deployments where an upstream trusted component injects user identity.

Configuration:

- `trusted_networks`: CIDR allowlist checked on startup and on each request
- `user_id_header`: primary trusted header (e.g. `X-Forwarded-User`)
- `legacy_user_id_headers`: optional compatibility headers for migration windows
- optional HMAC validation may be added by deployers that need stronger trust guarantees

Request flow:

1. verify client IP is inside `trusted_networks`
2. read `user_id_header`
3. if absent, optionally parse `legacy_user_id_headers`
4. resolve user via repository
5. return `User | None`

### 4.3 AuthMiddleware wiring

`AuthMiddleware` should execute:

1. public-path bypass
2. `provider.authenticate_request(request)`
3. if that returns `None`, continue current cookie/JWT path
4. if that returns `User`, stamp `request.state.user`, `request.state.auth`, and `runtime.user_context`

---

## 5. Reference Implementation

The current local branch contains a bridge implementation with the following characteristics:

- trusted CIDR validation via `ipaddress.ip_network`
- primary header: `X-Forwarded-User`
- compatibility parsing for legacy JSON header payloads
- user resolution via the existing auth repository
- middleware branch that prefers trusted-header auth before cookie auth

This fork implementation is intended as the reference candidate for upstreaming, with internal brand-specific details removed.

---

## 6. Backward Compatibility

This proposal is intentionally additive.

- Existing providers do not need to change.
- Existing deployments continue using cookie/JWT auth.
- Middleware behavior is unchanged unless a provider overrides `authenticate_request`.

Therefore, the expected compatibility risk is low.

---

## 7. Security Considerations

1. **Trusted headers are only safe behind a trusted proxy boundary.**
2. `trusted_networks` must fail fast on invalid CIDRs.
3. Requests from untrusted IPs must never be allowed to authenticate via headers.
4. If the request-level hook raises, middleware must not silently fall back in a way that weakens security.
5. Compatibility headers should be explicitly temporary and documented as migration-only.

---

## 8. Alternatives Considered

### A. Patch `AuthMiddleware` directly in every downstream branch

Rejected because it duplicates auth logic across downstream branches and bypasses the provider abstraction.

### B. Add a separate `AuthMiddlewarePlugin`

Rejected because it introduces another abstraction layer when `AuthProvider` already represents authentication extension points.

### C. Keep only cookie/JWT auth

Rejected because production reverse-proxy deployments are common and currently require invasive downstream patches.

---

## 9. Open Questions

1. Should HMAC verification be part of the initial upstream proposal or remain deployment-specific?
2. Should multiple request-level providers be chained, or is one active provider enough?
3. Should the request-level hook live on `AuthProvider` or a smaller sub-protocol?
4. Should official docs include a reverse-proxy deployment guide in the same PR?

---

## 10. Recommended Upstream Sequence

1. RFC issue publication
2. PR #1: `AuthProvider.authenticate_request`
3. PR #2: `TrustedHeaderAuthProvider`
4. PR #3: `AuthMiddleware` wiring + concurrency/context cleanup tests
