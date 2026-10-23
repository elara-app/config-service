# Agent guidelines — config-service

## Read first

- [README.md](README.md) — purpose, endpoint behavior, configuration summary, limitations.
- [infrastructure/README.md](https://github.com/elara-app/infrastructure#readme) — full-stack commands, ports, source-vs-published stacks and canonical verification.
- [centralized-configuration/README.md](https://github.com/elara-app/centralized-configuration#readme) — served files, profile matrix, secrets policy.
- Source of truth: `src/main/resources/application.yml`, `pom.xml`, `Dockerfile`, `.github/workflows/`.

## Scope and boundaries

- This repo owns Config Server runtime behavior only: `ConfigServiceApplication`, `application.yml`, the Dockerfile and CI.
- Served configuration content belongs to the `centralized-configuration` repository. Do not add service YAML files, profile values, gateway routes or secrets here.
- Do not modify client repositories (`api-gateway`, `inventory-service`, `unit-of-measure-service`, `discovery-service`) from a change in this repo; coordinate contract changes with them.
- Documentation-only changes must not touch code, configuration, Compose files, scripts or workflows.

## Quick facts

| Fact | Value |
|---|---|
| Package | `com.elara.app.config_service` |
| Service ID / image | `config-service` / `ghcr.io/elara-app/config-service` |
| Port | `${SERVER_PORT:8888}`; source Compose maps `SERVER_PORT_CONFIG` (default `8888`); published Compose hardcodes `8888` |
| Runtime dependencies | None beyond Spring Boot and Config Server: no Eureka client, no Vault client, no AMQP/Bus |
| Config repository | `${CONFIG_GIT_URI:https://github.com/elara-app/centralized-configuration.git}`, default label `main` |
| Health | `GET /actuator/health`; details always shown; only `health,info` exposed |

## Commands

Run from the repository root. The Maven wrapper is mode 644, so always use `sh ./mvnw`.

```bash
sh ./mvnw -B test                                        # full test suite (one class)
sh ./mvnw -B test -Dtest=ConfigServiceApplicationTests   # focused test
sh ./mvnw -B clean verify -Dspring.profiles.active=test  # full verification
sh ./mvnw spring-boot:run                                # serve locally; first request clones the repo
```

Full-stack start/stop/verify commands live only in [infrastructure/README.md](https://github.com/elara-app/infrastructure#readme).

## Repository map

| Path | Role |
|---|---|
| `src/main/java/com/elara/app/config_service/ConfigServiceApplication.java` | `@SpringBootApplication` + `@EnableConfigServer`; the only production class |
| `src/main/resources/application.yml` | Port, Git backend settings, actuator exposure |
| `src/test/java/com/elara/app/config_service/ConfigServiceApplicationTests.java` | Single context-load test |
| `Dockerfile` | Multi-stage image; layered jar; non-root `appuser`; no `HEALTHCHECK` (Compose supplies it) |
| `.github/workflows/test.yml` | PR test workflow (org reusable workflow) |
| `.github/workflows/publish-image.yml` | GHCR image publish on `main` |
| `pom.xml` | Java 21, Spring Boot 3.5.6, Spring Cloud 2025.0.0 |

## Contracts and invariants

- Keep the endpoint pair `GET /{application}/{profile}` and `GET /{application}/{profile}/{label}`.
- Keep status semantics: unknown application/profile returns 200 with empty `propertySources`; an unknown label returns 404. Do not "fix" either one into the other behavior.
- Keep placeholders raw. The server must not interpolate or validate `${...}` values; clients resolve them against their own environment and defaults.
- Keep actuator exposure at `health,info`. `/actuator/env` is shadowed by the config route and returns an empty environment; do not document or rely on it as an actuator endpoint.
- Keep the port on `${SERVER_PORT:8888}`; never hardcode 8888.
- Keep `clone-on-start: false`, `force-pull: true`, `timeout: 10`, `refresh-rate: 60` and `default-label: main`.
- Do not set an active profile in `application.yml`; startup uses the `default` profile.
- Do not add Eureka, Vault, security or Bus/AMQP dependencies. Refresh is client-owned; this server has no push/refresh endpoint.
- `spring.application.name` must stay `config-service`, matching the service ID.

## Configuration and runtime

- `CONFIG_GIT_URI` overrides the Git URI: the source Compose stack sets it to a read-only bind mount of `../centralized-configuration` (`/config-repo`); the published stack leaves it at the default GitHub URI and sets `SPRING_CLOUD_CONFIG_SERVER_GIT_CLONE_ON_START: "true"`.
- Label contract: `default-label: main`; clients may request another label in the URL, unknown labels are rejected with 404. Compose maps `CONFIG_GIT_DEFAULT_LABEL` to `SPRING_CLOUD_CONFIG_SERVER_GIT_DEFAULT_LABEL`.
- Lazy clone: with `clone-on-start: false`, a fresh start needs no network; the first config request (or a repository health probe) triggers JGit. Expect the first request to be slow.
- Unreachable remote: the JVM still starts, config requests fail and `/actuator/health` returns 503 with `configServer` DOWN; Compose healthchecks then block dependent services.
- The Docker image packages with `-DskipTests`; CI is the automated gate.
- Git JGit settings (`force-pull`, `timeout`, `refresh-rate`) control the local clone refresh; they do not resolve client placeholders.

## Testing and quality gates

- One test class, `ConfigServiceApplicationTests`, with one `contextLoads()` method; `@SpringBootTest` only, no HTTP or config-resolution assertions.
- Commands: `sh ./mvnw -B test`; focused: `sh ./mvnw -B test -Dtest=ConfigServiceApplicationTests`.
- CI: PRs to `develop`/`main` call the org reusable test workflow (`.github/workflows/test.yml`); pushes to `main` publish `ghcr.io/elara-app/config-service:latest` (`.github/workflows/publish-image.yml`; markdown-only changes ignored).
- `pom.xml` has no JaCoCo, Checkstyle, SpotBugs, PMD or coverage threshold. Do not add a coverage gate in an unrelated change, and do not claim one exists.
- Do not claim the test verifies serving behavior; it only loads the application context.

## Definition of done

- `sh ./mvnw -B test` passes; for `application.yml` or dependency changes, also run `sh ./mvnw -B clean verify -Dspring.profiles.active=test`.
- Endpoint behavior is checked when serving config changes: known app/profile, unknown app (200 with empty `propertySources`), unknown label (404).
- No config values, profiles or secrets were added to this repo; served content changes go to `centralized-configuration`.
- README/AGENTS updated when commands, ports, endpoints or env-var contracts change.
- `git diff --name-only` shows only the intended files; no unrelated reformatting.

## Stop and ask

- Changing the Git repository, default label, clone/pull settings or the git block in `application.yml`.
- Adding authentication, TLS, rate limiting or any new exposure on port 8888.
- Changing serving semantics, endpoint mappings or health status behavior.
- Adding dependencies (Eureka, Vault, security, Bus/AMQP) or an active profile.
- Anything requiring edits outside this repo (`centralized-configuration`, `infrastructure`, client services, CI workflows).
- Changing the container image contract (entrypoint, user, base image, packaging).

## Known traps

- Lazy clone: "starts and clones at startup" is wrong for the default configuration; only the published stack pre-clones.
- Unknown application/profile returns 200 with empty `propertySources`, not 404. Unknown label returns 404, not 200.
- Placeholders are returned raw; the server never resolves `${...}`.
- The published Compose stack sets `SPRING_CLOUD_CONFIG_SERVER_GIT_CLONE_ON_START: "true"` and needs GitHub access at startup; do not describe that stack as offline or SHA-pinned.
- `application.yml` has no active profile; startup logs a fallback to the `default` profile. Do not claim `dev`.
- `/actuator/env` is not a working actuator endpoint here; the config route captures it.
- Health 503 with `configServer` DOWN does not mean the JVM is dead.
- No refresh endpoint exists: `/actuator/refresh` falls through to the config route and returns an empty environment.
- `./mvnw` fails because the wrapper is mode 644; use `sh ./mvnw` and do not instruct `chmod +x`.

## Documentation maintenance

- README owns human-facing purpose, API behavior, configuration summary and troubleshooting; AGENTS owns constraints, commands and traps. Do not duplicate sections between them.
- Verify every port, endpoint, env var and status code against `application.yml`, the Compose files and workflows before editing docs.
- Use absolute cross-repo links: `https://github.com/elara-app/infrastructure#readme`, `https://github.com/elara-app/centralized-configuration#readme`.
- If behavior changes in code, update README and AGENTS in the same change.
- Remove stale statements instead of appending corrections; no marketing prose and no "last updated" metadata.
