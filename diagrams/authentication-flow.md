# Authentication Flow

## 1. Login (Flutter only)

```mermaid
sequenceDiagram
    actor U as User
    participant F as Flutter shell
    participant A as ILAPI
    participant D as PostgreSQL

    U->>F: username + password
    F->>A: POST /api/v1/auth/login
    A->>D: find user by username or email
    D-->>A: user row (bcrypt hash, role)
    A->>A: BCrypt verify
    A->>D: read permissions for the user's role
    A->>D: store SHA-256 of a new refresh token
    A-->>F: success envelope with accessToken (JWT), refreshToken, expiresIn, user (role, permissions)
    F->>F: keep session (browser local storage), build menu from permissions
    F-->>U: go to /dashboard
```

## 2. Session handoff to the embedded Angular module (no token in a URL)

```mermaid
sequenceDiagram
    participant F as Flutter shell
    participant I as Angular module (iframe)
    participant A as ILAPI

    F->>I: iframe src = http://localhost:4200/travel
    I->>F: postMessage {type: il-ready} to parent
    F->>F: accept only if event.origin == Angular origin
    F->>A: POST /api/v1/frontend-session (Flutter access token)
    A-->>F: {token} single use, expires in 60 s, tied to the user
    F->>I: postMessage {type: il-handoff, token} with targetOrigin = Angular origin
    I->>I: accept only from window.parent and an allowed portal origin
    I->>A: POST /api/v1/auth/handoff/exchange {token}
    A-->>I: Angular's own accessToken + refreshToken (token is now used up)
    I->>I: keep session in memory only, router guards continue
```

## 3. Token refresh and failure

```mermaid
flowchart TD
    Req[API call with access token] --> R{401?}
    R -- no --> Done[use response]
    R -- yes --> Ref["POST /api/v1/auth/refresh (once)"]
    Ref --> OK{refresh ok?}
    OK -- yes --> Retry[retry original call]
    OK -- "no, Flutter" --> Out[clear session, back to /login]
    OK -- "no, Angular" --> Re["ask Flutter for a new handoff token"]
    Re --> Fail{handoff ok?}
    Fail -- yes --> Retry
    Fail -- no --> Msg["show 'Session unavailable'"]
```

## 4. Logout

```mermaid
sequenceDiagram
    actor U as User
    participant F as Flutter shell
    participant A as ILAPI
    U->>F: Sign out
    F->>A: POST /api/v1/auth/logout (refresh token)
    A->>A: revoke Flutter's refresh token
    F->>F: delete stored session, router redirects to /login
    F->>F: Travel/Liability iframe is removed with the page (Angular memory session is discarded)
```

Known gap: Angular's own refresh token is not revoked on logout (see `11_Issues_and_Solutions.md`, Issue 7).
