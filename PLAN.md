# bt-auth-server plan

A Spring Boot authorization server, built by hand as a learning project. It is the back end of the identity provider at `auth.bjorntirsen.se`.

## Architecture

- **`auth.bjorntirsen.se` is the identity provider.** It is the only public hostname for login.
- **`bt-auth`** (separate repo: React client + Express BFF) is its front. It serves the login and account pages and forwards protocol paths to this service.
- **`bt-auth-server`** (this repo) is its back end: users, sessions, tokens and keys. It has no public domain and is reached only over the internal Docker network on the VPS.
- **`games.bjorntirsen.se`** comes later as the first client application.

## Decisions

- Maven, Java 25 (Amazon Corretto), Spring Boot, YAML configuration.
- Base package `se.bjorntirsen.auth`.
- Oracle Database Free as the database, in a container (`gvenzl/oracle-free`).
- Spring Data JDBC for persistence, Flyway for migrations.
- Spring Authorization Server for the protocol endpoints; nothing in OAuth2/OIDC is hand-rolled.
- Modular monolith: top-level packages are modules, verified with Spring Modulith. Modules expose a small public API; side effects across modules go through application events.
- Records and plain Java; no Lombok.
- Versions come from the Spring Boot parent. Never pin Spring Security or Spring Framework separately.
- No personal data in tokens or logs. Users have an opaque internal ID; the email is only for login.
- Store only what is needed, hashed where possible.
- Public repo. No secrets or signing keys are committed; configuration comes from environment variables.
- Deployed with Coolify on the VPS, built from the `Dockerfile`.

## Steps

Each step adds one concept and ends with a commit and a deploy.

### 1. A service that starts, deployed

- [x] Generate the project at start.spring.io (Spring Web, Actuator)
- [x] Run locally; `/actuator/health` returns `UP`
- [x] First commit, pushed to a public GitHub repo
- [x] `Dockerfile`; build and run the image locally
- [x] Deploy in Coolify: Dockerfile build pack, port 8080, no domain, health check `/actuator/health`, memory limit 512 MB, network alias `bt-auth-server`
- [x] In `bt-auth`: add a `checks.authServer` entry to `/api/health` that fetches `http://bt-auth-server:8080/actuator/health` (URL from an environment variable), and show it in the client

Done when the "Check backend" button on auth.bjorntirsen.se reports the auth server as healthy.

### 2. Oracle and a first migration

- [ ] `compose.yaml` with Oracle Free; Spring Boot Docker Compose support starts it on `spring-boot:run`
- [ ] Oracle driver, Spring Data JDBC, Flyway
- [ ] First Flyway migration
- [ ] Database included in the health check
- [ ] Integration test against Oracle with Testcontainers
- [ ] Oracle on the VPS (see open questions)
- [ ] In `bt-auth`: remove Drizzle, `pg`, the Postgres compose setup, `DATABASE_URL` and the database check; decide what `ok` and the 503 depend on afterward

Done when the deployed auth server reports its own database as healthy.

### 3. A `users` module

- [ ] `users` table with an opaque ID (UUID), email and password hash
- [ ] Repository and a registration endpoint, using records
- [ ] Passwords hashed with Spring Security's `PasswordEncoder`
- [ ] Logs carry the user ID only, never the email or the request body

Done when a user can be registered and found again in an integration test.

### 4. Login and the session

- [ ] Spring Security form login against the `users` module
- [ ] A session-based security filter chain (not stateless), with session fixation protection
- [ ] Sessions stored in Oracle with Spring Session JDBC, so logins survive a redeploy
- [ ] Cookie attributes decided: `HttpOnly`, `Secure`, `SameSite`
- [ ] CSRF protection for a React login form
- [ ] Custom login page served by `bt-auth` (React), posting to this service
- [ ] Express in `bt-auth` forwards the login post and passes cookies through unchanged
- [ ] A protected endpoint that returns the current user's ID, shown on an account page in `bt-auth`
- [ ] Logout that ends the session

Done when you can log in on auth.bjorntirsen.se, see your user ID, redeploy, and still be logged in.

### 5. Authorization server

- [ ] Spring Authorization Server, issuer `https://auth.bjorntirsen.se`
- [ ] Trust forwarded headers from the proxy
- [ ] Express in `bt-auth` forwards `/oauth2/*` and `/.well-known/*` to this service
- [ ] A test client registered from configuration, with its secret from the environment
- [ ] Authorization code flow with PKCE, tested end to end

A generated in-memory signing key is acceptable here: no real client exists yet, so tokens invalidated by a redeploy cost nothing. Step 6 replaces it.

Done when the full authorization code flow works against the deployed server.

### 6. A `keys` module

- [ ] Signing keys in a table, one `kid` per key
- [ ] Private keys encrypted at rest with a key-encryption key from the environment
- [ ] The JWKS built from the table
- [ ] Tokens issued before a redeploy still validate after it

### 7. Tokens and claims

- [ ] An `OAuth2AuthorizationService` on Oracle that stores hashes of authorization codes and refresh tokens instead of their values, and does not store JWT access tokens
- [ ] A token customizer that sets `sub` to an HMAC of the internal user ID, keyed by a secret from the environment
- [ ] No email or other personal data in tokens

### 8. Module boundaries

- [ ] Spring Modulith verification test across `users`, `keys` and the authorization server
- [ ] Application events for side effects across modules, such as revoking a user's authorizations when the user is deleted

### 9. First client

- [ ] A `clients` module: registered clients stored in Oracle instead of configuration
- [ ] `games.bjorntirsen.se` signs users in through `auth.bjorntirsen.se`
- [ ] Pairwise `sub`: the client ID added to the HMAC input, so each client sees a different identifier for the same user
- [ ] RP-initiated logout from the client

## Later

- Key rotation, scheduled or admin-triggered. A retired key stops signing but stays in the JWKS until every token it signed has expired, then it is removed.
- Admin API for clients and scopes.
- OpenAPI contract for custom (non-OIDC) endpoints: `springdoc-openapi` here, `openapi-typescript` in `bt-auth`, checked in CI.
- Renovate, updating the Spring Boot parent rather than individual Spring libraries.
- A fully internal variant behind WireGuard or Tailscale, to mimic a staff-only identity provider.

## Open questions

- **Memory on the VPS.** It has 4 GB. Oracle Free wants about 2 GB plus a few hundred MB per JVM. Check `free -h` and `docker stats --no-stream` before step 2's deploy; the fallbacks are adding swap, a bigger plan, or running Oracle only locally and in tests.
- **Maven or Gradle.** Match what the team at work uses if it differs.