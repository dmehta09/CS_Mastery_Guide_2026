# Project 13: Multi-Region Auth Service

## Difficulty Level: Advanced
**Estimated Time**: 3-4 weeks per language

## What You're Building

A globally distributed authentication service that works across multiple regions (like Auth0, Okta, or AWS Cognito) with JWT, OAuth 2.0, and session management.

## Key Concepts

- JWT (Access & Refresh tokens)
- OAuth 2.0 flows
- Session management
- Password hashing (Argon2, bcrypt)
- Multi-factor authentication
- Geo-replication
- Session affinity
- Token revocation

## Architecture

```
                    [Global Load Balancer]
                    (Route to nearest region)
                             │
         ┌───────────────────┼───────────────────┐
         ▼                   ▼                   ▼
    [US Region]         [EU Region]         [Asia Region]
    - Auth Service      - Auth Service      - Auth Service
    - PostgreSQL        - PostgreSQL        - PostgreSQL
    - Redis             - Redis             - Redis
         │                   │                   │
         └───────────────────┴───────────────────┘
                    [Cross-Region Replication]
```

## JWT Implementation

### Token Structure

```
Header.Payload.Signature

Header: { "alg": "RS256", "typ": "JWT" }
Payload: {
  "sub": "user_123",
  "email": "user@example.com",
  "iat": 1708084800,
  "exp": 1708088400,
  "roles": ["user"]
}
Signature: HMACSHA256(base64(header) + "." + base64(payload), secret)
```

### Generate Token

```typescript
import jwt from "jsonwebtoken";

function generateAccessToken(user: User): string {
  return jwt.sign(
    {
      sub: user.id,
      email: user.email,
      roles: user.roles
    },
    process.env.JWT_SECRET,
    { expiresIn: "15m" }
  );
}

function generateRefreshToken(user: User): string {
  const token = jwt.sign(
    { sub: user.id },
    process.env.REFRESH_SECRET,
    { expiresIn: "30d" }
  );

  // Store in database for revocation
  await db.saveRefreshToken({
    user_id: user.id,
    token_hash: hash(token),
    expires_at: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000)
  });

  return token;
}
```

### Verify Token

```typescript
async function verifyAccessToken(token: string) {
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    return decoded;
  } catch (error) {
    if (error instanceof jwt.TokenExpiredError) {
      throw new UnauthorizedError("Token expired");
    }
    throw new UnauthorizedError("Invalid token");
  }
}
```

## OAuth 2.0 Flow (Authorization Code)

```
1. User clicks "Login with Google"
2. Redirect to:
   https://accounts.google.com/o/oauth2/v2/auth?
     client_id=xxx&
     redirect_uri=https://myapp.com/callback&
     response_type=code&
     scope=email profile

3. User logs in to Google
4. Google redirects back with code:
   https://myapp.com/callback?code=abc123

5. Exchange code for token:
   POST https://oauth2.googleapis.com/token
   {
     "code": "abc123",
     "client_id": "xxx",
     "client_secret": "yyy",
     "redirect_uri": "https://myapp.com/callback",
     "grant_type": "authorization_code"
   }

6. Receive access_token and id_token
7. Verify id_token (JWT)
8. Create/update user in database
9. Generate your own tokens
10. Return to client
```

## Password Hashing

```typescript
import argon2 from "argon2";

// Hash password on registration
async function hashPassword(password: string): Promise<string> {
  return await argon2.hash(password, {
    type: argon2.argon2id,
    memoryCost: 65536, // 64 MB
    timeCost: 3,
    parallelism: 4
  });
}

// Verify password on login
async function verifyPassword(
  hash: string,
  password: string
): Promise<boolean> {
  try {
    return await argon2.verify(hash, password);
  } catch (error) {
    return false;
  }
}
```

## Multi-Factor Authentication (TOTP)

```typescript
import speakeasy from "speakeasy";

// Generate secret for user
function generateMFASecret(user: User) {
  const secret = speakeasy.generateSecret({
    name: `MyApp (${user.email})`,
    issuer: "MyApp"
  });

  // Save secret to database
  await db.saveMFASecret(user.id, secret.base32);

  // Return QR code for user to scan
  return {
    secret: secret.base32,
    qr_code: secret.otpauth_url
  };
}

// Verify TOTP code
function verifyMFACode(secret: string, code: string): boolean {
  return speakeasy.totp.verify({
    secret,
    encoding: "base32",
    token: code,
    window: 1 // Allow 30s drift
  });
}
```

## Token Revocation

```typescript
// Blacklist tokens
const tokenBlacklist = new Set<string>();

async function revokeToken(token: string) {
  const decoded = jwt.decode(token);
  const ttl = decoded.exp - Math.floor(Date.now() / 1000);

  // Store in Redis with TTL
  await redis.set(`blacklist:${token}`, "1", "EX", ttl);
  tokenBlacklist.add(token);
}

async function isTokenRevoked(token: string): Promise<boolean> {
  if (tokenBlacklist.has(token)) return true;

  const exists = await redis.exists(`blacklist:${token}`);
  if (exists) {
    tokenBlacklist.add(token);
    return true;
  }

  return false;
}
```

## API Endpoints

```http
# Register
POST /auth/register
{
  "email": "user@example.com",
  "password": "SecurePassword123!",
  "name": "John Doe"
}

# Login
POST /auth/login
{
  "email": "user@example.com",
  "password": "SecurePassword123!"
}
Response:
{
  "access_token": "eyJhbGc...",
  "refresh_token": "eyJhbGc...",
  "expires_in": 900
}

# Refresh Token
POST /auth/refresh
{
  "refresh_token": "eyJhbGc..."
}

# Logout (Revoke)
POST /auth/logout
Authorization: Bearer eyJhbGc...

# OAuth Login
GET /auth/oauth/google
→ Redirects to Google

GET /auth/oauth/google/callback?code=abc123
→ Returns tokens
```

## Success Criteria

✅ Secure password hashing
✅ JWT generation/validation
✅ OAuth 2.0 flows working
✅ MFA support
✅ Token revocation
✅ Multi-region sync
✅ Sub-100ms auth checks

---

**Next**: Project 14 (Background Job Processor)
