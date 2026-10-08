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
- Modular monolith: top-level packages are modules, verified with Spring Modulith once there are two.
- Public repo. No secrets or signing keys are committed; configuration comes from environment variables.
- Deployed with Coolify on the VPS, built from the `Dockerfile`.

## Steps

Each step adds one concept and ends with a commit and a deploy.

### 1. A service that starts, deployed

- [x] Generate the project at start.spring.io (Spring Web, Actuator)
- [x] Run locally; `/actuator/health` returns `UP`
- [ ] First commit, pushed to a public GitHub repo
- [ ] `Dockerfile`; build and run the image locally
- [ ] Deploy in Coolify: Dockerfile build pack, port 8080, no domain, health check `/actuator/health`, memory limit 512 MB, network alias `bt-auth-server`
- [ ] In `bt-auth`: add a `checks.authServer` entry to `/api/health` that fetches `http://bt-auth-server:8080/actuator/health` (URL from an environment variable), and show it in the client

Done when the "Check backend" button on auth.bjorntirsen.se reports the auth server as healthy.

### 2. Oracle and a first migration

- [ ] `compose.yaml` with Oracle Free; Spring Boot Docker Compose support starts it on `spring-boot:run`
- [ ] Oracle driver, Spring Data JDBC, Flyway
- [ ] First Flyway migration
- [ ] Database included in the health check
- [ ] Integration test against Oracle with Testcontainers
- [ ] Oracle on the VPS (see open questions)

### 3. A `users` module

- [ ] `users` table, repository, and a registration endpoint
- [ ] Passwords hashed with Spring Security's `PasswordEncoder`

### 4. Login

- [ ] Spring Security form login against the `users` module
- [ ] Custom login page served by `bt-auth` (React), posting to this service

### 5. Authorization server

- [ ] Spring Authorization Server, issuer `https://auth.bjorntirsen.se`
- [ ] Trust forwarded headers from the proxy
- [ ] Express in `bt-auth` forwards `/oauth2/*`, `/.well-known/*` and the login post to this service
- [ ] Registered clients and signing keys stored outside the repo
- [ ] Authorization code flow with PKCE, tested end to end

### 6. Module boundaries

- [ ] Spring Modulith verification test, once a second module exists

### 7. First client

- [ ] `games.bjorntirsen.se` signs users in through `auth.bjorntirsen.se`

## Later

- OpenAPI contract for custom (non-OIDC) endpoints: `springdoc-openapi` here, `openapi-typescript` in `bt-auth`, checked in CI.
- A fully internal variant behind WireGuard or Tailscale, to mimic a staff-only identity provider.

## Open questions

- **Memory on the VPS.** It has 4 GB. Oracle Free wants about 2 GB plus a few hundred MB per JVM. Check `free -h` and `docker stats --no-stream` before step 2's deploy; the fallbacks are adding swap, a bigger plan, or running Oracle only locally and in tests.
- **Maven or Gradle.** Match what the team at work uses if it differs.