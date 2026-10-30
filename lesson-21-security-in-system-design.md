# Lesson 21 — Security in System Design: Authentication, Authorization, Encryption, Rate Limiting, and API Security

Security is not something we add at the end of development. It must be part of the system architecture from the beginning.

In this lesson, we'll explore authentication, authorization, encryption, API security, rate limiting, secrets management, and common security threats—with examples from our ride-booking platform.

## 1. Why Security Matters in System Design

Imagine we are building a ride-booking platform.

The system stores:
- Passenger information.
- Driver information.
- Phone numbers and email addresses.
- Pickup and drop-off locations.
- Booking history.
- Payment information.

A security failure could expose personal information, allow unauthorized bookings, or let attackers take control of user accounts.

Security must protect three fundamental properties:

### CIA Triad

| Property | Meaning | Example |
|---|---|---|
| Confidentiality | Only authorized parties can access data | A passenger cannot see another passenger's personal details |
| Integrity | Data cannot be modified without authorization | A driver cannot change a completed booking's fare |
| Availability | Systems remain accessible to legitimate users | Attackers cannot easily take down the booking API |

These three principles form the foundation of information security.

---

## 2. Authentication vs Authorization

These are two of the most important concepts in application security.

### Authentication — Who are you?

Authentication verifies the identity of a user or service.

Examples:
- Email and password.
- OTP verification.
- Social login.
- Multi-factor authentication.
- Service-to-service credentials.

### Authorization — What are you allowed to do?

Authorization determines what an authenticated user can access or perform.

Example:

A passenger logs in successfully.

Authentication confirms:

```text
User: Passenger-123
Authenticated: Yes
```

Authorization determines:

```text
Can view own bookings: Yes
Can view another passenger's bookings: No
Can assign drivers: No
Can access admin dashboard: No
```

**Remember: Authentication establishes identity; authorization enforces permissions.**

A user being authenticated does not mean they can access every resource.

---

## 3. Authentication Architecture

A typical authentication flow looks like this:

```text
Client
  |
  | 1. Login credentials
  v
Authentication Service
  |
  | 2. Verify credentials
  v
User Database
  |
  | 3. Valid user
  v
Authentication Service
  |
  | 4. Issue access token
  v
Client
  |
  | 5. API request + token
  v
API Gateway / Backend
  |
  | 6. Validate token
  v
Protected Resource
```

### 3.1 Password Security

Never store passwords as plain text.

Instead, store a password hash created using a password-hashing algorithm.

Common choices:
- Argon2id.
- bcrypt.
- scrypt.

Password hashing should use a unique salt and an appropriate work factor.

Why not use ordinary SHA-256?

SHA-256 is designed to be fast. Attackers can test many password guesses quickly if they obtain the password database.

Password-hashing algorithms are intentionally more expensive to slow down brute-force attacks.

### 3.2 Multi-Factor Authentication (MFA)

MFA requires two or more independent authentication factors.

Common factors:

| Factor | Example |
|---|---|
| Something you know | Password or PIN |
| Something you have | Security key or authenticator device |
| Something you are | Fingerprint or facial recognition |

MFA reduces the risk of account takeover when passwords are compromised.

---

## 4. Session-Based Authentication vs Token-Based Authentication

Two common approaches are session-based authentication and token-based authentication.

### 4.1 Session-Based Authentication

The server creates a session after successful login.

```text
Client
  |
  | Login
  v
Authentication Server
  |
  | Create session
  v
Session Store (Redis)
  |
  | Session ID
  v
Client Cookie
```

On subsequent requests:

```text
Client
  |
  | Cookie: session_id
  v
Server
  |
  | Look up session
  v
Redis
  |
  v
Authenticated Request
```

Advantages:
- Centralized session management.
- Easy session revocation.
- Server controls session lifetime.

Disadvantages:
- Requires session storage.
- Session lookup adds infrastructure dependency.
- Session replication and availability must be handled.

### 4.2 Token-Based Authentication

The server issues a signed token after authentication.

The client sends the token with subsequent requests.

```text
Client
  |
  | Authorization: Bearer <token>
  v
API Gateway
  |
  | Verify signature and claims
  v
Backend Service
```

Advantages:
- Services can validate tokens without a central session lookup.
- Useful for distributed architectures.
- Can carry identity and authorization claims.

Disadvantages:
- Revocation is more complicated for self-contained tokens.
- Token theft can allow account access until expiry or revocation.
- Incorrect validation can create serious vulnerabilities.

**Important:** JWT is a token format, not an authentication strategy by itself.

---

## 5. JWT — JSON Web Token

A JWT commonly consists of three parts:

```text
HEADER.PAYLOAD.SIGNATURE
```

Example:

```text
xxxxx.yyyyy.zzzzz
```

### Header

Contains token metadata, such as the signing algorithm.

### Payload

Contains claims.

Example:

```json
{
  "sub": "passenger-123",
  "role": "PASSENGER",
  "iss": "auth.example.com",
  "aud": "booking-api",
  "exp": 1793376000
}
```

Common claims:

| Claim | Purpose |
|---|---|
| sub | Subject or user identifier |
| iss | Token issuer |
| aud | Intended audience |
| exp | Expiration time |
| iat | Issued-at time |
| nbf | Not valid before |

### Signature

The signature protects the token's integrity.

The server verifies the signature and validates relevant claims before trusting the token.

### Important JWT Security Rules

1. Validate the signature.
2. Allow only explicitly configured signing algorithms.
3. Validate issuer and audience.
4. Check expiration and not-before times.
5. Use short-lived access tokens where appropriate.
6. Protect signing keys.
7. Never put passwords or secrets in the payload.
8. Do not assume the payload is encrypted.

A signed JWT is normally readable by anyone who possesses it. Signing prevents undetected modification; it does not provide confidentiality.

---

## 6. Access Tokens and Refresh Tokens

Most token-based authentication systems use two types of tokens.

### Access Token

- Used to access protected APIs.
- Usually short-lived.
- Sent with API requests.

### Refresh Token

- Used to obtain a new access token.
- Usually longer-lived.
- Must be protected carefully.

Example:

```text
Login
  |
  v
Access Token + Refresh Token
  |
  v
Access Token Expires
  |
  v
Client Sends Refresh Token
  |
  v
Authentication Service Validates It
  |
  v
New Access Token
```

### Refresh Token Rotation

A secure design may rotate refresh tokens:

1. Client sends the current refresh token.
2. Server validates it.
3. Server issues a new refresh token.
4. Previous refresh token is invalidated.
5. Reuse of an invalidated token can trigger token-family revocation or additional security checks.

This helps detect refresh-token theft.

### Where Should Tokens Be Stored?

For browser applications, storage choices involve security and usability trade-offs.

| Storage | Main Consideration |
|---|---|
| HttpOnly Secure Cookie | JavaScript cannot read it, reducing token theft through XSS |
| localStorage | Accessible to JavaScript, so XSS can steal tokens |
| sessionStorage | Also accessible to JavaScript; lifetime is generally tab-scoped |
| In-memory storage | Less persistent, but can be lost on refresh |

Cookies require appropriate settings, such as:
- HttpOnly.
- Secure.
- SameSite.
- Appropriate Path and Domain restrictions.

Cookies also require CSRF protection appropriate to the application architecture.

There is no universally perfect storage choice. Consider XSS, CSRF, cross-origin requirements, and the application's threat model.

---

## 7. Authorization Models

After authentication, the system must decide what a user can do.

Three common authorization models are:

### 7.1 RBAC — Role-Based Access Control

Permissions are assigned to roles.

Example:

```text
ADMIN
  ├── Manage Users
  ├── Manage Drivers
  ├── View All Bookings
  └── Manage Operations

DISPATCHER
  ├── View Bookings
  ├── Assign Drivers
  └── Manage Routes

DRIVER
  ├── View Assigned Trips
  └── Update Trip Status

PASSENGER
  ├── Create Booking
  └── View Own Bookings
```

The user receives permissions through their role.

Advantages:
- Easy to understand.
- Easy to manage for well-defined organizational roles.

Disadvantages:
- Can become complicated when permissions depend on resource ownership or context.

### 7.2 ABAC — Attribute-Based Access Control

Authorization decisions depend on attributes.

Example:

```text
Allow access if:

user.role == "DRIVER"
AND
booking.assignedDriverId == user.id
AND
booking.status != "CANCELLED"
```

Attributes can describe:
- User.
- Resource.
- Action.
- Environment.

ABAC is useful when access depends on multiple conditions.

### 7.3 ReBAC — Relationship-Based Access Control

Permissions depend on relationships between entities.

Example:

```text
Passenger-123
   |
   | owns
   v
Booking-456
```

A passenger can access a booking because they own it.

This model is useful for collaboration platforms, organizations, shared resources, and hierarchical permissions.

### Comparison

| Model | Authorization Based On |
|---|---|
| RBAC | User roles |
| ABAC | Attributes and policies |
| ReBAC | Relationships between entities |

These models can be combined.

---

## 8. Object-Level Authorization and IDOR

One of the most common API security mistakes is checking whether a user is logged in but not checking whether they own the requested resource.

Consider:

```http
GET /api/bookings/1001
```

The server checks that the user is authenticated.

But what if booking `1001` belongs to another passenger?

If the API returns the booking anyway, this is an authorization vulnerability commonly associated with Insecure Direct Object Reference (IDOR).

A safer design:

```sql
SELECT *
FROM bookings
WHERE id = :booking_id
  AND passenger_id = :authenticated_user_id;
```

The user identity must come from trusted authentication context, not from a client-controlled `passengerId` parameter.

For administrative or dispatcher operations, apply the appropriate role, tenant, service-area, and resource-level authorization checks.

**Never rely on unpredictable IDs as an authorization mechanism.**

---

## 9. API Security

APIs are common entry points into a system.

Important security controls include:

### 9.1 Input Validation

Validate:
- Data types.
- Required fields.
- Maximum lengths.
- Allowed values.
- Numeric ranges.
- File types and file sizes.

Never trust client input.

### 9.2 SQL Injection

Unsafe query construction:

```sql
SELECT * FROM users
WHERE email = 'INPUT_FROM_USER';
```

If user input is directly concatenated into SQL, attackers may manipulate the query.

Use parameterized queries or safe ORM query mechanisms.

### 9.3 Broken Object-Level Authorization

Ensure users can access only the resources they are authorized to access.

### 9.4 Mass Assignment

Avoid blindly copying request fields into database models.

For example, a passenger should not be allowed to submit:

```json
{
  "name": "John",
  "role": "ADMIN",
  "isVerified": true
}
```

Use explicit request DTOs and allowlisted fields.

### 9.5 Sensitive Data Exposure

- Return only necessary fields.
- Avoid exposing internal identifiers unnecessarily.
- Redact sensitive data from logs.
- Use encryption for sensitive information where appropriate.

### 9.6 CORS

Cross-Origin Resource Sharing controls which browser origins can read responses from an API.

CORS is not authentication or authorization.

It does not prevent direct requests from scripts, command-line clients, or malicious servers.

Configure:
- Explicit allowed origins.
- Required methods.
- Required headers.
- Credential behavior.

Avoid wildcard origins when credentialed cross-origin requests are involved.

---

## 10. Rate Limiting and Abuse Prevention

Rate limiting restricts how frequently a client can make requests.

It protects against:
- Brute-force login attempts.
- OTP abuse.
- API scraping.
- Excessive resource consumption.
- Some forms of denial-of-service traffic.

### Common Rate-Limiting Algorithms

#### Fixed Window

Allow a maximum number of requests in a fixed time window.

Example:

```text
100 requests per minute
```

Simple, but traffic can burst at window boundaries.

#### Sliding Window Log

Store request timestamps and count those within the rolling window.

More accurate, but can require significant storage.

#### Sliding Window Counter

Combines the previous and current fixed-window counts using a weighted estimate.

A practical compromise between accuracy and storage.

#### Token Bucket

Tokens are added to a bucket at a fixed rate.

Each request consumes a token.

If no tokens remain, the request is rejected or delayed.

This permits controlled bursts.

#### Leaky Bucket

Requests enter a queue and are processed at a controlled rate.

This smooths traffic, but can introduce waiting time and queue management complexity.

### Example

Suppose an OTP API allows:

```text
Maximum: 5 OTP requests
Window: 15 minutes
```

Rate limits should often be applied using multiple dimensions:
- Account.
- IP address.
- Device or client.
- Phone number.
- API endpoint.

Be careful with IP-only limits because many legitimate users may share an IP address.

Return HTTP `429 Too Many Requests` when a request exceeds its limit. Consider a `Retry-After` header where useful.

---

## 11. Encryption: At Rest and In Transit

Encryption protects data from unauthorized access.

### 11.1 Encryption in Transit

Protects data while moving across networks.

Use TLS for:
- Browser to API Gateway.
- API Gateway to backend services.
- Service-to-service communication.
- Database connections where required.
- Message broker connections.

HTTPS uses TLS to protect HTTP communication.

For internal services, do not assume a private network is automatically trusted.

### 11.2 Encryption at Rest

Protects stored data.

Examples:
- Database encryption.
- Disk or volume encryption.
- Encrypted object storage.
- Encrypted backups.

Encryption at rest protects against certain storage-level threats, but does not automatically protect against an attacker who has legitimate application-level access.

### 11.3 Hashing vs Encryption

| Hashing | Encryption |
|---|---|
| One-way transformation | Reversible with the correct key |
| Used for password verification | Used to protect data confidentiality |
| No decryption operation | Requires decryption |
| Example: Argon2id | Example: AES-GCM |

Do not encrypt passwords when you should be hashing them.

---

## 12. Secrets Management

Applications often need secrets such as:
- Database passwords.
- API keys.
- JWT signing keys.
- Cloud credentials.
- Third-party service tokens.

Never commit secrets to Git.

Avoid:
- Hardcoding secrets in source code.
- Putting secrets in container images.
- Exposing secrets in logs.
- Passing secrets through insecure channels.

Prefer a dedicated secrets manager or secure deployment-time secret injection.

Examples of secret-management systems include:
- HashiCorp Vault.
- AWS Secrets Manager.
- Azure Key Vault.
- Google Secret Manager.

Important practices:

1. Restrict access to secrets.
2. Rotate credentials periodically and after suspected exposure.
3. Use separate secrets for development, QA, and production.
4. Prefer short-lived credentials where possible.
5. Audit secret access.
6. Avoid sharing one credential across unrelated services.

---

## 13. Network Security and Zero Trust

Traditional network security often assumes that internal traffic is trusted.

But in a distributed system, a compromised service may be used to attack other internal services.

Zero Trust follows the principle:

**Never trust automatically; verify explicitly.**

Common principles:
- Authenticate service identities.
- Authorize each service interaction.
- Use least-privilege access.
- Encrypt service-to-service communication.
- Monitor internal traffic.
- Continuously validate access.

### Network Segmentation

Separate systems into logical network zones.

Example:

```text
Internet
   |
   v
Public Load Balancer
   |
   v
Application Network
   |
   +------> Booking Service
   |
   +------> Driver Service
   |
   v
Private Data Network
   |
   +------> PostgreSQL
   |
   +------> Redis
   |
   +------> Message Broker
```

The database should not be directly accessible from the public internet.

Only authorized services should be able to connect to sensitive infrastructure.

---

## 14. Common Web and API Security Threats

### 14.1 XSS — Cross-Site Scripting

Attackers inject malicious scripts into content displayed by a browser.

Potential impact:
- Stealing accessible tokens.
- Performing actions as the victim.
- Reading sensitive page content.

Mitigations:
- Context-aware output encoding.
- Safe framework rendering.
- Avoid unsafe HTML injection.
- Content Security Policy.
- Sanitize untrusted HTML when HTML rendering is necessary.

### 14.2 CSRF — Cross-Site Request Forgery

An attacker tricks a user's browser into sending an unwanted request to a site where the user is already authenticated.

Relevant especially to cookie-based authentication.

Mitigations:
- CSRF tokens.
- SameSite cookie settings.
- Origin and Referer validation where appropriate.
- Avoid unsafe state-changing GET requests.

### 14.3 SSRF — Server-Side Request Forgery

An attacker tricks a server into making requests to unintended destinations.

Example:

An API accepts a URL to retrieve an image. An attacker submits an internal metadata-service URL.

Mitigations:
- Allowlist permitted destinations.
- Block private and internal IP ranges where appropriate.
- Validate DNS resolution and redirects.
- Restrict outbound network access.

### 14.4 DDoS — Distributed Denial of Service

Attackers generate excessive traffic to exhaust system resources.

Mitigations:
- CDN and edge protection.
- DDoS protection services.
- Rate limiting.
- Connection limits.
- Autoscaling with safeguards.
- Load shedding.

### 14.5 Brute Force and Credential Stuffing

Attackers repeatedly try passwords or use credentials leaked from other services.

Mitigations:
- MFA.
- Rate limiting.
- Password breach detection.
- Login anomaly detection.
- Progressive delays or risk-based challenges.
- Secure account recovery.

### 14.6 Broken Access Control

Users gain access to actions or resources they should not be allowed to access.

Mitigations:
- Server-side authorization.
- Resource ownership checks.
- Least privilege.
- Regular access-control testing.

---

## 15. Security in Microservices

Microservices introduce additional security boundaries.

A request may travel through several services, and each service must establish whether the caller is trusted and authorized.

Example:

```text
Passenger Client
       |
       v
API Gateway
       |
       v
Booking Service
       |
       +------> Driver Service
       |
       +------> Payment Service
       |
       v
Message Broker
```

Security considerations:

### Gateway Responsibilities
- Token validation.
- Initial authentication.
- Rate limiting.
- Request size limits.
- TLS termination where appropriate.

### Service Responsibilities
- Enforce resource-level authorization.
- Validate trusted caller identity.
- Avoid trusting arbitrary forwarded headers.
- Use least-privilege service credentials.
- Validate message producers and consumers.

**Important:** API Gateway validation does not remove the need for authorization inside backend services.

### Service-to-Service Authentication

Common approaches:
- Mutual TLS (mTLS).
- Workload identity.
- Short-lived service tokens.
- Cloud-native identity systems.

With mTLS, both client and server authenticate using certificates.

---

## 16. Security in Message-Driven Systems

Message brokers also need security controls.

Protect:
- Broker connections.
- Topics and queues.
- Producer permissions.
- Consumer permissions.
- Message contents.
- Administrative interfaces.

Example:

```text
Booking Service
   |
   | Authorized producer
   v
Kafka Topic: booking-events
   |
   | Authorized consumer
   v
Notification Service
```

A notification consumer should not automatically have permission to publish to every administrative topic.

Use:
- TLS.
- Authentication.
- Topic-level authorization.
- Least-privilege credentials.
- Sensitive-data minimization.
- Schema validation.

Do not place passwords, access tokens, or unnecessary personal information inside events.

---

## 17. Threat Modeling

Threat modeling is a structured process for identifying possible security threats before they become incidents.

A useful starting point is STRIDE.

| Threat | Meaning | Example |
|---|---|---|
| Spoofing | Pretending to be someone else | Stolen passenger token |
| Tampering | Unauthorized data modification | Changing booking fare |
| Repudiation | Denying an action occurred | No audit record for driver assignment |
| Information Disclosure | Exposing confidential data | Accessing another passenger's location |
| Denial of Service | Making a system unavailable | Flooding OTP endpoint |
| Elevation of Privilege | Gaining higher permissions | Passenger becoming an admin |

### Threat Modeling Process

1. Identify valuable assets.
2. Map system components and data flows.
3. Identify trust boundaries.
4. Identify possible threats.
5. Estimate likelihood and impact.
6. Define mitigations.
7. Revisit the model as the system changes.

### Example: Ride Booking

Assets:
- Passenger identity.
- Driver location.
- Booking records.
- Payment details.

Threat:
A passenger modifies the booking ID in an API request to view another passenger's trip.

Mitigation:
- Authenticate the caller.
- Enforce resource ownership.
- Log sensitive access.
- Add automated authorization tests.

---

## 18. Secure System Design Checklist

### Identity and Authentication
- [ ] Secure password hashing.
- [ ] MFA for sensitive accounts and operations.
- [ ] Access token expiry.
- [ ] Refresh token rotation where appropriate.
- [ ] Secure session invalidation.

### Authorization
- [ ] RBAC or an appropriate policy model.
- [ ] Object-level authorization.
- [ ] Tenant isolation.
- [ ] Least privilege.
- [ ] Server-side permission enforcement.

### API Security
- [ ] Input validation.
- [ ] Parameterized queries.
- [ ] Rate limiting.
- [ ] Request size limits.
- [ ] Secure CORS configuration.
- [ ] Safe error responses.

### Infrastructure
- [ ] TLS in transit.
- [ ] Encryption at rest.
- [ ] Private database networking.
- [ ] Service-to-service authentication.
- [ ] Secure secrets management.
- [ ] Regular credential rotation.

### Monitoring and Auditing
- [ ] Authentication failure monitoring.
- [ ] Privileged action auditing.
- [ ] Suspicious access detection.
- [ ] Sensitive data redaction.
- [ ] Security incident response process.

---

## 19. Common Security Design Mistakes

1. Treating authentication as authorization.
2. Assuming JWT payloads are encrypted.
3. Trusting user IDs sent by the client.
4. Relying on frontend route guards for security.
5. Assuming CORS protects an API from unauthorized clients.
6. Storing passwords using fast general-purpose hashes.
7. Storing long-lived tokens in JavaScript-accessible storage without considering XSS.
8. Exposing internal services and databases publicly.
9. Giving every microservice access to every resource.
10. Logging credentials and sensitive personal information.
11. Using predictable IDs as the only access-control mechanism.
12. Treating an internal network as automatically trusted.

---

## 20. Revision Questions

1. Explain the CIA triad.
2. What is the difference between authentication and authorization?
3. What is the difference between session-based and token-based authentication?
4. What are the three parts of a JWT?
5. Why is a signed JWT not necessarily encrypted?
6. What is the difference between access tokens and refresh tokens?
7. What is refresh token rotation?
8. Explain RBAC, ABAC, and ReBAC.
9. What is IDOR, and how can we prevent it?
10. What is SQL injection?
11. Why is CORS not an authentication mechanism?
12. Explain fixed-window and token-bucket rate limiting.
13. What is the difference between hashing and encryption?
14. Explain encryption at rest and in transit.
15. What is Zero Trust?
16. What is mTLS?
17. Explain XSS, CSRF, SSRF, and DDoS.
18. What is STRIDE threat modeling?
19. Why should backend services enforce authorization even when an API Gateway validates tokens?
20. How would you secure communication between microservices and a message broker?

---

## 21. Practical Exercise — Secure the Ride-Booking Platform

Design a security architecture for the following system:

- Passenger Client
- Driver Client
- Admin Portal
- API Gateway
- Authentication Service
- Booking Service
- Driver Service
- PostgreSQL
- Redis
- Kafka

Your design should answer:

1. How will passengers, drivers, and admins authenticate?
2. How will you distinguish their permissions?
3. How will you prevent passengers from accessing other passengers' bookings?
4. How will you protect driver location data?
5. How will you secure service-to-service communication?
6. How will you store database credentials and JWT signing keys?
7. What rate limits would you apply to login, OTP, and booking creation?
8. How will you protect Kafka topics?
9. What security events should be audited?
10. How will you respond if a passenger's refresh token is compromised?

### Final Takeaway

A secure system does not depend on one security mechanism.

It uses multiple layers: identity verification, authorization, encryption, network isolation, rate limiting, secure data handling, and continuous monitoring.

This is called **defense in depth**: if one security control fails, other controls should still reduce the impact.

### Practice focus

Pay special attention to authentication vs authorization, JWT security, and object-level authorization. These are particularly important when designing APIs used by multiple user types.