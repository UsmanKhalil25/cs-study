---
topic_type: security
status: learning
difficulty: intermediate
tags:
  - cs
  - authentication
  - oauth
  - jwt
  - firebase
  - google-oauth
  - github-oauth
  - graphql
  - token-validation
  - user-sync
prerequisites:
  - "[[HTTP And REST APIs]]"
  - "[[JSON Web Tokens]]"
  - "[[Cryptography Basics]]"
date: 2026-04-29
updated: 2026-06-14
---

# External Authentication Providers

## Overview

External authentication allows your application to delegate identity verification to trusted third-party providers (Firebase, Google, GitHub) rather than managing passwords yourself. The provider handles login UI, credential storage, and MFA, then issues cryptographically signed tokens that your backend verifies to establish user identity.

## Trust Boundary Model

External auth is not “trust whatever the client sends.” The provider authenticates the user; your backend still owns token verification, local authorization, user linking, session policy, and revocation behavior.

```mermaid
flowchart TD
    A[Browser / Mobile App] -->|OAuth redirect or SDK login| B[Identity Provider]
    B -->|ID token / auth code| A
    A -->|Token to verify| C[Backend Auth Boundary]
    C --> D[JWKS / Token Introspection]
    D --> B
    C --> E[Local User Record]
    C --> F[Authorization Rules]
    C --> G[Session / App Token]
    G --> A
    A --> H[Application API]
    H --> C
```

| Token Type | What It Proves | Backend Handling |
|------------|----------------|------------------|
| ID token | User identity from OIDC-style providers | Verify signature, issuer, audience, expiry, nonce when relevant |
| Access token | Permission to call a resource API | Treat as provider-specific; often opaque outside that provider |
| Authorization code | Short-lived credential to exchange server-side | Exchange with client secret/PKCE; never treat as identity alone |
| App session | Your application's login state | Issue after provider verification and local user sync |

## How External Auth Providers Work

### The Core Pattern

All external auth follows the same fundamental flow:

1. User authenticates with the provider (Google login button, Firebase UI, GitHub OAuth)
2. Provider issues a token (JWT or opaque) proving the user's identity
3. Client sends this token to your backend
4. Your backend **cryptographically verifies** the token
5. If valid, your backend creates/finds the user record and establishes a session

```mermaid
sequenceDiagram
    participant User
    participant Client
    participant Provider as Auth Provider<br/>(Firebase/Google/GitHub)
    participant Backend as Your Backend

    User->>Client: Click "Sign in with X"
    Client->>Provider: Redirect to auth page
    Provider->>User: Show login/consent
    User->>Provider: Enter credentials + approve
    Provider->>Client: Return token (code exchange or direct)
    Client->>Backend: POST /auth with token
    Backend->>Provider: Verify token (JWKS or API call)
    Provider-->>Backend: Token valid + user claims
    Backend->>Backend: Sync user to DB
    Backend-->>Client: Session token / user data
    Client-->>User: Logged in
```

### Firebase Auth

Firebase Auth is a **managed identity platform** that wraps multiple providers (Google, GitHub, email/password, phone, etc.) behind a unified SDK.

**How Firebase ID Tokens Work:**

- Firebase ID tokens are **JWTs** signed by Firebase's private keys
- Tokens expire after **1 hour** (3600 seconds)
- The Firebase client SDK automatically refreshes tokens in the background
- Token format: `header.payload.signature` (standard JWT)

**Firebase ID Token Payload:**

```json
{
  "iss": "https://securetoken.google.com/<projectId>",
  "aud": "<projectId>",
  "auth_time": 1714329600,
  "user_id": "<firebase-uid>",
  "sub": "<firebase-uid>",
  "iat": 1714329600,
  "exp": 1714333200,
  "email": "user@example.com",
  "email_verified": true,
  "firebase": {
    "identities": {
      "google.com": ["1234567890"],
      "email": ["user@example.com"]
    },
    "sign_in_provider": "google.com"
  }
}
```

**Key claims:**
- `iss` — Issuer, always `https://securetoken.google.com/<projectId>`
- `aud` — Audience, your Firebase project ID
- `sub` / `user_id` — The Firebase UID (unique per user, per project)
- `exp` — Expiration (1 hour from issue)
- `firebase.sign_in_provider` — Which provider the user signed in with

**Verifying Firebase Tokens on Your Backend:**

Two approaches:

**Approach 1: Firebase Admin SDK (recommended)**

```typescript
import admin from 'firebase-admin';

// Initialize once at server startup
admin.initializeApp({
  credential: admin.credential.cert({
    projectId: process.env.FIREBASE_PROJECT_ID,
    clientEmail: process.env.FIREBASE_CLIENT_EMAIL,
    privateKey: process.env.FIREBASE_PRIVATE_KEY?.replace(/\\n/g, '\n'),
  }),
});

// Verify token on each request
async function verifyFirebaseToken(idToken: string) {
  try {
    const decodedToken = await admin.auth().verifyIdToken(idToken);
    return decodedToken;
    // decodedToken.uid, decodedToken.email, decodedToken.firebase.sign_in_provider
  } catch (error) {
    // Token is invalid, expired, or revoked
    throw new Error('Invalid Firebase token');
  }
}
```

**Approach 2: Manual JWT verification (no Admin SDK dependency)**

```typescript
import jwt from 'jsonwebtoken';
import jwksClient from 'jwks-rsa';

const client = jwksClient({
  jwksUri: 'https://www.googleapis.com/service_accounts/v1/jwk/securetoken@system.gserviceaccount.com',
});

function getSigningKey(header: jwt.JwtHeader, callback: jwt.SigningKeyCallback) {
  client.getSigningKey(header.kid!, (err, key) => {
    if (err) return callback(err);
    const signingKey = key!.getPublicKey();
    callback(null, signingKey);
  });
}

async function verifyFirebaseTokenManual(idToken: string) {
  return new Promise((resolve, reject) => {
    jwt.verify(
      idToken,
      getSigningKey,
      {
        algorithms: ['RS256'],
        issuer: `https://securetoken.google.com/${process.env.FIREBASE_PROJECT_ID}`,
        audience: process.env.FIREBASE_PROJECT_ID,
      },
      (err, decoded) => {
        if (err) return reject(err);
        resolve(decoded);
      }
    );
  });
}
```

**Firebase Token Refresh Flow:**

```mermaid
sequenceDiagram
    participant Client
    participant Firebase
    participant Backend

    Client->>Firebase: Auto-refresh (every ~55 min)
    Firebase-->>Client: New ID token (1hr validity)
    Client->>Backend: Request with new token
    Backend->>Firebase: Verify token
    Firebase-->>Backend: Valid
    Backend-->>Client: Response
```

The Firebase client SDK handles refresh automatically. You don't need to implement refresh logic on the client — just send the current token with each request.

**What Firebase Provides vs What You Store:**

| Firebase Provides | You Store Yourself |
|---|---|
| `uid` (Firebase UID) | Your own `user_id` (UUID) |
| `email` | Business-specific preferences |
| `email_verified` | Role/permissions |
| `display_name` | Order history, activity logs |
| `photo_url` | Provider linking data |
| `phone_number` | Custom profile fields |
| Provider-specific IDs | Your session tokens |

### Google OAuth 2.0

Google uses the **OpenID Connect** protocol (built on OAuth 2.0) for authentication.

**Authorization Code Flow (Server Flow):**

```mermaid
sequenceDiagram
    participant User
    participant Client
    participant Backend
    participant Google

    User->>Client: Click "Sign in with Google"
    Client->>Backend: Request auth URL
    Backend->>Backend: Generate state + nonce
    Backend-->>Client: Return auth URL
    Client->>Google: Redirect to accounts.google.com/o/oauth2/v2/auth
    Google->>User: Show login + consent screen
    User->>Google: Approve
    Google->>Client: Redirect to /callback?code=XXX&state=YYY
    Client->>Backend: Forward code + state
    Backend->>Backend: Verify state matches
    Backend->>Google: POST /token (code + client_secret)
    Google-->>Backend: access_token + id_token + refresh_token
    Backend->>Google: Verify id_token (JWKS)
    Google-->>Backend: Valid
    Backend->>Backend: Decode id_token claims
    Backend-->>Client: Session established
```

**Step-by-Step Breakdown:**

**Step 1: Generate anti-forgery state token**

```typescript
import crypto from 'crypto';

function generateState(): string {
  return crypto.randomBytes(32).toString('hex');
}

// Store in session or Redis with TTL
const state = generateState();
session.set('oauth_state', state);
```

**Step 2: Build authorization URL**

```typescript
function buildGoogleAuthUrl(state: string): string {
  const params = new URLSearchParams({
    client_id: process.env.GOOGLE_CLIENT_ID!,
    redirect_uri: process.env.GOOGLE_REDIRECT_URI!,
    response_type: 'code',
    scope: 'openid email profile',
    state: state,
    nonce: crypto.randomBytes(16).toString('hex'),
    prompt: 'consent', // force consent screen
  });

  return `https://accounts.google.com/o/oauth2/v2/auth?${params.toString()}`;
}
```

**Scopes and what data they provide:**

| Scope | Data Provided |
|---|---|
| `openid` | Required for OIDC, enables ID token |
| `email` | `email`, `email_verified` claims |
| `profile` | `name`, `given_name`, `family_name`, `picture`, `locale` |
| `https://www.googleapis.com/auth/drive.file` | Access to user's Drive files |

**Step 3: Exchange authorization code for tokens**

```typescript
async function exchangeCodeForTokens(code: string) {
  const tokenResponse = await fetch('https://oauth2.googleapis.com/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      code,
      client_id: process.env.GOOGLE_CLIENT_ID!,
      client_secret: process.env.GOOGLE_CLIENT_SECRET!,
      redirect_uri: process.env.GOOGLE_REDIRECT_URI!,
      grant_type: 'authorization_code',
    }),
  });

  if (!tokenResponse.ok) {
    throw new Error('Token exchange failed');
  }

  const tokens = await tokenResponse.json();
  return {
    accessToken: tokens.access_token,
    idToken: tokens.id_token,
    refreshToken: tokens.refresh_token, // only if access_type=offline was requested
    expiresIn: tokens.expires_in,
  };
}
```

**Step 4: Validate Google ID Token**

**Method A: Google's tokeninfo endpoint (debugging only, not for production)**

```typescript
async function validateWithTokenInfo(idToken: string) {
  const response = await fetch(
    `https://oauth2.googleapis.com/tokeninfo?id_token=${idToken}`
  );

  if (!response.ok) {
    throw new Error('Invalid token');
  }

  const payload = await response.json();

  // Verify audience matches your client ID
  if (payload.aud !== process.env.GOOGLE_CLIENT_ID) {
    throw new Error('Token audience mismatch');
  }

  return payload;
}
```

> [!warning] Tokeninfo endpoint is for debugging only
> Google may throttle or error on this endpoint. Always use local JWKS verification in production.

**Method B: Manual JWT verification with Google's public keys (production)**

```typescript
import jwt from 'jsonwebtoken';
import jwksClient from 'jwks-rsa';

const googleJwksClient = jwksClient({
  jwksUri: 'https://www.googleapis.com/oauth2/v3/certs',
  cache: true,
  cacheMaxEntries: 5,
  cacheMaxAge: 600_000, // 10 minutes
  rateLimit: true,
  jwksRequestsPerMinute: 10,
});

function getGoogleSigningKey(header: jwt.JwtHeader, callback: jwt.SigningKeyCallback) {
  googleJwksClient.getSigningKey(header.kid!, (err, key) => {
    if (err) return callback(err);
    callback(null, key!.getPublicKey());
  });
}

async function verifyGoogleIdToken(idToken: string) {
  return new Promise((resolve, reject) => {
    jwt.verify(
      idToken,
      getGoogleSigningKey,
      {
        algorithms: ['RS256'],
        issuer: 'https://accounts.google.com',
        audience: process.env.GOOGLE_CLIENT_ID,
      },
      (err, decoded) => {
        if (err) return reject(err);
        resolve(decoded);
      }
    );
  });
}
```

**Google ID Token Claims:**

```json
{
  "iss": "https://accounts.google.com",
  "azp": "<client-id>",
  "aud": "<client-id>",
  "sub": "10769150350006150715113082367",
  "email": "user@gmail.com",
  "email_verified": true,
  "name": "John Doe",
  "picture": "https://lh3.googleusercontent.com/...",
  "iat": 1714329600,
  "exp": 1714333200,
  "nonce": "random-nonce-value"
}
```

> [!warning] Never use `email` as unique identifier
> Always use `sub` — it is permanent and never changes, even if the user changes their email.

**Google's Discovery Document:**

Google publishes configuration at `https://accounts.google.com/.well-known/openid-configuration`:

```json
{
  "issuer": "https://accounts.google.com",
  "authorization_endpoint": "https://accounts.google.com/o/oauth2/v2/auth",
  "token_endpoint": "https://oauth2.googleapis.com/token",
  "userinfo_endpoint": "https://openidconnect.googleapis.com/v1/userinfo",
  "jwks_uri": "https://www.googleapis.com/oauth2/v3/certs",
  "id_token_signing_alg_values_supported": ["RS256"]
}
```

### GitHub OAuth

GitHub OAuth is **simpler but different** from Google's OIDC flow. GitHub does **not** issue JWT ID tokens — it returns an opaque access token, and you must call the `/user` API to get identity information.

**GitHub OAuth Flow:**

```mermaid
sequenceDiagram
    participant User
    participant Client
    participant Backend
    participant GitHub

    User->>Client: Click "Sign in with GitHub"
    Client->>Backend: Request auth URL
    Backend->>Backend: Generate state
    Backend-->>Client: auth URL
    Client->>GitHub: Redirect to github.com/login/oauth/authorize
    GitHub->>User: Show authorization page
    User->>GitHub: Authorize
    GitHub->>Client: Redirect to /callback?code=XXX&state=YYY
    Client->>Backend: Forward code + state
    Backend->>Backend: Verify state
    Backend->>GitHub: POST /login/oauth/access_token
    GitHub-->>Backend: access_token (opaque)
    Backend->>GitHub: GET /user (with access_token)
    GitHub-->>Backend: User profile JSON
    Backend->>Backend: Find/create user in DB
    Backend-->>Client: Session established
```

**Step 1: Build authorization URL**

```typescript
function buildGitHubAuthUrl(state: string): string {
  const params = new URLSearchParams({
    client_id: process.env.GITHUB_CLIENT_ID!,
    redirect_uri: process.env.GITHUB_REDIRECT_URI!,
    scope: 'read:user user:email',
    state: state,
  });

  return `https://github.com/login/oauth/authorize?${params.toString()}`;
}
```

**GitHub OAuth scopes:**

| Scope | Data Provided |
|---|---|
| `read:user` | User profile (name, avatar, bio, etc.) |
| `user:email` | All email addresses (including private) |
| `repo` | Full repo access |
| `admin:org` | Organization admin access |

**Step 2: Exchange code for access token**

```typescript
async function exchangeGitHubCode(code: string, state: string) {
  // Verify state first
  if (state !== session.get('oauth_state')) {
    throw new Error('Invalid state parameter');
  }

  const response = await fetch('https://github.com/login/oauth/access_token', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Accept: 'application/json',
    },
    body: JSON.stringify({
      client_id: process.env.GITHUB_CLIENT_ID!,
      client_secret: process.env.GITHUB_CLIENT_SECRET!,
      code,
      redirect_uri: process.env.GITHUB_REDIRECT_URI!,
    }),
  });

  const data = await response.json();

  if (data.error) {
    throw new Error(`GitHub OAuth error: ${data.error_description}`);
  }

  return data.access_token;
}
```

**Step 3: Get user info from GitHub API**

```typescript
async function getGitHubUser(accessToken: string) {
  const response = await fetch('https://api.github.com/user', {
    headers: {
      Authorization: `Bearer ${accessToken}`,
      Accept: 'application/vnd.github.v3+json',
    },
  });

  if (!response.ok) {
    throw new Error('Failed to fetch GitHub user');
  }

  const user = await response.json();

  // Also fetch emails if user:email scope was requested
  const emailsResponse = await fetch('https://api.github.com/user/emails', {
    headers: {
      Authorization: `Bearer ${accessToken}`,
      Accept: 'application/vnd.github.v3+json',
    },
  });

  const emails = await emailsResponse.json();
  const primaryEmail = emails.find((e: any) => e.primary)?.email;

  return {
    providerId: String(user.id), // GitHub numeric ID — never changes
    username: user.login,
    displayName: user.name,
    email: primaryEmail || user.email,
    avatarUrl: user.avatar_url,
    profileUrl: user.html_url,
  };
}
```

> [!important] GitHub does NOT use JWT for identity
> Unlike Google/Firebase, GitHub returns an opaque access token. You MUST call the `/user` API endpoint to get identity information. There is no ID token to decode or verify cryptographically.

## Token Validation Deep Dive

### JWT Structure

A JWT consists of three Base64Url-encoded parts separated by dots:

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWUsImlhdCI6MTUxNjIzOTAyMn0.POstGetfAytaZS82wHcjoTyoqhMyxXiWdR7Nn7A29DNSl0EiXLdwJ6xC6AfgZWF1bOsS_TuYI3OG85AmiExREkrS6tDfTQ2B3WXlrr-wp5AokiRbz3_oB4OxG-W9KcEEbDRcZc0nH3L7LzYptiy1PtAylQGxHTWZXtGz4ht0bAecBgmpdgXMguEIcoqPJ1n3pIWk_dUZegpqx0Lka21H6XxUTxiy8OcaarA8zdnPUnV6AmNP3ecFawIFYdvJB_cm-GvpCSbr8G8y_Mllj8f4x9nBH8pQux89_6gUY618iYv7tuPWBFfEbLxtF2pZS6YC1aSfLQXNe78TbSScrwLIcWN9wKGw
```

**Header** (decoded):
```json
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "abc123"
}
```

**Payload** (decoded):
```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "iat": 1516239022,
  "exp": 1516242622
}
```

**Signature**: Cryptographic signature created by signing `header.payload` with the provider's private key.

### JWKS — JSON Web Key Set

Providers publish their public keys at a JWKS endpoint. Your backend fetches these keys to verify JWT signatures.

**Google JWKS endpoint:** `https://www.googleapis.com/oauth2/v3/certs`
**Firebase JWKS endpoint:** `https://www.googleapis.com/service_accounts/v1/jwk/securetoken@system.gserviceaccount.com`

Example JWKS response:

```json
{
  "keys": [
    {
      "kid": "abc123",
      "kty": "RSA",
      "alg": "RS256",
      "use": "sig",
      "n": "0vx7agoebGcQSuuPiLJXZptN9nndrQmbXEps2aiAFbWhM78LhWx4cbbf...",
      "e": "AQAB"
    },
    {
      "kid": "def456",
      "kty": "RSA",
      "alg": "RS256",
      "use": "sig",
      "n": "tDEeDEpQnF7Cg...",
      "e": "AQAB"
    }
  ]
}
```

The `kid` (Key ID) in the JWT header tells you which key to use for verification.

**Verification flow:**

```mermaid
graph TD
    A[Receive JWT] --> B[Decode header to get kid]
    B --> C{Key in cache?}
    C -->|Yes| D[Use cached public key]
    C -->|No| E[Fetch JWKS from provider]
    E --> F[Find key matching kid]
    F --> G[Cache key with TTL]
    D --> H[Verify signature RS256]
    G --> H
    H --> I{Signature valid?}
    I -->|No| J[Reject token]
    I -->|Yes| K[Verify claims: iss, aud, exp]
    K --> L{Claims valid?}
    L -->|No| J
    L -->|Yes| M[Token authenticated]
```

### Claims You MUST Verify

| Claim | What to Check | Why |
|---|---|---|
| `iss` (issuer) | Matches expected provider URL | Prevents tokens from other providers |
| `aud` (audience) | Matches your client ID / project ID | Prevents tokens meant for other apps |
| `exp` (expiration) | Current time < exp | Prevents use of expired tokens |
| `iat` (issued at) | Reasonable time (not in future) | Detects clock skew attacks |
| `nonce` | Matches the nonce you sent | Prevents replay attacks |

```typescript
function validateClaims(payload: any, expectedIssuer: string, expectedAudience: string) {
  const now = Math.floor(Date.now() / 1000);

  if (payload.iss !== expectedIssuer) {
    throw new Error(`Invalid issuer: expected ${expectedIssuer}, got ${payload.iss}`);
  }

  if (payload.aud !== expectedAudience) {
    throw new Error(`Invalid audience: expected ${expectedAudience}, got ${payload.aud}`);
  }

  if (payload.exp < now) {
    throw new Error(`Token expired at ${new Date(payload.exp * 1000).toISOString()}`);
  }

  if (payload.iat > now + 300) { // 5 min clock skew tolerance
    throw new Error('Token issued in the future — possible clock manipulation');
  }
}
```

> [!danger] NEVER trust a token without cryptographic verification
> Any attacker can create a JWT with arbitrary claims. The signature is the ONLY thing that proves the token came from the provider. Always verify the signature using the provider's public keys via JWKS.

## User Synchronization Patterns

### Database Schema

```mermaid
erDiagram
    USERS {
        uuid id PK
        string email
        string display_name
        timestamp created_at
        timestamp updated_at
        timestamp last_login
    }

    USER_PROVIDERS {
        uuid id PK
        uuid user_id FK
        string provider "firebase, google, github"
        string provider_id "unique per provider"
        string provider_email
        string access_token
        string refresh_token
        timestamp token_expires_at
        timestamp linked_at
    }

    USER_PROFILES {
        uuid id PK
        uuid user_id FK
        string avatar_url
        string bio
        json preferences
        json metadata
    }

    USERS ||--o{ USER_PROVIDERS : "has"
    USERS ||--o| USER_PROFILES : "has"
```

### First-Time Login Flow

```typescript
async function handleExternalLogin(
  provider: 'firebase' | 'google' | 'github',
  providerData: ExternalUserRecord
): Promise<User> {
  return db.transaction(async (tx) => {
    // 1. Check if user already exists with this provider
    const existingProvider = await tx.userProviders.findFirst({
      where: {
        provider,
        providerId: providerData.providerId,
      },
      include: { user: true },
    });

    if (existingProvider) {
      // 2. Returning user — update profile data and last login
      await tx.userProviders.update({
        where: { id: existingProvider.id },
        data: {
          providerEmail: providerData.email,
          lastSeenAt: new Date(),
        },
      });

      await tx.users.update({
        where: { id: existingProvider.userId },
        data: {
          lastLogin: new Date(),
          email: providerData.email,
          displayName: providerData.displayName,
        },
      });

      return existingProvider.user;
    }

    // 3. Check if email already exists (potential account merge)
    const existingUserByEmail = await tx.users.findFirst({
      where: { email: providerData.email },
    });

    if (existingUserByEmail) {
      // 4. Link this provider to existing account
      await tx.userProviders.create({
        data: {
          userId: existingUserByEmail.id,
          provider,
          providerId: providerData.providerId,
          providerEmail: providerData.email,
          linkedAt: new Date(),
        },
      });

      await tx.users.update({
        where: { id: existingUserByEmail.id },
        data: { lastLogin: new Date() },
      });

      return existingUserByEmail;
    }

    // 5. Brand new user — create everything
    const newUser = await tx.users.create({
      data: {
        email: providerData.email,
        displayName: providerData.displayName,
        lastLogin: new Date(),
        providers: {
          create: {
            provider,
            providerId: providerData.providerId,
            providerEmail: providerData.email,
            linkedAt: new Date(),
          },
        },
        profile: {
          create: {
            avatarUrl: providerData.avatarUrl,
          },
        },
      },
      include: { providers: true, profile: true },
    });

    return newUser;
  });
}
```

### What to Store

| Field | Why |
|---|---|
| `provider` | Which auth provider (firebase, google, github) |
| `provider_id` | Immutable ID from provider (Firebase UID, Google `sub`, GitHub numeric ID) |
| `provider_email` | Email at time of link (may change) |
| `access_token` | For calling provider APIs (optional, store encrypted) |
| `refresh_token` | For offline access (Google only, store encrypted) |
| `token_expires_at` | When the access token expires |
| `linked_at` | When this provider was linked to the account |
| `last_seen_at` | Last time this provider was used to log in |

### Handling Email Changes

```typescript
async function handleEmailChange(userId: string, newEmail: string) {
  // Update the primary user email
  await db.users.update({
    where: { id: userId },
    data: { email: newEmail },
  });

  // Update all provider email records
  await db.userProviders.updateMany({
    where: { userId },
    data: { providerEmail: newEmail },
  });
}
```

### Account Merging Strategy

```mermaid
graph TD
    A[User logs in with Provider B] --> B{Email matches existing user?}
    B -->|No| C[Create new account]
    B -->|Yes| D{Existing user has Provider A?}
    D -->|No| E[Link Provider B to existing account]
    D -->|Yes| F{Same provider_id?}
    F -->|Yes| G[Normal login — same account]
    F -->|No| H[Conflict — different accounts, same email]
    H --> I[Ask user to verify ownership]
    I --> J[Merge accounts or reject]
```

### Linking Multiple Providers

```typescript
async function linkProvider(
  userId: string,
  provider: 'firebase' | 'google' | 'github',
  providerData: ExternalUserRecord
) {
  // Check if this provider_id is already linked to ANOTHER user
  const conflict = await db.userProviders.findFirst({
    where: {
      provider,
      providerId: providerData.providerId,
      userId: { not: userId },
    },
  });

  if (conflict) {
    throw new Error('This provider account is already linked to another user');
  }

  // Check if user already has this provider linked
  const existing = await db.userProviders.findFirst({
    where: { userId, provider },
  });

  if (existing) {
    throw new Error(`You already have ${provider} linked to your account`);
  }

  return db.userProviders.create({
    data: {
      userId,
      provider,
      providerId: providerData.providerId,
      providerEmail: providerData.email,
      linkedAt: new Date(),
    },
  });
}
```

## GraphQL Integration

### Architecture Overview

```mermaid
graph LR
    A[Client] -->|Authorization: Bearer <token>| B[GraphQL Endpoint]
    B --> C[Context Function]
    C --> D[Verify Token]
    D --> E[Attach user to context]
    E --> F[Resolvers]
    F --> G[Access ctx.user]
```

### Passing the Token

The client sends the external provider's token in the `Authorization` header:

```typescript
// Client side (Apollo Client example)
import { ApolloClient, InMemoryCache, createHttpLink } from '@apollo/client';
import { setContext } from '@apollo/client/link/context';

const httpLink = createHttpLink({
  uri: 'https://api.example.com/graphql',
});

const authLink = setContext(async (_, { headers }) => {
  // Get the Firebase ID token (auto-refreshed by Firebase SDK)
  const firebaseUser = auth.currentUser;
  const token = firebaseUser ? await firebaseUser.getIdToken() : null;

  return {
    headers: {
      ...headers,
      Authorization: token ? `Bearer ${token}` : '',
    },
  };
});

const client = new ApolloClient({
  link: authLink.concat(httpLink),
  cache: new InMemoryCache(),
});
```

### Context Creation — Token Verification

```typescript
import { Context } from 'apollo-server-core';
import admin from 'firebase-admin';

interface GraphQLContext {
  user: AuthenticatedUser | null;
  db: PrismaClient;
}

interface AuthenticatedUser {
  id: string;           // Your internal user ID
  email: string;
  displayName: string;
  provider: string;     // 'firebase', 'google', 'github'
  providerId: string;   // Provider's unique ID
  roles: string[];
}

async function createContext({ req }: { req: Request }): Promise<GraphQLContext> {
  const authHeader = req.headers.authorization || '';
  const token = authHeader.startsWith('Bearer ') ? authHeader.slice(7) : null;

  let authenticatedUser: AuthenticatedUser | null = null;

  if (token) {
    try {
      // Step 1: Verify the Firebase ID token
      const decodedToken = await admin.auth().verifyIdToken(token);

      // Step 2: Find the user in your database
      const userProvider = await db.userProviders.findFirst({
        where: {
          provider: 'firebase',
          providerId: decodedToken.uid,
        },
        include: { user: true },
      });

      if (userProvider) {
        authenticatedUser = {
          id: userProvider.user.id,
          email: userProvider.user.email,
          displayName: userProvider.user.displayName,
          provider: 'firebase',
          providerId: decodedToken.uid,
          roles: userProvider.user.roles,
        };
      }
    } catch (error) {
      // Token invalid — user will be null (unauthenticated)
      // Resolvers can check ctx.user for auth
      console.error('Token verification failed:', error);
    }
  }

  return {
    user: authenticatedUser,
    db,
  };
}
```

### Supporting Multiple Providers in Context

```typescript
async function verifyAndFindUser(token: string): Promise<AuthenticatedUser | null> {
  // Try Firebase first (most common)
  try {
    const decoded = await admin.auth().verifyIdToken(token);
    const provider = await db.userProviders.findFirst({
      where: { provider: 'firebase', providerId: decoded.uid },
      include: { user: true },
    });
    if (provider) {
      return {
        id: provider.user.id,
        email: provider.user.email,
        displayName: provider.user.displayName,
        provider: 'firebase',
        providerId: decoded.uid,
        roles: provider.user.roles,
      };
    }
  } catch {
    // Not a Firebase token, try Google
  }

  // Try Google ID token
  try {
    const decoded = await verifyGoogleIdToken(token);
    const provider = await db.userProviders.findFirst({
      where: { provider: 'google', providerId: (decoded as any).sub },
      include: { user: true },
    });
    if (provider) {
      return {
        id: provider.user.id,
        email: provider.user.email,
        displayName: provider.user.displayName,
        provider: 'google',
        providerId: (decoded as any).sub,
        roles: provider.user.roles,
      };
    }
  } catch {
    // Not a Google token either
  }

  // For GitHub, you'd need to exchange the code server-side first
  // and store a session token — GitHub doesn't send JWTs from the client

  return null;
}
```

### Resolver Access to Authenticated User

```typescript
const resolvers = {
  Query: {
    // Public query — no auth required
    publicPosts: async (_: any, __: any, ctx: GraphQLContext) => {
      return ctx.db.posts.findMany({
        where: { published: true },
      });
    },

    // Authenticated query — requires user
    me: async (_: any, __: any, ctx: GraphQLContext) => {
      if (!ctx.user) {
        throw new AuthenticationError('You must be logged in');
      }
      return ctx.user;
    },

    // User-specific data
    myOrders: async (_: any, __: any, ctx: GraphQLContext) => {
      if (!ctx.user) {
        throw new AuthenticationError('You must be logged in');
      }
      return ctx.db.orders.findMany({
        where: { userId: ctx.user.id },
      });
    },
  },

  Mutation: {
    // Mutation requiring authentication
    createPost: async (
      _: any,
      { input }: { input: CreatePostInput },
      ctx: GraphQLContext
    ) => {
      if (!ctx.user) {
        throw new AuthenticationError('You must be logged in');
      }

      return ctx.db.posts.create({
        data: {
          title: input.title,
          content: input.content,
          authorId: ctx.user.id,
        },
      });
    },

    // Admin-only mutation
    deleteUser: async (
      _: any,
      { userId }: { userId: string },
      ctx: GraphQLContext
    ) => {
      if (!ctx.user || !ctx.user.roles.includes('admin')) {
        throw new ForbiddenError('Admin access required');
      }

      return ctx.db.users.delete({ where: { id: userId } });
    },
  },
};
```

### Authentication Error Class

```typescript
import { GraphQLError } from 'graphql';

export class AuthenticationError extends GraphQLError {
  constructor(message: string) {
    super(message, { extensions: { code: 'UNAUTHENTICATED' } });
  }
}

export class ForbiddenError extends GraphQLError {
  constructor(message: string) {
    super(message, { extensions: { code: 'FORBIDDEN' } });
  }
}
```

### Complete GraphQL Server Setup

```typescript
import { ApolloServer } from '@apollo/server';
import { expressMiddleware } from '@apollo/server/express4';
import express from 'express';
import admin from 'firebase-admin';

// Initialize Firebase Admin
admin.initializeApp({
  credential: admin.credential.cert({
    projectId: process.env.FIREBASE_PROJECT_ID,
    clientEmail: process.env.FIREBASE_CLIENT_EMAIL,
    privateKey: process.env.FIREBASE_PRIVATE_KEY?.replace(/\\n/g, '\n'),
  }),
});

const app = express();

const server = new ApolloServer({
  typeDefs,
  resolvers,
});

await server.start();

app.use(
  '/graphql',
  express.json(),
  expressMiddleware(server, {
    context: createContext,
  })
);

app.listen(4000, () => {
  console.log('GraphQL server running at http://localhost:4000/graphql');
});
```

## Key Details

> [!warning] Never store client secrets in frontend code
> `client_secret` values must ONLY exist on your backend. The frontend only needs `client_id`.

> [!warning] Always use HTTPS
> Tokens transmitted over HTTP can be intercepted. All auth endpoints must use HTTPS.

> [!warning] Validate state parameter
> The `state` parameter prevents CSRF attacks. Always generate a random state, store it server-side, and verify it matches on callback.

> [!tip] Cache JWKS keys
> Fetch JWKS once and cache with the HTTP Cache-Control TTL. Google/Firebase rotate keys infrequently. The `jwks-rsa` library handles this automatically.

> [!tip] Use short-lived tokens
> External provider tokens expire in 1 hour. Create your own session mechanism (JWT or session cookie) with appropriate expiration for your app.

> [!warning] GitHub access tokens are NOT JWTs
> GitHub returns opaque tokens. You cannot decode them — you must call the `/user` API endpoint. This means GitHub OAuth requires a server-side code exchange step.

> [!tip] Store refresh tokens securely
> Google refresh tokens are only returned once (on first consent). Store them encrypted in your database. They have no expiration but can be revoked.

> [!danger] Never trust claims without signature verification
> An attacker can forge a JWT with any payload. The cryptographic signature is the only proof of authenticity. Always verify using the provider's public keys.

## Protocol Comparison: OAuth2 vs OIDC vs SAML

| | OAuth 2.0 | OIDC | SAML 2.0 |
|---|---|---|---|
| **Purpose** | Authorization (access delegation) | Authentication (identity) | Authentication (enterprise SSO) |
| **Token format** | Opaque or JWT | JWT (ID token) | XML assertions |
| **Transport** | HTTP/JSON | HTTP/JSON | HTTP/XML |
| **Era** | 2012 | 2014 | 2005 |
| **Use case** | Third-party API access | Social login, consumer apps | Enterprise SSO, B2B, LDAP-connected |
| **IdP examples** | GitHub, Google | Google, Microsoft, Auth0 | Okta, Azure AD, OneLogin |

**When to use SAML:** Your customers are enterprises that manage employees via Active Directory or Okta. They need SSO so employees log in once (IdP) and get access to your app (SP) without creating a separate account.

### SAML Flow

```mermaid
sequenceDiagram
    participant User
    participant SP as Service Provider<br/>(Your App)
    participant IdP as Identity Provider<br/>(Okta / Azure AD)

    User->>SP: Access protected resource
    SP->>User: Redirect to IdP with SAMLRequest
    User->>IdP: Authenticate (username + password + MFA)
    IdP->>User: SAML Response (signed XML assertion)
    User->>SP: POST SAML Response to ACS URL
    SP->>SP: Verify signature, extract attributes
    SP->>User: Create session, grant access
```

### SAML Implementation (Node.js with `samlify`)

```typescript
import * as samlify from "samlify";
import * as validator from "@authenio/samlify-node-xmllint";

samlify.setSchemaValidator(validator);

const sp = samlify.ServiceProvider({
  entityID: "https://yourapp.com/saml/metadata",
  assertionConsumerService: [{ Binding: samlify.Constants.BindingNamespace.Post, Location: "https://yourapp.com/saml/acs" }],
  privateKey: process.env.SAML_SP_PRIVATE_KEY,
});

const idp = samlify.IdentityProvider({
  metadata: process.env.SAML_IDP_METADATA_XML, // downloaded from Okta/Azure
});

// Generate SSO redirect URL
app.get("/saml/login", async (req, res) => {
  const { context } = await sp.createLoginRequest(idp, "redirect");
  res.redirect(context);
});

// Handle SAML response from IdP
app.post("/saml/acs", async (req, res) => {
  const { extract } = await sp.parseLoginResponse(idp, "post", req);
  const email = extract.attributes.email as string;
  const session = await createUserSession(email);
  res.cookie("session", session.token, { httpOnly: true, secure: true });
  res.redirect("/dashboard");
});
```

## Session Management Patterns

After authenticating via any provider (Firebase, OAuth, SAML), you must manage the user's ongoing session.

### Session Token Strategies

| Pattern | How It Works | Pros | Cons |
|---|---|---|---|
| **Opaque session cookie** | Random token stored in DB/Redis, cookie sent with each request | Easy to revoke, no token size limits | DB lookup per request |
| **JWT (stateless)** | Self-contained signed token, no server storage | No DB lookup, scales easily | Can't revoke until expiry |
| **JWT + refresh token** | Short-lived JWT (15min) + long-lived refresh token (7d) | Balance: revocable, low-DB-lookup | Two tokens to manage |

### Sliding Expiry with Refresh Tokens

```typescript
// On every API request — extend session if it's within the refresh window
app.use(async (req, res, next) => {
  const token = req.cookies.accessToken;
  if (!token) return next();

  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET) as JwtPayload;
    req.user = payload;

    // Slide the expiry: if token expires in < 5 minutes, issue a new one
    const expiresAt = payload.exp! * 1000;
    const minsRemaining = (expiresAt - Date.now()) / 60_000;
    if (minsRemaining < 5) {
      const newToken = jwt.sign({ userId: payload.userId }, process.env.JWT_SECRET, {
        expiresIn: "15m",
      });
      res.cookie("accessToken", newToken, { httpOnly: true, secure: true, sameSite: "strict" });
    }
  } catch {
    res.clearCookie("accessToken");
  }
  next();
});

// Refresh token endpoint (called when access token expires)
app.post("/auth/refresh", async (req, res) => {
  const refreshToken = req.cookies.refreshToken;
  const stored = await db.refreshTokens.findUnique({ where: { token: refreshToken } });

  if (!stored || stored.expiresAt < new Date()) {
    return res.status(401).json({ error: "Refresh token expired" });
  }

  // Rotate refresh token (prevents replay attacks)
  await db.refreshTokens.delete({ where: { token: refreshToken } });
  const newRefresh = crypto.randomBytes(32).toString("hex");
  await db.refreshTokens.create({
    data: { token: newRefresh, userId: stored.userId, expiresAt: addDays(new Date(), 7) },
  });

  const accessToken = jwt.sign({ userId: stored.userId }, process.env.JWT_SECRET, { expiresIn: "15m" });
  res.cookie("accessToken", accessToken, { httpOnly: true, secure: true, sameSite: "strict" });
  res.cookie("refreshToken", newRefresh, { httpOnly: true, secure: true, sameSite: "strict" });
  res.json({ ok: true });
});
```

## Authorization (RBAC)

Authentication = who you are. Authorization = what you're allowed to do. These are separate concerns.

**Role-Based Access Control (RBAC):**

```typescript
// Define roles and permissions
const permissions = {
  admin:  ["users:read", "users:write", "users:delete", "reports:read"],
  editor: ["users:read", "reports:read", "content:write"],
  viewer: ["users:read", "reports:read"],
} as const;

type Role = keyof typeof permissions;
type Permission = (typeof permissions)[Role][number];

function hasPermission(role: Role, permission: Permission): boolean {
  return (permissions[role] as readonly string[]).includes(permission);
}

// Middleware
function requirePermission(permission: Permission) {
  return (req: Request, res: Response, next: NextFunction) => {
    const userRole = req.user?.role as Role;
    if (!userRole || !hasPermission(userRole, permission)) {
      return res.status(403).json({ error: "Forbidden" });
    }
    next();
  };
}

app.delete("/users/:id", requirePermission("users:delete"), deleteUser);
app.get("/reports", requirePermission("reports:read"), getReports);
```

**Attribute-Based Access Control (ABAC) — more flexible for complex rules:**

```typescript
// Users can only edit their own posts (unless admin)
function canEditPost(user: User, post: Post): boolean {
  return user.role === "admin" || post.authorId === user.id;
}

app.put("/posts/:id", async (req, res) => {
  const post = await db.posts.findById(req.params.id);
  if (!canEditPost(req.user, post)) return res.status(403).json({ error: "Forbidden" });
  // ...
});
```

## Passkeys & WebAuthn

Passkeys replace passwords with public-key cryptography stored on the user's device (Face ID, Touch ID, Windows Hello). Resistant to phishing, no secrets sent over the wire.

### Flow

```mermaid
sequenceDiagram
    participant User
    participant Browser
    participant Server
    participant Authenticator as Authenticator (Face ID / Touch ID)

    note over User,Authenticator: Registration
    User->>Server: "Register passkey"
    Server-->>Browser: challenge + relyingParty options
    Browser->>Authenticator: create credential
    Authenticator->>User: biometric prompt
    User->>Authenticator: approve
    Authenticator-->>Browser: public key + attestation
    Browser->>Server: store public key

    note over User,Authenticator: Authentication
    User->>Server: "Sign in with passkey"
    Server-->>Browser: challenge
    Browser->>Authenticator: sign challenge with private key
    Authenticator->>User: biometric prompt
    User->>Authenticator: approve
    Authenticator-->>Browser: signed assertion
    Browser->>Server: verify signature with stored public key
    Server-->>User: authenticated
```

### Implementation (Node.js + SimpleWebAuthn)

```typescript
import { generateRegistrationOptions, verifyRegistrationResponse,
         generateAuthenticationOptions, verifyAuthenticationResponse } from "@simplewebauthn/server";

const RP_ID = "example.com";
const RP_NAME = "My App";

// Step 1: Server generates registration options
app.post("/auth/passkey/register/options", async (req, res) => {
  const user = req.user;
  const options = await generateRegistrationOptions({
    rpName: RP_NAME, rpID: RP_ID,
    userName: user.email,
    attestationType: "none",
    excludeCredentials: user.passkeys.map((pk) => ({ id: pk.credentialId, type: "public-key" })),
  });
  await redis.setex(`challenge:${user.id}`, 60, options.challenge);
  res.json(options);
});

// Step 2: Verify registration response and store credential
app.post("/auth/passkey/register/verify", async (req, res) => {
  const user = req.user;
  const expectedChallenge = await redis.get(`challenge:${user.id}`);
  const { verified, registrationInfo } = await verifyRegistrationResponse({
    response: req.body, expectedChallenge: expectedChallenge!,
    expectedOrigin: "https://example.com", expectedRPID: RP_ID,
  });
  if (verified && registrationInfo) {
    await db.passkeys.create({ userId: user.id, credentialId: registrationInfo.credentialID,
      publicKey: registrationInfo.credentialPublicKey, counter: registrationInfo.counter });
  }
  res.json({ verified });
});
```

**When to use passkeys:** new consumer apps (mobile-first), apps replacing password login, high-security contexts. Not yet universal — keep email/password as fallback for older devices.

## When to Use

- **Firebase Auth** — When you want a managed auth solution with multiple providers, auto token refresh, and easy client SDK integration
- **Google OAuth** — When you need access to Google APIs (Drive, Calendar, Gmail) in addition to authentication
- **GitHub OAuth** — When your app targets developers or needs GitHub repo/org access
- **Custom backend verification** — When you have your own API server and need to verify external tokens before granting access
- **GraphQL integration** — When your API uses GraphQL and needs per-request authentication via context

## Related Topics

- [[JSON Web Tokens]] — The underlying token format used by Firebase and Google
- [[Cryptography Basics]] — Understanding asymmetric cryptography (RS256) for signature verification
- [[HTTP And REST APIs]] — How OAuth flows use HTTP redirects and POST requests
- [[Session Management]] — How to create your own sessions after external auth verification
- [[API Security]] — Broader API security patterns including rate limiting and token revocation

## External Links

- [Firebase Admin SDK — Verify ID Tokens](https://firebase.google.com/docs/auth/admin/verify-id-tokens)
- [Google OpenID Connect Documentation](https://developers.google.com/identity/openid-connect/openid-connect)
- [Google OAuth 2.0 Playground](https://developers.google.com/oauthplayground/)
- [GitHub OAuth Apps Documentation](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps)
- [JWT.io — Debugger and Libraries](https://jwt.io/)
- [RFC 7519 — JSON Web Token](https://tools.ietf.org/html/rfc7519)
- [OpenID Connect Discovery](https://openid.net/specs/openid-connect-discovery-1_0.html)
- [jwks-rsa Library](https://github.com/auth0/node-jwks-rsa)


- [[microservices]] — JWT propagation, OAuth 2.0/OIDC, and service-to-service auth patterns
