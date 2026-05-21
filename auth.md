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

MD5 and SHA-256 were designed to be fast — millions of hashes per second on modern hardware. An attacker with a GPU can try billions of password guesses per second against a stolen hash.

### The Right Tool — bcrypt

bcrypt was designed specifically for passwords. It has a **cost factor** that controls how much computation is needed:

```
cost = 10  →  ~100ms per hash
cost = 12  →  ~400ms per hash
cost = 14  →  ~1.5s per hash
```

100ms feels instant to a user logging in. For an attacker trying a billion guesses — suddenly impossible.

**The salt is built in.** bcrypt generates and stores the salt automatically:

```
bcrypt("sunshine99")  →  "$2b$10$xQ7kL...hashed..."
                              ↑    ↑
                           algo  cost factor + salt + hash all in one string
```

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
Request 2:  "Show me my bookings"                       →  Server: "...who are you?"
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

The server looks up `abc123` in its session store (database or Redis). This is **stateful** — the server holds the state.

### Approach 2 — Token in Header

After login, the server issues a token. The client stores it and manually sends it with every request.

```
Login response:
  { "token": "eyJhbGc..." }

Every subsequent request client sends manually:
  Authorization: Bearer eyJhbGc...
```

The server verifies the token itself — no lookup needed. This is **stateless** — the server holds nothing.

### Cookie Properties Worth Knowing

| Flag | What it does |
|---|---|
| `HttpOnly` | JavaScript cannot read this cookie — blocks XSS theft |
| `Secure` | Only sent over HTTPS |
| `SameSite=Strict` | Only sent on same-site requests — blocks CSRF |
| `Max-Age` | How long until it expires |

### Where Tokens Are Stored (Client Side)

| Storage | XSS risk | CSRF risk | Notes |
|---|---|---|---|
| localStorage | ✗ High | ✓ None | JS can read it — malicious scripts can steal it |
| sessionStorage | ✗ High | ✓ None | Same as localStorage, cleared on tab close |
| HttpOnly Cookie | ✓ None | ✗ Needs mitigation | Best of both worlds |
| Memory (JS variable) | ✓ None | ✓ None | Gone on page refresh |

A common pattern: store the token in an **HttpOnly cookie** even though it's a token-based system. You get the security of cookies with the flexibility of tokens.

---

## Layer 1, Topic 3 — What a JWT Is

**One sentence:** A JWT (JSON Web Token) is a self-contained, signed package of claims that the server issues and the client presents on every request.

### Structure

```
HEADER . PAYLOAD . SIGNATURE
```

**Chunk 1 — Header**
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

**Chunk 2 — Payload**
```json
{
  "userId": 42,
  "email": "mario@example.com",
  "role": "guest",
  "iat": 1716239022,
  "exp": 1716242622
}
```

> The payload is **base64-encoded — not encrypted**. Anyone can decode and read it. Never put passwords, card numbers, or secrets in a JWT payload.

**Chunk 3 — Signature**
```
HMAC-SHA256(
  base64(header) + "." + base64(payload),
  SECRET_KEY
)
```

### Why the Signature Matters

If an attacker tampers with the payload and re-encodes it, the server recomputes the signature:

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
| Best for | Single server / monolith | Microservices, third-party verification |

### Access Token vs Refresh Token

- **Access token** — short-lived (15min–1hr), used on every request
- **Refresh token** — long-lived (days/weeks), used only to get a new access token

---

## Layer 2, Topic 4 — Registration & Login Flow

### Registration — The Happy Path

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

**Step 1 — Input Validation**
- Email: present? valid format? max length?
- Password: present? min 8 chars? **max 72 chars** (bcrypt silently truncates beyond this)

**Step 2 — Check Email Uniqueness**

Use neutral messaging to prevent account enumeration:
```
❌  "This email is already registered"
✓   "If this email is available, you'll receive a confirmation"
```

**Step 3 — Hash the Password**

```java
String hash = passwordEncoder.encode(password);
// BCryptPasswordEncoder with cost factor 12 = ~400ms
```

Never log the plaintext password anywhere.

**Step 4 — DB Schema**

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

### Login — The Happy Path

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

### Error Messages — The Golden Rule

```
❌  "No account found with that email"
❌  "Wrong password"
✓   "Invalid email or password"
```

Same message, always, regardless of which credential failed.

### Brute Force Protection

| Technique | How it works |
|---|---|
| Rate limiting | Max N attempts per IP per minute |
| Account lockout | Lock account after N failed attempts |
| Exponential backoff | Each failure adds delay |
| CAPTCHA | Trigger after N failures |

---

## Layer 2, Topic 5 — Refresh Tokens

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

### Refresh Token DB Schema

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

### Token Rotation

Every time a refresh token is used, it is replaced with a new one.

```
Client uses refreshToken_A  →  POST /auth/refresh
Server:
  1. Look up refreshToken_A in DB
  2. Verify it's valid, not revoked, not expired
  3. Mark refreshToken_A as REVOKED
  4. Issue refreshToken_B + new accessToken
  5. Return both to client
```

### Refresh Token Reuse Detection

```
Attacker stole refreshToken_A before rotation
User already rotated: A → B → C
Attacker now tries refreshToken_A

Server sees: refreshToken_A was already used and revoked
→ REVOKE THE ENTIRE FAMILY
→ force user to log in again
```

### Where to Store Tokens on the Client

```
accessToken   →  JavaScript memory (variable)
refreshToken  →  HttpOnly cookie
```

XSS attacks can't steal either token; access is in memory, refresh is HttpOnly.

---

## Layer 2, Topic 6 — Logout & Token Revocation

### Why Logout is Harder Than It Seems

With sessions, logout is trivial:
```sql
DELETE FROM sessions WHERE id = $1
-- done. token is dead instantly.
```

With JWT, the server holds no state. The token remains **cryptographically valid until it expires** even after the client deletes it.

### What "Logout" Actually Means

1. **Client-side cleanup** — delete accessToken from memory, delete refreshToken cookie
2. **Refresh token revocation** — mark refreshToken as revoked in DB
3. **Access token invalidation** — the hard part

### Solution — The Token Blocklist

When instant revocation is required, maintain a blocklist in Redis:

```java
// On logout — store jti with TTL = remaining token lifetime
// On every request — check if jti is in blocklist
```

`jti` is a unique ID per token (standard JWT claim). Redis auto-deletes entries when they expire, keeping the blocklist small.

### The Tradeoff Spectrum

```
FULLY STATELESS                              FULLY STATEFUL
     |────────────────────────────────────────────|
  No blocklist                            Full blocklist
  No revocation                           Instant revocation
  Fastest performance                     DB/Redis hit every request
  Acceptable for most apps                Required for high-security apps
```

For miniAgoda:
- Regular guest endpoints → no blocklist, short expiry is fine
- Admin / payment endpoints → check blocklist on every request

### Revocation Approaches Summary

| Approach | Revocation speed | Performance cost | Complexity |
|---|---|---|---|
| Short expiry only | At expiry (minutes) | None | Low |
| Refresh token revocation | Stops new access tokens | Low | Low |
| Redis blocklist (jti) | Instant | Small (Redis lookup) | Medium |
| DB user status check | ~instant | Medium | Low |

### Logout Endpoint Logic

```
POST /auth/logout

Server:
  1. Verify accessToken
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

---

## Layer 3, Topic 7 — Protecting Routes with Middleware

In Spring Boot, the "middleware" concept maps to a **JWT filter** + **Security configuration**.

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
```

---

## Layer 3, Topic 8 — Resource Ownership

### The Problem Route Protection Doesn't Solve

Route protection only checks **authentication**. It doesn't check **ownership**.

```
Mario  logs in → token { userId: 42 }
Luigi  logs in → token { userId: 99 }

Mario sends:  DELETE /bookings/77   ← Luigi's booking
Server check: is user authenticated?  ✓
Server:       deletes Luigi's booking  ✗
```

This is called **BOLA** (Broken Object Level Authorization) / **IDOR** (Insecure Direct Object Reference) — the #1 API security vulnerability.

### Approach 1 — Check in Service Layer

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

```java
// Repository
public interface BookingRepository extends JpaRepository<Booking, Long> {
    Optional<Booking> findByIdAndUserId(Long id, Long userId);
}

// Service
public void cancelBooking(Long bookingId, Long requestingUserId) {
    Booking booking = bookingRepository
        .findByIdAndUserId(bookingId, requestingUserId)
        .orElseThrow(() -> new AccessDeniedException("Booking not found or access denied"));

    booking.setStatus(BookingStatus.CANCELLED);
    bookingRepository.save(booking);
}
```

### Approach 3 — @PreAuthorize with SpEL

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

```java
@Configuration
@EnableMethodSecurity   // ← enables @PreAuthorize
public class SecurityConfig { ... }
```

### Admin Override Pattern

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

// Or with @PreAuthorize
@PreAuthorize("hasRole('ADMIN') or @bookingAuthz.isOwner(#bookingId, authentication)")
public void cancelBooking(Long bookingId) { ... }
```

### The Error Response — Don't Leak Information

```java
// ❌ Leaks that the resource exists but user doesn't own it
throw new AccessDeniedException("You do not own this booking");   // 403

// ✓ Reveals nothing
throw new AccessDeniedException("Booking not found or access denied");  // 404
```

### Custom Exception Handler

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

---

## Layer 3, Topic 9 — Roles & Permissions

### RBAC — Role Based Access Control

```
User ──→ Role ──→ Permissions
mario    GUEST    [BOOKING_CREATE, BOOKING_CANCEL_OWN, PAYMENT_CREATE]
kenji    MANAGER  [HOTEL_MANAGE_OWN, BOOKING_VIEW_OWN_HOTEL]
sarah    ADMIN    [*]
```

### Step 1 — Define Roles and Permissions

```java
public enum Role {
    GUEST,
    MANAGER,
    ADMIN
}

public enum Permission {
    BOOKING_CREATE,
    BOOKING_VIEW_OWN,
    BOOKING_CANCEL_OWN,
    BOOKING_VIEW_ALL,
    BOOKING_CANCEL_ANY,
    HOTEL_VIEW,
    HOTEL_MANAGE_OWN,
    HOTEL_MANAGE_ANY,
    PAYMENT_CREATE,
    PAYMENT_VIEW_OWN,
    PAYMENT_REFUND_OWN,
    PAYMENT_REFUND_ANY,
    USER_VIEW_ANY,
    USER_SUSPEND
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
        Permission.BOOKING_VIEW_OWN,
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

    public List<SimpleGrantedAuthority> getAuthorities() {
        List<SimpleGrantedAuthority> authorities = new ArrayList<>();
        permissions.stream()
            .map(p -> new SimpleGrantedAuthority(p.name()))
            .forEach(authorities::add);
        authorities.add(new SimpleGrantedAuthority("ROLE_" + this.name()));
        return authorities;
    }
}
```

### Step 3 — Load Authorities into SecurityContext

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

```java
@RestController
@RequestMapping("/bookings")
public class BookingController {

    @PostMapping
    @PreAuthorize("hasAuthority('BOOKING_CREATE')")
    public ResponseEntity<BookingResponse> createBooking(...) { ... }

    @GetMapping("/{id}")
    @PreAuthorize("hasAuthority('BOOKING_VIEW_ALL') or " +
                  "@bookingAuthz.isOwner(#id, authentication)")
    public ResponseEntity<BookingResponse> getBooking(@PathVariable Long id, ...) { ... }

    @DeleteMapping("/{id}")
    @PreAuthorize("hasAuthority('BOOKING_CANCEL_ANY') or " +
                  "(hasAuthority('BOOKING_CANCEL_OWN') and " +
                  "@bookingAuthz.isOwner(#id, authentication))")
    public ResponseEntity<Void> cancelBooking(@PathVariable Long id, ...) { ... }
}
```

### Step 5 — DB Schema for Roles

```sql
ALTER TABLE users ADD COLUMN role VARCHAR(50) DEFAULT 'GUEST';

CREATE TABLE hotel_managers (
    user_id   BIGINT REFERENCES users(id),
    hotel_id  BIGINT REFERENCES hotels(id),
    PRIMARY KEY (user_id, hotel_id)
);
```

### Manager Ownership Check

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

> **Important:** If a user's role changes, their existing token still carries the old role until it expires. Short-lived access tokens are your safety net. For immediate role updates, use the blocklist to invalidate their current token.

---

## Layer 4, Topic 10 — OAuth / Social Login

### What OAuth Actually Is

- **OAuth 2.0** handles the authorization flow
- **OpenID Connect (OIDC)** adds the identity layer — who the user actually is

You use both together. The user's Google password **never touches miniAgoda**.

### The Flow

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

### Step 1 — Dependencies

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```

### Step 2 — Configuration

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

### Step 4 — OAuth2 Success Handler

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

### Account Linking

If a user previously registered with email/password and now tries "Login with Google" with the same email:

- **Option A — Auto-link:** risky — email spoofing possible
- **Option B — Prompt to link:** verify ownership first (recommended for miniAgoda)
- **Option C — Separate accounts:** simplest, confusing UX

---

## Layer 4, Topic 11 — Common Attacks & Defenses

### Attack 1 — Brute Force

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

```java
// In AuthService
public AuthResponse login(LoginRequest request, String ip) {
    rateLimiter.checkAndIncrement(request.getEmail(), ip);
    // ... rest of login logic
}

// In AuthController
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

**Defense — Have I Been Pwned API (k-anonymity)**

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

**Sub-attack A — Algorithm Confusion (`alg:none`)**

```java
// Always explicitly specify allowed algorithms — modern JJWT rejects "none" by default
Jwts.parser()
    .verifyWith(getSigningKey())
    .build()
    .parseSignedClaims(token);
```

Never use outdated JWT libraries. JJWT 0.12+ is safe.

**Sub-attack B — RS256 to HS256 Confusion**

Always explicitly verify with the expected algorithm and key type. Modern libraries handle this, but understanding the attack matters.

### Attack 4 — Token Theft via XSS

**Defense — HttpOnly Cookies**

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

### Attack 5 — CSRF

**Defense A — SameSite Cookie Flag**

```java
.sameSite("Strict")  // cookie only sent on same-site requests
```

**Defense B — CSRF Token**

```java
http.csrf(csrf -> csrf
    .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
);
```

### Attack 6 — Account Enumeration

Always the same response, same timing, for login, register, and password reset:

```java
return ResponseEntity
    .status(HttpStatus.UNAUTHORIZED)
    .body(new ErrorResponse("Invalid email or password"));
```

Password reset:
```
❌  "No account found with that email"
✓   "If that email is registered, you'll receive a reset link"
```

### Attack 7 — Mass Assignment

**Defense — Explicit DTOs**

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

---

## Layer 4, Topic 12 — Fitting Auth into miniAgoda

### The Full System Map

```
┌─────────────────────────────────────────────────────────────┐
│                        miniAgoda API                        │
│                                                             │
│  PUBLIC                  AUTHENTICATED         ADMIN ONLY   │
│  GET  /hotels            POST /bookings        GET  /admin/users     │
│  POST /auth/register     GET  /bookings/my     POST /admin/refunds   │
│  POST /auth/login        DELETE /bookings/:id  PUT  /admin/hotels    │
│  POST /auth/refresh      POST /payments                     │
│  POST /auth/oauth2/**    POST /payments/refund              │
│  POST /auth/logout       GET  /users/me                     │
└─────────────────────────────────────────────────────────────┘
```

### Complete Auth Flow — Registration to Booking

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

```java
// 1. JwtAuthFilter — extracts token, verifies, checks blocklist, sets SecurityContext
// 2. SecurityConfig route rules — public / authenticated / role-restricted
// 3. @PreAuthorize on controller method — fine-grained permission check
// 4. Ownership check inside service — does this user own this specific resource?
// 5. Business logic — actually do the thing
```

### Booking Controller — After Auth

```java
@RestController
@RequestMapping("/bookings")
@RequiredArgsConstructor
public class BookingController {

    private final BookingService bookingService;

    @PostMapping
    @PreAuthorize("hasAuthority('BOOKING_CREATE')")
    public ResponseEntity<BookingResponse> createBooking(
            @RequestBody  @Valid BookingRequest request,
            @AuthenticationPrincipal UserDetails userDetails
    ) {
        Long userId = Long.parseLong(userDetails.getUsername());
        return ResponseEntity.ok(bookingService.create(userId, request));
    }

    @GetMapping("/my")
    @PreAuthorize("hasAuthority('BOOKING_VIEW_OWN')")
    public ResponseEntity<List<BookingResponse>> getMyBookings(
            @AuthenticationPrincipal UserDetails userDetails
    ) {
        Long userId = Long.parseLong(userDetails.getUsername());
        return ResponseEntity.ok(bookingService.findByUser(userId));
    }

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

    private final AuthService authService;

    @PostMapping("/register")
    public ResponseEntity<Void> register(
            @RequestBody @Valid RegisterRequest request
    ) {
        authService.register(request);
        return ResponseEntity.status(HttpStatus.CREATED).build();
    }

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

    @GetMapping("/me")
    public ResponseEntity<UserResponse> me(
            @AuthenticationPrincipal UserDetails userDetails
    ) {
        Long userId = Long.parseLong(userDetails.getUsername());
        return ResponseEntity.ok(authService.getMe(userId));
    }

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
CREATE TABLE users (
    id            BIGSERIAL PRIMARY KEY,
    email         VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255),
    name          VARCHAR(255),
    picture       VARCHAR(500),
    role          VARCHAR(50) DEFAULT 'GUEST',
    provider      VARCHAR(50) DEFAULT 'LOCAL',
    provider_id   VARCHAR(255),
    suspended     BOOLEAN DEFAULT FALSE,
    created_at    TIMESTAMP DEFAULT NOW(),
    last_login_at TIMESTAMP
);

CREATE UNIQUE INDEX idx_users_provider
    ON users(provider, provider_id)
    WHERE provider_id IS NOT NULL;

CREATE TABLE refresh_tokens (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT REFERENCES users(id) ON DELETE CASCADE,
    token_hash  VARCHAR(255) NOT NULL,
    family_id   UUID NOT NULL,
    expires_at  TIMESTAMP NOT NULL,
    revoked     BOOLEAN DEFAULT FALSE,
    created_at  TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_refresh_tokens_user ON refresh_tokens(user_id);

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
