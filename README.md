# Config Service

Centralized configuration server for the Elara microservices platform, built with **Spring Cloud Config Server**.

This service externalizes runtime configuration by service and profile, so application binaries stay environment-agnostic and operational changes can be managed from versioned configuration.

## What this service does

`config-service` is a Spring Boot application with `@EnableConfigServer`.  
At startup, it connects to the remote configuration repository and serves configuration documents to client services over HTTP.

Current upstream config repository:
- `https://github.com/elara-app/centralized-configuration.git`

Current local server settings (`src/main/resources/application.yml`):
- `server.port: 8000`
- `spring.application.name: config-service`
- `spring.cloud.config.server.git.clone-on-start: true`
- active profile: `dev`

## How it interacts with the rest of the platform

Services that consume this Config Server:
- [inventory-service](https://github.com/elara-app/inventory-service.git)
- [unit-of-measure-service](https://github.com/elara-app/unit-of-measure-service.git)
- [api-gateway](https://github.com/elara-app/api-gateway.git)

### Interaction flow

1. `config-service` starts and clones/refreshes `centralized-configuration`.
2. A client service boots and resolves its `spring.application.name` + active profile.
3. The client requests configuration from Config Server (for example `/{application}/{profile}`).
4. Config Server returns ordered `propertySources` composed from the matching files.
5. The client continues startup with resolved properties (datasource, Eureka, gateway, RabbitMQ, etc.).

## Relationship with `centralized-configuration`

This repository should contain **server behavior only**.  
Service-specific runtime properties are owned by `centralized-configuration`.

Current mapping:

| Client service | Remote config files | Profiles |
| --- | --- | --- |
| api-gateway | `api-gateway-dev.yml` | `dev` |
| inventory-service | `inventory-service-dev.yml`, `inventory-service-test.yml`, `inventory-service-prod.yml` | `dev`, `test`, `prod` |
| unit-of-measure-service | `unit-of-measure-service-dev.yml`, `unit-of-measure-service-test.yml`, `unit-of-measure-service-prod.yml` | `dev`, `test`, `prod` |

## Vault secrets (brief note)

`centralized-configuration/service-secrets/` includes:
- `inventory-service-dev-secrets.json`
- `uom-service-dev-secrets.json`

These JSON payloads are uploaded to Vault and consumed by inventory-service and unit-of-measure-service for `dev` profile secret resolution.  
Secret-management details are intentionally documented in depth in `centralized-configuration`, not here.

## Config resolution endpoints

Useful Config Server endpoints:

- `GET /{application}/{profile}`
- `GET /{application}/{profile}/{label}`

Examples:

```bash
curl http://localhost:8000/inventory-service/dev
curl http://localhost:8000/unit-of-measure-service/test
curl http://localhost:8000/api-gateway/dev
```

The response contains `propertySources` in precedence order, which is the key artifact for troubleshooting effective values.

## Build, test, and run

Requirements:
- Java 21
- Maven Wrapper (`./mvnw`)
- Network access to `centralized-configuration` repository

Commands:

```bash
./mvnw clean install
./mvnw test
./mvnw spring-boot:run
```

Single test execution:

```bash
./mvnw test -Dtest=ConfigServiceApplicationTests
./mvnw test -Dtest=ConfigServiceApplicationTests#contextLoads
```


## Operational best practices

- Keep this repository focused on Config Server runtime behavior; do not duplicate service profile configuration here.
- Enforce strict file naming in `centralized-configuration`: `<spring.application.name>-<profile>.yml`.
- Keep credentials and secret values out of plain config files; use Vault-backed secret injection.
- Preserve backward compatibility for shared keys to support rolling deployments across services.
- Validate every configuration change through Config Server endpoints before promoting changes across environments.
