# bt-auth-server

The back end of the identity provider at [auth.bjorntirsen.se](https://auth.bjorntirsen.se): users, sessions, tokens and signing keys, built on Spring Authorization Server.

This is a learning project, written by hand to get properly familiar with Java and Spring. AI tools are used as advisors only, never to write the code. The step-by-step roadmap and the decisions behind it are in [PLAN.md](PLAN.md).

## Status

Work in progress. The service builds, deploys and reports its health; the identity provider features are being added one step at a time, following the plan.

## Architecture

- **auth.bjorntirsen.se** is the identity provider and the only public hostname for login.
- **[bt-auth](https://github.com/bjorntirsen/bt-auth)** is its front: it serves the login and account pages and forwards protocol paths to this service.
- **bt-auth-server** (this repo) is its back end. It has no public domain and is reached only over the internal Docker network on the server.
- **games.bjorntirsen.se** will be the first client application.

## Stack

Java 25 (Amazon Corretto), Spring Boot, Spring Authorization Server, Spring Modulith, Maven, PostgreSQL with Spring Data JDBC and Flyway.

## Running locally

Requirements: Java 25 and Docker.

    ./mvnw spring-boot:run

The health check is at http://localhost:8080/actuator/health.

Run the tests with:

    ./mvnw test

Build and run the container image:

    docker build -t bt-auth-server .
    docker run -p 8080:8080 bt-auth-server

## Configuration

All configuration comes from environment variables; no secrets or keys are committed. Variables are listed here as they are introduced.

## Deployment

Deployed with Coolify on a self-hosted server, built from the `Dockerfile`.

## License

[MIT](LICENSE)
