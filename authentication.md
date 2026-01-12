# Authentication

Living Content implements a multi-layer authentication system with a central
auth service, GAIM-scoped access, and anonymous session support.

## Authentication Architecture

```mermaid
flowchart TB
    subgraph clients["Clients"]
        CLI["CLI<br/>(lco-admin, lco-gaim)"]
        BROWSER["Browser<br/>(Frontend)"]
        ANON["Anonymous User"]
    end

    subgraph auth["Authentication Layer"]
        ADC["Application Default<br/>Credentials (GCP)"]
        AUTH_SVC["Central Auth Service<br/>auth.service.livingcontent.co"]
        API_KEY["API Key<br/>(lco_gaim_*)"]
    end

    subgraph identity["Identity"]
        IP["Identity Platform"]
        FS["Firestore<br/>User Profiles"]
    end

    subgraph gaim["GAIM"]
        GAIM_API["GAIM API"]
    end

    CLI --> ADC
    BROWSER --> AUTH_SVC
    ANON --> GAIM_API
    AUTH_SVC --> IP
    AUTH_SVC --> FS
    GAIM_API --> FS
    API_KEY --> GAIM_API
```

## Authentication Layers

| Layer              | Method               | Used By                 | Purpose             |
| ------------------ | -------------------- | ----------------------- | ------------------- |
| Platform CLI       | ADC (gcloud)         | `lco-admin`, `lco-gaim` | Admin operations    |
| Service-to-Service | API Key              | GAIM → TM               | Machine operations  |
| Frontend Users     | Central Auth Service | Browsers                | User authentication |
| Anonymous          | Server-generated ID  | Unauthenticated         | Trial access        |

## Central Auth Service

Single authentication endpoint for all frontend users.

### Sign-In Methods

All authentication is handled server-side via backend redirects. No client-side
JavaScript SDK is required.

```mermaid
graph TB
    subgraph methods["Sign-In Methods"]
        MAGIC["Magic Link<br/>(Passwordless Email)"]
        GOOGLE["Google OAuth"]
    end

    subgraph flow["Server-Side Auth Flow"]
        LOGIN["/login endpoint"]
        GOOGLE_OAUTH["Google OAuth<br/>(server redirect)"]
        IP_API["Identity Platform<br/>REST API"]
        CODE["Auth Code"]
        GAIM["GAIM Callback"]
    end

    GOOGLE --> LOGIN
    LOGIN --> GOOGLE_OAUTH
    GOOGLE_OAUTH --> IP_API
    MAGIC --> LOGIN
    LOGIN --> IP_API
    IP_API --> CODE
    CODE --> GAIM
```

| Method       | Description                |
| ------------ | -------------------------- |
| Magic Link   | Passwordless email sign-in |
| Google OAuth | Google account sign-in     |

### Authentication Flow

```mermaid
sequenceDiagram
    participant FE as GAIM Frontend
    participant Auth as Auth Service
    participant Google as Google OAuth
    participant IP as Identity Platform API
    participant API as GAIM API
    participant FS as Firestore

    FE->>Auth: Redirect to /login<br/>?method=google&redirect_uri=...&gaim_id=...

    alt Magic Link
        Auth->>IP: POST /accounts:sendOobCode
        IP-->>User: Email with link
        User->>Auth: GET /auth/email/callback?oobCode=...
        Auth->>IP: POST /accounts:signInWithEmailLink
    else Google OAuth
        Auth->>Google: Redirect to accounts.google.com
        Google-->>Auth: GET /auth/google/callback?code=...
        Auth->>Google: Exchange code for tokens
        Auth->>IP: POST /accounts:signInWithIdp
    end

    IP-->>Auth: Identity Platform ID Token
    Auth->>Auth: Generate auth code<br/>(JWT, short TTL)
    Auth->>FE: Redirect with code

    FE->>API: POST /auth/exchange<br/>{code}
    API->>API: Validate code signature
    API->>API: Check gaim_id, expiration
    API->>FS: Create/update user profile
    API->>API: Generate session tokens
    API-->>FE: Set session cookies
```

## Token Types

### Auth Code

Short-lived JWT for secure token exchange.

| Property | Value                               |
| -------- | ----------------------------------- |
| Format   | JWT                                 |
| TTL      | Short-lived                         |
| Usage    | One-time                            |
| Contents | `id_token`, `gaim_id`, `jti`, `exp` |

### Identity Platform ID Token

Google-signed JWT containing user identity.

| Property | Value                                      |
| -------- | ------------------------------------------ |
| Format   | JWT (RS256)                                |
| Issuer   | `https://securetoken.google.com/{project}` |
| TTL      | Standard (Identity Platform default)       |
| Contents | `uid`, `email`, `name`, `picture`          |

### API Key

Machine-to-machine authentication for service calls.

| Property | Value                 |
| -------- | --------------------- |
| Format   | `lco_gaim_{random}`   |
| Length   | 48+ characters        |
| Usage    | GAIM → Tenant Manager |

## Session Management

### Session Cookies

```mermaid
flowchart LR
    subgraph cookies["HTTP Cookies"]
        SESSION["gaim_session<br/>Access Token"]
        REFRESH["gaim_refresh<br/>Refresh Token"]
        ANON["anon_user_id<br/>Anonymous ID"]
    end

    subgraph flags["Cookie Flags"]
        HTTP["httpOnly"]
        SECURE["Secure"]
        SAME["SameSite=Lax"]
    end

    cookies --> flags
```

| Cookie      | Contents        | Purpose            |
| ----------- | --------------- | ------------------ |
| `{session}` | GAIM-signed JWT | Access token       |
| `{refresh}` | Opaque token ID | Refresh token      |
| `{anon}`    | Signed anon ID  | Anonymous identity |

### Token Refresh

```mermaid
sequenceDiagram
    participant FE as Frontend
    participant API as GAIM API

    FE->>API: Request (expired session)
    API-->>FE: 401 Unauthorized

    FE->>API: POST /auth/refresh<br/>(refresh cookie)
    API->>API: Validate refresh token
    API->>API: Generate new session
    API-->>FE: New session cookie

    FE->>API: Retry original request
    API-->>FE: Success
```

## Anonymous Access

Server-generated identity for unauthenticated users.

### Anonymous ID Format

```plaintext
anon-{uuid}
```

Example: `anon-550e8400-e29b-41d4-a716-446655440000`

### Anonymous Flow

```mermaid
sequenceDiagram
    participant FE as Frontend
    participant API as GAIM API

    FE->>API: Request (no auth)
    API->>API: Generate anon ID
    API->>API: Sign anon ID
    API-->>FE: Set anon_user_id cookie
    API-->>FE: Response

    Note over FE,API: Subsequent requests use anon cookie
```

### Session Migration

Convert anonymous session to authenticated session.

```mermaid
sequenceDiagram
    participant FE as Frontend
    participant Auth as Auth Service
    participant API as GAIM API
    participant FS as Firestore

    Note over FE: User has anon session with history

    FE->>Auth: Login with anonAccessToken
    Auth-->>FE: Auth code

    FE->>API: POST /auth/exchange<br/>{code, anonAccessToken}
    API->>API: Verify anon ownership
    API->>FS: Migrate session data
    API->>FS: Update user profile
    API-->>FE: New session cookies

    Note over FE: History preserved under new identity
```

## User Roles

Per-GAIM role-based access control.

| Role     | Permissions                           |
| -------- | ------------------------------------- |
| `admin`  | Full GAIM management, user management |
| `editor` | Modify content, manage sessions       |
| `viewer` | Read-only access                      |
| `user`   | Standard end-user access              |

### Role Hierarchy

```mermaid
graph TB
    ADMIN["admin"] --> EDITOR["editor"]
    EDITOR --> VIEWER["viewer"]
    VIEWER --> USER["user"]
```

## Firestore User Schema

Path: `gaims/{gaim_id}/users/{uid}`

```json
{
  "uid": "dent42",
  "email": "arthur@heartofgold.ship",
  "name": "Arthur Dent",
  "picture": "https://guide.galaxy/arthur.jpg",
  "role": "editor",
  "created_at": "2024-01-17T08:00:00Z",
  "last_active": "2024-01-22T16:30:00Z"
}
```

## Secrets Management

### Infrastructure Credentials

Fetched at container startup by init container.

```mermaid
flowchart LR
    SM["Secret Manager"] --> INIT["Init Container"]
    INIT --> VOL["/run/credentials/"]
    VOL --> APP["Application"]
```

| Secret           | Path                       | Purpose      |
| ---------------- | -------------------------- | ------------ |
| Redis password   | `/run/credentials/{redis}` | Redis auth   |
| MongoDB password | `/run/credentials/{mongo}` | MongoDB auth |

### Auth Service Secrets

The central auth service uses these secrets from Secret Manager:

| Secret                  | Purpose                           |
| ----------------------- | --------------------------------- |
| `{signing-key}`         | JWT signing for auth codes        |
| `{identity-api-key}`    | Identity Platform REST API access |
| `{oauth-client-id}`     | Google OAuth client ID            |
| `{oauth-client-secret}` | Google OAuth client secret        |

### Application Secrets

Loaded at runtime from Firestore with envelope encryption.

```mermaid
flowchart LR
    FS["Firestore<br/>Encrypted Secrets"] --> DEK["DEK<br/>(Secret Manager)"]
    DEK --> KMS["Cloud KMS"]
    KMS --> APP["Decrypted Secret"]
```

| Feature        | Description          |
| -------------- | -------------------- |
| Encryption     | Fernet (symmetric)   |
| Key Storage    | Secret Manager (DEK) |
| Key Protection | Cloud KMS            |
| Hot Reload     | Yes (async runtime)  |

## CLI Authentication

### Application Default Credentials

```bash
# Authenticate CLI
gcloud auth application-default login

# CLI uses ADC automatically
lco-admin tenant list
lco-gaim config get
```

### API Key Generation

```bash
# Generate GAIM API key
lco-admin gaim token create --gaim-name=marvin

# Output: lco_gaim_xxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

## Middleware Stack

Request processing order in GAIM API:

```mermaid
flowchart TB
    REQ["Incoming Request"] --> INIT["InitializationMiddleware<br/>App readiness"]
    INIT --> AUTH["AuthMiddleware<br/>Token validation"]
    AUTH --> RATE["RateLimitMiddleware<br/>User + IP limits"]
    RATE --> SEC["SecurityHeadersMiddleware<br/>CSP, X-Frame-Options"]
    SEC --> METRICS["MetricsMiddleware<br/>Request timing"]
    METRICS --> CORS["CustomCORSMiddleware<br/>CORS handling"]
    CORS --> HANDLER["Route Handler"]
```

## Rate Limiting

Per-endpoint limits enforced at middleware layer. Limits vary by endpoint
sensitivity.

## Related Documentation

- [Platform Overview](platform-overview.md) - Architecture context
- [Infrastructure Layer](infrastructure-layer.md) - Auth Service details
- [Application Layer](application-layer.md) - GAIM auth endpoints
- [Networking](networking.md) - Auth routing
