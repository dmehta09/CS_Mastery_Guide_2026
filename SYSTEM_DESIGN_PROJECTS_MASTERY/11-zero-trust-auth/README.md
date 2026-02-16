# Project 11: Zero-Trust Authentication System

## Difficulty: Advanced
**Estimated Time**: 3-4 weeks

## Problem Statement

Design a zero-trust authentication and authorization system with mTLS (mutual TLS), OAuth 2.0, JWT, session management, and service-to-service authentication for microservices.

**Zero-Trust Principle**: "Never trust, always verify" - Verify every request, even within internal network.

## Functional Requirements

1. **User Authentication**: Login with username/password, OAuth, SSO
2. **Token Management**: Generate, validate, refresh JWT tokens
3. **Service-to-Service Auth**: mTLS for microservices communication
4. **Authorization**: Role-based access control (RBAC)
5. **Session Management**: Track active sessions, revoke tokens
6. **Multi-Factor Authentication**: SMS, TOTP, WebAuthn

## Architecture

```
                    Client
                      │
                      ▼
            ┌──────────────────┐
            │  Auth Gateway    │
            │  - JWT validation│
            │  - Rate limiting │
            └────────┬─────────┘
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
    ┌────────┐  ┌────────┐  ┌────────┐
    │ User   │  │Product │  │Order   │
    │Service │  │Service │  │Service │
    └────┬───┘  └───┬────┘  └───┬────┘
         │          │            │
         └──────────┴────────────┘
              mTLS encryption
```

## JWT (JSON Web Token)

### Structure

```
Header.Payload.Signature

Header: {"alg": "RS256", "typ": "JWT"}
Payload: {
  "sub": "user-123",
  "email": "user@example.com",
  "roles": ["user", "admin"],
  "iat": 1708084800,  # Issued at
  "exp": 1708088400   # Expires at
}
Signature: RSASHA256(base64(header) + "." + base64(payload), privateKey)
```

### Implementation

```typescript
import jwt from 'jsonwebtoken';

class JWTService {
  private privateKey: string;
  private publicKey: string;

  generateAccessToken(user: User): string {
    return jwt.sign(
      {
        sub: user.id,
        email: user.email,
        roles: user.roles
      },
      this.privateKey,
      {
        algorithm: 'RS256',
        expiresIn: '15m',  // Short-lived
        issuer: 'auth.example.com'
      }
    );
  }

  generateRefreshToken(user: User): string {
    const token = jwt.sign(
      { sub: user.id },
      this.privateKey,
      {
        algorithm: 'RS256',
        expiresIn: '30d',  // Long-lived
        issuer: 'auth.example.com'
      }
    );

    // Store in database for revocation
    await db.saveRefreshToken({
      userId: user.id,
      tokenHash: sha256(token),
      expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000)
    });

    return token;
  }

  verifyToken(token: string): TokenPayload {
    try {
      return jwt.verify(token, this.publicKey, {
        algorithms: ['RS256'],
        issuer: 'auth.example.com'
      });
    } catch (error) {
      if (error instanceof jwt.TokenExpiredError) {
        throw new AuthError('TOKEN_EXPIRED');
      }
      throw new AuthError('INVALID_TOKEN');
    }
  }
}
```

## OAuth 2.0 Flow

### Authorization Code Flow

```
1. User → App: Click "Login with Google"
2. App → Google: Redirect to authorization page
   https://accounts.google.com/o/oauth2/auth?
     client_id=xxx&
     redirect_uri=https://myapp.com/callback&
     response_type=code&
     scope=email profile

3. User → Google: Login & consent
4. Google → App: Redirect with code
   https://myapp.com/callback?code=abc123

5. App → Google: Exchange code for tokens
   POST https://oauth2.googleapis.com/token
   {
     "code": "abc123",
     "client_id": "xxx",
     "client_secret": "yyy",
     "redirect_uri": "https://myapp.com/callback",
     "grant_type": "authorization_code"
   }

6. Google → App: Return tokens
   {
     "access_token": "eyJhbGc...",
     "refresh_token": "1//0gH...",
     "id_token": "eyJhbGc...",
     "expires_in": 3600
   }

7. App: Verify id_token (JWT)
8. App: Create user or update profile
9. App: Create session, return cookies
```

## mTLS (Mutual TLS)

### Concept

```
Traditional TLS:
  Client verifies server certificate
  Server doesn't verify client

mTLS:
  Client verifies server certificate
  Server verifies client certificate
  ✅ Both parties authenticated
```

### Implementation

```typescript
import https from 'https';
import fs from 'fs';

// Server with mTLS
const server = https.createServer({
  cert: fs.readFileSync('server-cert.pem'),
  key: fs.readFileSync('server-key.pem'),
  ca: fs.readFileSync('ca-cert.pem'),  // Certificate authority
  requestCert: true,                    // Require client cert
  rejectUnauthorized: true              // Reject invalid certs
}, (req, res) => {
  const clientCert = req.socket.getPeerCertificate();

  if (!clientCert || !clientCert.subject) {
    res.writeHead(401);
    res.end('Unauthorized');
    return;
  }

  // Extract service name from certificate
  const serviceName = clientCert.subject.CN;
  console.log(`Request from service: ${serviceName}`);

  res.writeHead(200);
  res.end('OK');
});

// Client with mTLS
const options = {
  hostname: 'api.example.com',
  port: 443,
  path: '/api/data',
  method: 'GET',
  cert: fs.readFileSync('client-cert.pem'),
  key: fs.readFileSync('client-key.pem'),
  ca: fs.readFileSync('ca-cert.pem')
};

https.request(options, (res) => {
  // Handle response
});
```

## Role-Based Access Control (RBAC)

```typescript
enum Permission {
  USER_READ = 'user:read',
  USER_WRITE = 'user:write',
  ADMIN_READ = 'admin:read',
  ADMIN_WRITE = 'admin:write'
}

const roles = {
  user: [Permission.USER_READ, Permission.USER_WRITE],
  admin: [Permission.USER_READ, Permission.USER_WRITE, Permission.ADMIN_READ, Permission.ADMIN_WRITE]
};

function authorize(userRoles: string[], requiredPermission: Permission): boolean {
  const userPermissions = userRoles.flatMap(role => roles[role] || []);
  return userPermissions.includes(requiredPermission);
}

// Express middleware
function requirePermission(permission: Permission) {
  return (req, res, next) => {
    if (!authorize(req.user.roles, permission)) {
      return res.status(403).json({ error: 'Forbidden' });
    }
    next();
  };
}

// Usage
app.delete('/api/users/:id',
  requirePermission(Permission.ADMIN_WRITE),
  deleteUser
);
```

## Multi-Factor Authentication (MFA)

### TOTP (Time-based One-Time Password)

```typescript
import speakeasy from 'speakeasy';
import QRCode from 'qrcode';

class MFAService {
  async setupMFA(userId: string): Promise<{ secret: string; qrCode: string }> {
    const secret = speakeasy.generateSecret({
      name: `MyApp (${userId})`,
      issuer: 'MyApp'
    });

    // Generate QR code for user to scan
    const qrCode = await QRCode.toDataURL(secret.otpauth_url);

    // Save secret to database (encrypted!)
    await db.saveMFASecret(userId, encrypt(secret.base32));

    return {
      secret: secret.base32,
      qrCode
    };
  }

  async verifyMFA(userId: string, token: string): Promise<boolean> {
    const secret = await db.getMFASecret(userId);

    if (!secret) {
      throw new Error('MFA not set up');
    }

    return speakeasy.totp.verify({
      secret: decrypt(secret),
      encoding: 'base32',
      token,
      window: 1  // Allow 30s clock drift
    });
  }
}
```

## Token Revocation

```sql
-- Revoked tokens table
CREATE TABLE revoked_tokens (
  jti VARCHAR(255) PRIMARY KEY,  -- JWT ID
  revoked_at TIMESTAMP DEFAULT NOW(),
  expires_at TIMESTAMP NOT NULL,

  INDEX idx_expires (expires_at)
);

-- Refresh tokens table
CREATE TABLE refresh_tokens (
  id UUID PRIMARY KEY,
  user_id UUID NOT NULL,
  token_hash VARCHAR(64) NOT NULL,
  expires_at TIMESTAMP NOT NULL,
  revoked BOOLEAN DEFAULT FALSE,

  INDEX idx_user (user_id),
  INDEX idx_expires (expires_at)
);
```

```typescript
async function verifyToken(token: string): Promise<TokenPayload> {
  const payload = jwt.verify(token, publicKey);

  // Check if token is revoked
  const isRevoked = await redis.get(`revoked:${payload.jti}`);
  if (isRevoked) {
    throw new AuthError('TOKEN_REVOKED');
  }

  return payload;
}

async function revokeToken(jti: string, expiresAt: Date): Promise<void> {
  const ttl = Math.floor((expiresAt.getTime() - Date.now()) / 1000);

  // Store in Redis with TTL
  await redis.set(`revoked:${jti}`, '1', 'EX', ttl);

  // Also store in database for persistence
  await db.query(
    'INSERT INTO revoked_tokens (jti, expires_at) VALUES ($1, $2)',
    [jti, expiresAt]
  );
}
```

## Session Management

```typescript
interface Session {
  id: string;
  userId: string;
  ipAddress: string;
  userAgent: string;
  createdAt: Date;
  lastActivityAt: Date;
  expiresAt: Date;
}

class SessionManager {
  async createSession(userId: string, req: Request): Promise<string> {
    const sessionId = uuid();

    await redis.setex(
      `session:${sessionId}`,
      3600,  // 1 hour
      JSON.stringify({
        userId,
        ipAddress: req.ip,
        userAgent: req.headers['user-agent'],
        createdAt: new Date(),
        lastActivityAt: new Date()
      })
    );

    return sessionId;
  }

  async getSession(sessionId: string): Promise<Session | null> {
    const data = await redis.get(`session:${sessionId}`);
    return data ? JSON.parse(data) : null;
  }

  async extendSession(sessionId: string): Promise<void> {
    await redis.expire(`session:${sessionId}`, 3600);
  }

  async revokeSession(sessionId: string): Promise<void> {
    await redis.del(`session:${sessionId}`);
  }

  async revokeAllUserSessions(userId: string): Promise<void> {
    // Find all sessions for user
    const keys = await redis.keys(`session:*`);

    for (const key of keys) {
      const session = await redis.get(key);
      if (session && JSON.parse(session).userId === userId) {
        await redis.del(key);
      }
    }
  }
}
```

## API Design

```http
# Login
POST /api/v1/auth/login
Body: {
  "email": "user@example.com",
  "password": "password123"
}
Response: {
  "access_token": "eyJhbGc...",
  "refresh_token": "eyJhbGc...",
  "expires_in": 900
}

# Refresh token
POST /api/v1/auth/refresh
Body: {
  "refresh_token": "eyJhbGc..."
}

# Logout
POST /api/v1/auth/logout
Authorization: Bearer eyJhbGc...

# Revoke all sessions
POST /api/v1/auth/revoke-all
Authorization: Bearer eyJhbGc...

# Setup MFA
POST /api/v1/auth/mfa/setup
Response: {
  "secret": "JBSWY3DPEHPK3PXP",
  "qr_code": "data:image/png;base64,..."
}

# Verify MFA
POST /api/v1/auth/mfa/verify
Body: {
  "token": "123456"
}
```

## Summary

### What We Built:
- Zero-trust authentication system
- JWT with refresh tokens
- OAuth 2.0 integration
- mTLS for service-to-service auth
- RBAC for authorization
- MFA with TOTP
- Token revocation

### Key Concepts:
- ✅ JWT structure and validation
- ✅ OAuth 2.0 authorization code flow
- ✅ mTLS (mutual TLS)
- ✅ RBAC (role-based access control)
- ✅ Token revocation strategies
- ✅ Session management

### Real-World Examples:
- Auth0: Managed authentication platform
- Okta: Enterprise identity management
- AWS Cognito: User authentication for apps

---

**Next Project**: [12. High-Frequency Trading Platform](../12-trading-platform/README.md)
