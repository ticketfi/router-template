# Auth Configuration for Railway Production

This document outlines the authentication configuration needed for the Apollo Router in production Railway deployment.

## Environment Variables Required

### Apollo Router (Railway)

```bash
# Apollo GraphOS
APOLLO_KEY=your-apollo-key-here
APOLLO_GRAPH_REF=your-graph-ref-here

# Optional: Enable MCP Server for AI assistance
MCP_ENABLE=0  # Set to 1 to enable
```

### Backend Auth Service

The auth backend service needs these environment variables:

```bash
# JWT Configuration
JWT_SECRET=your-secure-jwt-secret
JWT_ACCESS_EXPIRATION=15m
JWT_REFRESH_EXPIRATION=7d

# Cookie Configuration
COOKIE_DOMAIN=.ticketfi.ai     # .ticketfi.dev for dev, .ticketfi.net for staging
COOKIE_SECURE=true             # Must be true for HTTPS
COOKIE_SAMESITE=none           # Required for cross-domain cookies

# Database
DATABASE_URL=postgresql://...

# CORS Origins (comma-separated)
CORS_ORIGINS=https://auth.ticketfi.ai,https://organizer.ticketfi.ai,https://manager.ticketfi.ai,https://promoter.ticketfi.ai,https://venue.ticketfi.ai,https://talent.ticketfi.ai,https://attendee.ticketfi.ai
```

### Frontend Auth App (Vercel/Railway)

```bash
# Auth API Endpoint
NEXT_PUBLIC_AUTH_API_ENDPOINT=https://auth-api.ticketfi.ai  # .dev for dev, .net for staging

# Frontend URLs
NEXT_PUBLIC_AUTH_DOMAIN=https://auth.ticketfi.ai
NEXT_PUBLIC_HOSTNAME=ticketfi.ai
NEXT_PUBLIC_APP_URL=https://dashboard.ticketfi.ai

# GraphQL Endpoint (for non-auth queries)
NEXT_PUBLIC_GRAPHQL_ENDPOINT=https://api.ticketfi.ai/graphql
```

## Domain Architecture

### Production (.ai)

- **WWW:** `https://ticketfi.ai`
- **Auth Frontend:** `https://auth.ticketfi.ai` - Sign-in/sign-up UI
- **Auth API:** `https://auth-api.ticketfi.ai` - REST auth endpoints
- **GraphQL Router:** `https://api.ticketfi.ai` - Apollo Router
- **Apps:** `https://{app}.ticketfi.ai` - organizer, manager, etc.

### Staging (.net)

- **WWW:** `https://ticketfi.net`
- **Auth Frontend:** `https://auth.ticketfi.net`
- **Auth API:** `https://auth-api.ticketfi.net`
- **GraphQL Router:** `https://api.ticketfi.net`
- **Apps:** `https://{app}.ticketfi.net`

### Development (.dev)

- **WWW:** `https://ticketfi.dev`
- **Auth Frontend:** `https://auth.ticketfi.dev`
- **Auth API:** `https://auth-api.ticketfi.dev`
- **GraphQL Router:** `https://api.ticketfi.dev`
- **Apps:** `https://{app}.ticketfi.dev`

## CORS Configuration

The `router.yaml` includes all necessary origins for auth:

```yaml
cors:
  policies:
    - origins:
        # Production domains (.ai)
        - https://ticketfi.ai
        - https://auth.ticketfi.ai
        - https://auth-api.ticketfi.ai # ✅ Auth REST API
        - https://dashboard.ticketfi.ai
        - https://organizer.ticketfi.ai
        - https://manager.ticketfi.ai
        - https://promoter.ticketfi.ai
        - https://venue.ticketfi.ai
        - https://talent.ticketfi.ai
        - https://attendee.ticketfi.ai
        - https://api.ticketfi.ai
      allow_credentials: true # ✅ Required for cookies
      allow_headers:
        - content-type
        - authorization
        - cookie # ✅ Required for auth
      expose_headers:
        - set-cookie # ✅ Required for auth
```

## Cookie Flow

### Sign In Flow

1. User visits `https://organizer.ticketfi.ai`
2. Middleware detects no auth, redirects to `https://auth.ticketfi.ai/sign-in?returnUrl=...`
3. User enters credentials
4. Frontend calls `POST https://auth-api.ticketfi.ai/signin` with `credentials: 'include'`
5. Backend sets httpOnly cookies with domain `.ticketfi.ai`
6. Frontend redirects to `returnUrl` (organizer app)
7. Organizer app reads cookies and validates JWT

### Cookie Settings (Backend)

```typescript
res.cookie('access_token', token, {
  httpOnly: true, // ✅ Can't be accessed by JavaScript
  secure: true, // ✅ HTTPS only
  sameSite: 'none', // ✅ Allow cross-domain
  domain: '.ticketfi.ai', // ✅ Shared across subdomains (note the dot)
  maxAge: 15 * 60 * 1000, // 15 minutes
});

res.cookie('refresh_token', refreshToken, {
  httpOnly: true,
  secure: true,
  sameSite: 'none',
  domain: '.ticketfi.ai',
  maxAge: 7 * 24 * 60 * 60 * 1000, // 7 days
});
```

## Backend REST Endpoints

Auth backend service must expose these REST endpoints:

### POST /signin

Authenticates user and sets httpOnly cookies.

**Request:**

```json
{
  "email": "user@example.com",
  "password": "password123"
}
```

**Response:**

```json
{
  "success": true,
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "fullName": "John Doe",
    "primaryPersona": "ORGANIZER",
    "availablePersonas": ["ORGANIZER", "MANAGER"]
  },
  "sessionId": "session-uuid"
}
```

**Cookies Set:**

- `access_token` (httpOnly, 15min)
- `refresh_token` (httpOnly, 7d)
- `session_id` (httpOnly, session)

### POST /signup

Registers new user and sets httpOnly cookies.

**Request:**

```json
{
  "email": "user@example.com",
  "password": "password123",
  "fullName": "John Doe"
}
```

**Response:** Same as `/signin`

### POST /signout

Clears httpOnly cookies.

**Response:**

```json
{
  "success": true
}
```

**Cookies Cleared:**

- `access_token`
- `refresh_token`
- `session_id`

### GET /verify

Verifies authentication status via cookies.

**Response:**

```json
{
  "authenticated": true,
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "fullName": "John Doe",
    "primaryPersona": "ORGANIZER",
    "availablePersonas": ["ORGANIZER", "MANAGER"]
  }
}
```

## Railway Services Configuration

### Service 1: Apollo Router

- **Name:** `ticketfi-router`
- **Port:** 4000
- **Domain:** `api.ticketfi.ai`
- **Environment:** Production
- **Config:** Uses `router.yaml` from this repo

### Service 2: Auth Backend

- **Name:** `ticketfi-auth-api`
- **Port:** 4001 (internal)
- **Domain:** `auth-api.ticketfi.ai`
- **Environment:** Node.js/NestJS
- **Endpoints:** `/signin`, `/signup`, `/signout`, `/verify`

### Service 3: Auth Frontend

- **Name:** `ticketfi-auth`
- **Port:** 3000 (internal)
- **Domain:** `auth.ticketfi.ai`
- **Environment:** Next.js
- **Deployment:** Vercel (preferred) or Railway

## Security Checklist

Production deployment must have:

- [ ] `COOKIE_SECURE=true` - HTTPS only
- [ ] `COOKIE_DOMAIN=.ticketfi.ai` - Shared across subdomains (note the dot!)
- [ ] `COOKIE_SAMESITE=none` - Allow cross-domain
- [ ] `allow_credentials: true` in CORS - Required for cookies
- [ ] JWT_SECRET is strong and secure (32+ characters)
- [ ] All auth endpoints use HTTPS (not HTTP)
- [ ] CORS origins are explicitly listed (no wildcards)
- [ ] Database connections use SSL
- [ ] Secrets stored in Railway/Vercel secrets (not hardcoded)

## Testing Production Auth

### 1. Test Cookie Setting

```bash
curl -X POST https://auth-api.ticketfi.ai/signin \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password"}' \
  -v
```

Look for `Set-Cookie` headers in response.

### 2. Test Cookie Reading

```bash
curl https://auth-api.ticketfi.ai/verify \
  -H "Cookie: access_token=..." \
  -v
```

Should return authenticated user.

### 3. Test Cross-Domain

1. Sign in at `https://auth.ticketfi.ai/sign-in`
2. Navigate to `https://organizer.ticketfi.ai`
3. Should be authenticated (no redirect to sign-in)

## Troubleshooting

### Cookies Not Being Set

- Check `COOKIE_SECURE=true` is set
- Verify domain has HTTPS (not HTTP)
- Check `Set-Cookie` header has `Domain=.ticketfi.ai` (note the dot)
- Ensure `SameSite=None; Secure` is present

### Cookies Not Being Sent

- Check frontend uses `credentials: 'include'` in fetch
- Verify CORS has `allow_credentials: true`
- Check cookie domain matches request domain
- Ensure cookies haven't expired

### 401 Unauthorized

- Check JWT_SECRET matches between services
- Verify access_token hasn't expired
- Check JWT middleware is validating correctly
- Ensure Bearer token format if using Authorization header

## Migration from GraphQL to REST

Auth endpoints were migrated from GraphQL to REST for better cookie support:

- **Before:** Apollo Client mutations with unreliable cookie handling
- **After:** Direct fetch calls with explicit `credentials: 'include'`

See `/frontend/docs/AUTH_REST_API_MIGRATION.md` for full migration details.

## Environment-Specific Notes

### Development (.dev)

- Use `COOKIE_DOMAIN=.ticketfi.dev`
- Doppler config: `dev`
- Can use less strict security for testing

### Staging (.net)

- Use `COOKIE_DOMAIN=.ticketfi.net`
- Doppler config: `stg`
- Should mirror production security

### Production (.ai)

- Use `COOKIE_DOMAIN=.ticketfi.ai`
- Doppler config: `prd`
- All security settings must be production-grade

## References

- [Apollo Router CORS Documentation](https://www.apollographql.com/docs/router/configuration/cors)
- [HTTP Cookie Security](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies)
- [SameSite Cookie Attribute](https://web.dev/samesite-cookies-explained/)
- [Railway Documentation](https://docs.railway.app/)

---

**Last Updated:** October 6, 2025
**Status:** ✅ Production Ready
**Environment:** Railway + Vercel
