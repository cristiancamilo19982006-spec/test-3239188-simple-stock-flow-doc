# Security Policy

> Security is not a feature — it is a system property built from day one. This document
> defines the mandatory practices.
> Any deviation must be explicitly approved by the Tech Lead.

---

## Security principles

1. **Defense in Depth:** Multiple security layers. If one fails, the others contain the damage.
2. **Least Privilege:** Each component has only the minimum necessary permissions.
3. **Fail Secure:** In case of error, the system denies access, does not allow it.
4. **Security by Design:** Security controls are designed from the start, not added at the end.
5. **Zero Trust:** Always verify, never implicitly trust, even within the internal network.

---

## Authentication

### JWT (JSON Web Tokens)

| Property | Required value |
|----------|---------------|
| Signing algorithm | RS256 (asymmetric) or HS256 with 256+ bit secret |
| Access token expiration | 1 hour (`exp`) |
| Refresh token expiration | 7 days |
| Required claims | `sub` (userId), `iat`, `exp`, `jti` (unique token ID) |
| Client storage | `httpOnly cookie` (web) or Keychain/Keystore (mobile) |

**Prohibited in the payload:**
- Passwords
- Card data
- Full PII (only the user ID)

### Refresh Token

- Stored in the database (with bcrypt hash)
- Mandatory rotation on each use (one refresh token = one use)
- Invalidated on logout and on password change
- ALL active tokens invalidated if use of a revoked token is detected

---

## Authorization

### RBAC (Role-Based Access Control)

| Role | Description | Permissions |
|------|-------------|------------|
| `SUPER_ADMIN` | System technical administrator | All |
| `ADMIN` | Legal firm or organization admin | `users:manage`, `templates:manage`, `reports:export` |
| `LAWYER` | Legal professional creating/editing documents | `documents:create`, `documents:read`, `documents:update` |
| `CITIZEN` | End user requesting PQRs and legal help | `pqr:create`, `pqr:read_own` |
| `GUEST` | Unauthenticated user | `public:read` |

**Permission model:**