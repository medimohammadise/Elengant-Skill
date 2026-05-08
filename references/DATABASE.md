# Database Best Practices (PostgreSQL Runtime)

## Contents
- [When to use this guide](#when-to-use-this-guide)
- [Defaults](#defaults)
- [Spring Boot Configuration](#spring-boot-configuration)
- [Testcontainers Integration](#testcontainers-integration)
- [Docker Compose (Dev)](#docker-compose-dev)
- [Production Tips](#production-tips)
- [Local Developer Experience](#local-developer-experience)
- [Observability](#observability)
- [Validation / Checks](#validation--checks)
- [Troubleshooting](#troubleshooting)
- [References](#references)

## When to use this guide
- PostgreSQL configuration and connection properties
- `compose.yaml` and Docker Compose integration
- Testcontainers setup and image selection
- Production database operations and runtime tuning

For `@Entity` mapping, Spring Data JPA repositories, Hibernate `ddl-auto`, and ORM performance patterns, see [SPRING-DATA-JPA-HIBERNATE.md](SPRING-DATA-JPA-HIBERNATE.md).

## Defaults
- **Engine:** PostgreSQL (preferred version: **18**; configure in `versions.json`).
- **Schema management:** Hibernate `ddl-auto`, documented in [SPRING-DATA-JPA-HIBERNATE.md](SPRING-DATA-JPA-HIBERNATE.md).
- **Driver:** `org.postgresql:postgresql` (bundled via start.spring.io dependency).
- **Testcontainers:** Use `postgres:18-alpine` images.

## Spring Boot Configuration

`src/main/resources/application.properties`:
```properties
# Datasource
spring.datasource.url=${SPRING_DATASOURCE_URL:jdbc:postgresql://localhost:5432/mydb}
spring.datasource.username=${SPRING_DATASOURCE_USERNAME:user}
spring.datasource.password=${SPRING_DATASOURCE_PASSWORD:password}
spring.datasource.driver-class-name=org.postgresql.Driver

# JPA / Hibernate
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=false
spring.jpa.properties.hibernate.format_sql=false
spring.jpa.open-in-view=false
```

## Testcontainers Integration
```java
@TestConfiguration(proxyBeanMethods = false)
class TestcontainersConfiguration {
  @Bean
  @ServiceConnection
  PostgreSQLContainer postgresContainer() {
    return new PostgreSQLContainer("postgres:18-alpine")
      .withReuse(true);
  }
}
```
Use `@Import(TestcontainersConfiguration.class)` in integration tests. Keep class **package-private** (Boot 4 requirement).

## Docker Compose (Dev)
`compose.yaml` (used by `spring-boot-docker-compose`):
```yaml
services:
  postgres:
    image: postgres:18-alpine
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "user"]
      interval: 10s
      timeout: 5s
      retries: 5
volumes:
  postgres_data:
```

## Production Tips
- **Pooling:** Use HikariCP defaults; tune `maximum-pool-size` and `connection-timeout`.
- **Indexes:** Verify query plans with `EXPLAIN ANALYZE` under production-like data volumes. Entity-level index declarations belong in [SPRING-DATA-JPA-HIBERNATE.md](SPRING-DATA-JPA-HIBERNATE.md).
- **Secrets:** Inject via environment variables or Vault/Key Vault; never commit plaintext.
- **Schema Validation:** Keep `spring.jpa.hibernate.ddl-auto=validate` in prod. See [SPRING-DATA-JPA-HIBERNATE.md](SPRING-DATA-JPA-HIBERNATE.md).
- **UTF-8:** Ensure DB encoding is UTF8 (default for official Postgres images).

### Connection Pool Sizing
HikariCP pool size should match your workload. A good starting formula:

```properties
# connections = (2 × CPU cores) + effective_spindle_count
# For a typical 4-core server with SSD:
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.idle-timeout=300000
spring.datasource.hikari.max-lifetime=1800000
spring.datasource.hikari.connection-timeout=30000
```

## Local Developer Experience
- Enable `spring-boot-docker-compose` (Boot 3.1+) to auto-start `compose.yaml` on `./mvnw spring-boot:run`.
- Provide `.env.sample` with placeholders: `SPRING_DATASOURCE_PASSWORD`, etc. (see [Project Setup](PROJECT-SETUP.md)).

## Observability
- Expose Postgres metrics via `pg_stat_statements`; integrate with Micrometer if needed.
- Consider **pgBouncer** for high-connection scenarios; document in ops runbook.

## Validation / Checks
- Verify connectivity to PostgreSQL from the application.
- Verify that `compose.yaml` credentials match `spring.datasource.*` properties.
- For entity persistence checks, also read [SPRING-DATA-JPA-HIBERNATE.md](SPRING-DATA-JPA-HIBERNATE.md).

## Troubleshooting
- Common error: `FATAL: password authentication failed` — verify `spring.datasource.*` and `compose.yaml` env vars match.
- Timeouts in CI: increase Testcontainers startup timeout or use `withReuse(true)` + `~/.testcontainers.properties`.

## References

- [Spring Boot Data Access](https://docs.spring.io/spring-boot/reference/data/sql.html)
- [Spring Data JPA & Hibernate Guide](SPRING-DATA-JPA-HIBERNATE.md)
- [Testcontainers PostgreSQL Module](https://java.testcontainers.org/modules/databases/postgres/)
- [Docker Deployment Guide](DOCKER.md) — `compose.yaml` setup
- [Configuration Best Practices](CONFIGURATION.md) — externalized config & secrets
- [Project Setup](PROJECT-SETUP.md) — `.env.sample` for database credentials
