[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/elara-app/config-service)

# config-service

`config-service` is the Spring Cloud Config Server for the Elara platform. It serves externalized configuration over HTTP to `api-gateway`, `inventory-service` and `unit-of-measure-service` from the versioned [centralized-configuration](https://github.com/elara-app/centralized-configuration) Git repository.

This repository owns the Config Server runtime only: a single `@SpringBootApplication` with `@EnableConfigServer` and the Git backend settings in `src/main/resources/application.yml`. It does not own the property values it serves, Vault secrets, gateway routes or business data; those live in `centralized-configuration` and the client repositories.

## At a glance

| Item | Value |
|---|---|
| Service ID | `config-service` |
| Runtime port | `${SERVER_PORT:8888}` (Compose default `8888`) |
| Stack | Java 21, Spring Boot 3.5.6, Spring Cloud 2025.0.0, Config Server 4.3.0 |
| Datastore | None |
| Runtime dependencies | None: no Eureka, Vault or messaging client |
| Config source | `${CONFIG_GIT_URI:https://github.com/elara-app/centralized-configuration.git}`, default label `main` |
| Health | `GET /actuator/health` (details always shown) |
| Image | `ghcr.io/elara-app/config-service` |

## Responsibilities and boundaries

- Serves the `Environment` JSON for a requested `{application}` and `{profile}` from the configured Git repository.
- Runs independently: it does not register with Eureka, read Vault or consume RabbitMQ/Spring Cloud Bus.
- Owns server runtime behavior and Git backend settings. Client property values are owned by `centralized-configuration`; secrets are owned by Vault.
- Has no authentication and no TLS: anything that can reach port 8888 can read every served property, including the dev default credentials in the dev YAMLs.

## Runtime architecture

`ConfigServiceApplication` is the only production class. Per request, Config Server resolves the Git repository (JGit clone or local clone refresh) and composes `propertySources`, in precedence order, from `<spring.application.name>-<profile>.yml` files.

- `clone-on-start: false`: startup does not clone the repository. The first config request triggers the clone, so it can be noticeably slower than later requests. A repository health probe can also trigger the first attempt.
- `force-pull: true`, `timeout: 10`, `refresh-rate: 60`: the local clone is refreshed from the remote at most every 60 seconds.
- Consumers: `api-gateway`, `inventory-service` and `unit-of-measure-service` fetch their configuration over HTTP. `discovery-service` is not a consumer, and config-service does not appear in the Eureka registry.
- No push channel: the server has no Spring Cloud Bus/AMQP dependency and exposes no refresh endpoint. Clients decide when to reload configuration.
- In the source Compose stack, `CONFIG_GIT_URI` defaults to a read-only bind mount of `../centralized-configuration`, so cloning works offline. The [published demo stack](../infra/compose.published.yml) sets `SPRING_CLOUD_CONFIG_SERVER_GIT_CLONE_ON_START: "true"` and clones the default GitHub URI at startup, so it needs GitHub access.

See [infra/README.md](../infra/README.md) for the stack layout and [centralized-configuration](../centralized-configuration/README.md) for the served files.

## API

| Endpoint | Behavior |
|---|---|
| `GET /{application}/{profile}` | Returns the `Environment` JSON for the application/profile. |
| `GET /{application}/{profile}/{label}` | Same shape; echoes the requested label. |
| `GET /actuator/health` | Health check; `configServer` status reflects repository reachability. |
| `GET /actuator/info` | Exposed actuator endpoint; empty by default. |

Response shape:

```json
{"name":"inventory-service","profiles":["dev"],"label":null,"version":"<repo-sha>","propertySources":[{"name":"...","source":{...}}]}
```

Behavior rules:

- Unknown application or profile is not an error: HTTP 200 with `"propertySources": []`. There is no global `application.yml` in the repository, so `/application/{profile}` is also empty.
- An unknown label is HTTP 404 (default Spring error JSON).
- Placeholders such as `${SERVER_PORT:8081}` or `${spring.application.name}` are returned exactly as written. The server does not resolve or validate them; clients resolve them against their own environment and defaults.
- `health` and `info` are the only exposed actuator endpoints. `/actuator/env` is captured by the config route and returns an empty environment (`application=actuator`, `profile=env`).
- Comma-separated profiles are accepted; in practice the last listed profile wins for same-named files.

Examples:

```bash
curl -s http://127.0.0.1:8888/inventory-service/dev
curl -s http://127.0.0.1:8888/api-gateway/dev/main
curl -s http://127.0.0.1:8888/actuator/health
```

## Run and verify

Prerequisites: Java 21. Docker is only needed for the Compose stack; network access to GitHub is only needed when serving from the remote URI.

```bash
sh ./mvnw -B test                                          # one context-load test
sh ./mvnw -B clean verify -Dspring.profiles.active=test    # full verification
sh ./mvnw spring-boot:run                                  # serve locally on port 8888
```

With the default remote URI, the first `spring-boot:run` request clones GitHub online.

To verify the service in the full platform, use the canonical commands in [infra/README.md#verify-the-stack](../infra/README.md#verify-the-stack). Once the stack is up:

```bash
curl -s http://127.0.0.1:8888/inventory-service/dev | jq '.propertySources[].name'
curl -s http://127.0.0.1:8888/actuator/health
```

In the published stack, startup can take a while because the container clones GitHub before the healthcheck passes.

## Configuration

All keys live in `src/main/resources/application.yml`:

| Setting | Source | Value |
|---|---|---|
| `server.port` | `SERVER_PORT` env var | `${SERVER_PORT:8888}` |
| `spring.cloud.config.server.git.uri` | `CONFIG_GIT_URI` env var | `https://github.com/elara-app/centralized-configuration.git` |
| `spring.cloud.config.server.git.default-label` | config; Compose maps `CONFIG_GIT_DEFAULT_LABEL` | `main` |
| `clone-on-start` | config; published Compose overrides via env var | `false` |
| `force-pull` | config | `true` |
| `timeout` / `refresh-rate` | config | `10` s / `60` s |
| Actuator exposure | config | `health,info` |

`application.yml` sets no active profile; startup falls back to the `default` profile. The `-Dspring.profiles.active=test` in the verify command is a command-line override for the build only.

The client profile matrix (api-gateway has `dev` only; inventory-service and unit-of-measure-service have `dev`, `test`, `prod` with sparse prod placeholders) and the file naming contract are owned by [centralized-configuration](../centralized-configuration/README.md).

## Testing and quality gates

- One test class: `ConfigServiceApplicationTests` with a single `contextLoads()` method (`@SpringBootTest`, no assertions and no HTTP call). It verifies context startup only.
- Run it with `sh ./mvnw -B test`, or focus it with `sh ./mvnw -B test -Dtest=ConfigServiceApplicationTests`.
- CI: pull requests to `develop` and `main` call the org reusable test workflow (`.github/workflows/test.yml`). Pushes to `main` publish `ghcr.io/elara-app/config-service:latest` (`.github/workflows/publish-image.yml`; markdown-only changes are ignored).
- The Docker image packages with `-DskipTests`, so CI tests are the only automated gate.
- There is no JaCoCo, Checkstyle, SpotBugs or coverage threshold in `pom.xml`.

## Current limitations

- Placeholders are returned raw; the server performs no interpolation or schema/key validation of served values.
- No authentication or TLS on port 8888. Every served property, including dev default credentials, is readable by anyone who can reach the port.
- Serving is cached: responses come from the server's local clone of the repository, refreshed at most every 60 seconds, so recent commits to the config repo can take up to a minute to appear and there is no server-sent refresh.
- Prod files in `centralized-configuration` are sparse placeholders, not complete production configuration. The server serves what exists and does not synthesize defaults.
- With an unreachable Git remote the JVM still starts, but config requests fail and `/actuator/health` returns 503 with `configServer` DOWN, which blocks Compose services that depend on the healthcheck.
- The context-load test does not exercise serving behavior, label handling or health.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `propertySources: []` with HTTP 200 | Unknown application/profile or missing file | Match `<spring.application.name>-<profile>.yml` in `centralized-configuration`; gateway only has `dev` |
| HTTP 404 on a config request | Unknown label | Use the default `main` or an existing branch/tag |
| First request after start is slow | `clone-on-start: false`; the first request clones | Wait for the clone; check the URI and network when serving from the remote |
| `/actuator/health` 503 with `configServer` DOWN | Git remote unreachable or clone failed | Fix `CONFIG_GIT_URI`/network; the published stack needs GitHub access |
| Values still contain `${...}` | By design: placeholders are raw | Resolve them in the client via env vars/defaults |
| `/actuator/env` returns an empty environment | `env` is not exposed; the config route captures it | Use `/actuator/health` and `/actuator/info` |
| Config changes do not appear | Served from the local clone | Wait for the refresh rate (60 s) or restart the container |
| Container unhealthy although startup logs look fine | Healthcheck follows `configServer` repository status | Inspect the health details; restore repository access |

## Documentation

- [AGENTS.md](AGENTS.md) — commands, invariants and traps for coding agents.
- [infra/README.md](../infra/README.md) — full-stack quickstart, ports and source-vs-published behavior.
- [centralized-configuration/README.md](../centralized-configuration/README.md) — served files, profile matrix and secrets policy.
- Source of truth: `src/main/resources/application.yml`, `Dockerfile`, `.github/workflows/`.
