# Lesson 12 - Authentication and API Security

> Reading time: about 35 minutes  
> Practice time: about 90 minutes

## Outcome

Distinguish authentication from authorization, store passwords safely, issue revocable opaque tokens, enforce ownership in database queries, and review an API as an attacker would.

## Security mindset

Security is not one middleware added at the end. It is a set of constraints on every boundary:

- What input is accepted?
- Who is making the request?
- What are they allowed to access?
- What information leaves the system?
- What resources can they exhaust?

The examples teach mechanisms, not a promise that a home-grown identity system is automatically production-ready. For high-risk applications, use a mature identity provider or thoroughly reviewed libraries.

## 1. Authentication versus authorization

Authentication answers:

> Who are you?

Authorization answers:

> Are you allowed to perform this action on this resource?

A valid login does not authorize a user to read another user's courses.

## 2. Passwords are hashed, not encrypted

Encryption is reversible with a key. Password verification should use a slow, salted password-derivation function.

Node.js provides scrypt:

~~~javascript
import { promisify } from "node:util";
import { randomBytes, scrypt as scryptCallback } from "node:crypto";

const scrypt = promisify(scryptCallback);
const keyLength = 64;

export async function hashPassword(password) {
  const salt = randomBytes(16);
  const derivedKey = await scrypt(password, salt, keyLength);

  return {
    salt: salt.toString("hex"),
    hash: derivedKey.toString("hex"),
  };
}
~~~

The random salt ensures identical passwords do not produce identical stored values.

## 3. Verify with a timing-safe comparison

~~~javascript
import { promisify } from "node:util";
import {
  scrypt as scryptCallback,
  timingSafeEqual,
} from "node:crypto";

const scrypt = promisify(scryptCallback);

export async function verifyPassword(password, stored) {
  const salt = Buffer.from(stored.salt, "hex");
  const expected = Buffer.from(stored.hash, "hex");
  const actual = await scrypt(password, salt, expected.length);

  return timingSafeEqual(expected, actual);
}
~~~

Validate password length before hashing so attackers cannot submit unbounded input. Use one generic login failure message so the response does not reveal whether an account exists.

## 4. Issue an opaque session token

An opaque token is random data with no client-readable claims:

~~~javascript
import { createHash, randomBytes } from "node:crypto";

export function createSessionToken() {
  const token = randomBytes(32).toString("base64url");
  const tokenHash = createHash("sha256")
    .update(token)
    .digest("hex");

  return {
    token,
    tokenHash,
  };
}
~~~

Return the raw token once. Store only its hash with user ID and expiration. If the database leaks, stored hashes do not immediately become usable bearer tokens.

## 5. Authenticate bearer tokens

~~~javascript
export function createAuthenticate(sessionRepository) {
  return async function authenticate(request, response, next) {
    try {
      const authorization = request.get("authorization");

      if (!authorization?.startsWith("Bearer ")) {
        return response.status(401).json({
          error: {
            code: "AUTHENTICATION_REQUIRED",
            message: "Authentication is required",
          },
        });
      }

      const token = authorization.slice("Bearer ".length);
      const session = await sessionRepository.findValidByToken(token);

      if (!session) {
        return response.status(401).json({
          error: {
            code: "INVALID_TOKEN",
            message: "Token is invalid or expired",
          },
        });
      }

      request.user = { id: session.userId };
      next();
    } catch (error) {
      next(error);
    }
  };
}
~~~

The repository hashes the received token before querying. It also checks expiration and revocation.

## 6. Authorization belongs in the data access path

Unsafe:

~~~javascript
repository.findOneBy({ id: courseId });
~~~

Safer ownership query:

~~~javascript
repository.findOneBy({
  id: courseId,
  userId: authenticatedUserId,
});
~~~

Filtering by both resource ID and owner prevents accidentally loading another user's row and forgetting a later comparison.

For many APIs, returning 404 for an inaccessible resource avoids confirming that another user's record exists.

## 7. Registration and login flows

Registration:

1. Validate and normalize email.
2. Validate password policy and input size.
3. Check a database uniqueness constraint.
4. Hash the password.
5. Store only the password hash and salt.
6. Return a safe user representation.

Login:

1. Rate-limit attempts.
2. Look up the normalized identity.
3. Run password verification.
4. Return the same failure message for invalid identity or password.
5. Create a short-lived server-side session.
6. Return the raw token once.

## 8. Defensive HTTP configuration

Apply:

- Small JSON body limits
- Strict input validation
- Secure response headers
- Narrow CORS policy when browser clients require it
- TLS at the public boundary
- Rate limits on expensive or sensitive routes
- Timeouts
- Dependency updates
- Secret redaction

CORS is a browser policy. It is not authentication and does not stop nonbrowser clients.

Lesson 13 turns the rate-limit requirement into separate login, general API, and expensive-operation policies. Keeping it in its own lesson prevents security from collapsing into one enormous checklist.

## 9. Token choice

Opaque tokens:

- Easy to revoke centrally
- Require a session lookup
- Keep claims off the client

Signed JWT access tokens:

- Can be verified without a session lookup
- Require careful issuer, audience, algorithm, key, and expiration validation
- Are harder to revoke immediately

Choose from system requirements, not fashion. This course begins with opaque tokens because their server-side lifecycle is easier to observe.

## Real-world applications and edge cases

### Where this appears

- Users stay logged in across several devices.
- Administrators revoke sessions after suspected compromise.
- Password-reset and email-verification links grant temporary authority.
- Support staff have elevated actions that ordinary users must never receive.

### Edge cases to investigate

- Login timing or error messages reveal whether an email address exists.
- Credential stuffing uses valid passwords leaked from another service.
- A password-reset token is reusable, long-lived, stored raw, or not invalidated after use.
- A stolen bearer token remains usable after the user changes their password.
- Logout revokes one device when the user expected every session to end.
- A user's role changes, but cached authorization continues granting old permissions.
- Cookie authentication introduces cross-site request-forgery concerns.
- A newly issued session is fixed to a token chosen by an attacker.
- Clock differences affect token or session expiration.
- A privileged operation checks role but not ownership or resource state.
- Sensitive fields leak through broad object serialization.

Design complete credential lifecycles: issue, store, use, expire, rotate, revoke, audit, and recover.

## Pause and predict

- Is a logged-in user automatically authorized for every course?
- Why store a token hash instead of the raw token?
- Does CORS protect an API from scripts outside browsers?
- Why should login use one generic failure message?

## Guided practice

Add:

- POST /auth/register
- POST /auth/login
- POST /auth/logout
- Authentication middleware
- Expiring server-side sessions
- Ownership on every course and study-session query

Write a table for every endpoint containing:

| Endpoint | Authentication | Authorization | Sensitive output |
|---|---|---|---|
| POST /auth/login | No | Public | Raw token once |
| GET /courses | Yes | Own rows only | No password data |

Complete the table before implementation.

## Common mistakes

- Encrypting or fast-hashing passwords
- Storing raw session tokens
- Using authentication without resource authorization
- Loading by ID before checking owner
- Logging passwords, authorization headers, or tokens
- Treating CORS as access control
- Returning different login messages for unknown user and wrong password
- Creating JWTs without validating all relevant claims
- Implementing custom cryptographic algorithms

## Independent challenge

Prove isolation with tests:

1. Register User A and User B.
2. Create a course as User A.
3. Attempt read, update, and delete as User B.
4. Attempt with missing, random, expired, and revoked tokens.
5. Verify logs and responses never contain password hashes or raw tokens.

Then write a short threat model naming assets, likely attackers, entry points, and the three most important controls.

## Knowledge check

- How do authentication and authorization differ?
- Why are passwords salted?
- What does timingSafeEqual help avoid?
- Why query by both resource ID and owner?
- What are the tradeoffs between opaque tokens and JWTs?

## Explore with an AI agent

- Threat-model my login and ownership design without proposing code first.
- Review every database query for broken object-level authorization.
- Explain scrypt salt and work factor using the exact password flow in this lesson.
- Challenge my token design with expiration, revocation, replay, and database-leak scenarios.

## Official reading

- [Node.js crypto API](https://nodejs.org/api/crypto.html)
- [Node.js security best practices](https://nodejs.org/en/learn/getting-started/security-best-practices)
- [Express production security practices](https://expressjs.com/en/advanced/best-practice-security.html)
- [PostgreSQL constraints](https://www.postgresql.org/docs/current/ddl-constraints.html)

## Definition of done

- Passwords use a salted, purpose-built password function.
- Raw bearer tokens are not stored.
- Every protected query enforces ownership.
- Invalid and expired credentials produce consistent responses.
- Security-sensitive values never appear in logs or API output.
