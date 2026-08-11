# AGENTS.md

This file provides guidance to agents when working with code in this repository.

## Stack

- Java 17 (source), Eclipse Temurin 22 JDK (Docker runtime)
- Spring Boot 3.3.7 + Spring Cloud Config + Spring Data JPA
- Oracle Autonomous Database (ojdbc11 + mTLS wallet)
- Maven wrapper (`./mvnw`) — do not install Maven separately

## Commands

```bash
# Build & test
./mvnw clean package            # build JAR + run all tests + generate JaCoCo report
./mvnw test                     # run all tests only
./mvnw test -Dtest=StudentServiceImplTest          # run a single test class
./mvnw test -Dtest=StudentControllerTest#testCreateStudent  # run a single test method

# Run locally (requires Oracle wallet + DB credentials in application.properties)
./mvnw spring-boot:run

# Docker (handles wallet automatically)
docker-compose up --build
```

## Critical: Database & Oracle Wallet

- The `Wallet_N72BZHZWYZGTE7OH/` directory contains Oracle mTLS credentials (`.sso`, `.p12`, `.pem`, `.ora`, `.properties`). **Do not delete or move it.**
- The wallet must be mounted at the path referenced by `TNS_ADMIN`. In Docker, `TNS_ADMIN=/app/wallet`.
- `application.properties` has placeholder values `<db_url>`, `<db_username>`, `<db_password>`. The CI/CD pipeline (`.github/workflows/main.yml`) injects real values via GitHub Secrets using `sed` before building. **Never commit real credentials.**
- `spring.jpa.hibernate.ddl-auto=none` — schema is never auto-generated or updated; all DDL must be applied manually to the Oracle DB.

## Critical: Spring Cloud Config

- The app tries to fetch configuration from a Config Server at `CONFIG_SERVER_URL` (default: `http://localhost:8888`) on startup.
- `spring.cloud.config.fail-fast=false` makes startup succeed even if the Config Server is unreachable; retry is configured with backoff.
- `BdgetApplicationTests` calls `BdgetApplication.main()` directly, which means the full Spring context (including config server connection attempt) starts during the test run.

## Architecture & Patterns

- **Layer structure:** `controller` → `service` (interface + `ServiceImpl`) → `repository` (JpaRepository) → Oracle DB.
- `updateStudent` returns `null` (not an exception/Optional) when the ID does not exist — callers must check for null.
- `StudentController` has `@CrossOrigin(origins = "*")` — all origins are allowed globally on that controller.
- Validation annotations (`@NotBlank`, `@Size`, `@Pattern`) are on the entity (`Student`), not on DTOs. There are no DTOs; the entity is passed directly to/from the API.
- Validation error messages are in Spanish (e.g., `"El nombre solo puede contener letras"`).
- `GlobalExceptionHandler` handles `RuntimeException` (→ 400), `MethodArgumentNotValidException` (→ 400 with field-level errors map), and `Exception` (→ 500). Error responses always include `timestamp`, `status`, and `message`/`errors`/`details`.

## Testing

- Controller tests use `@WebMvcTest` + `@MockBean` (slice test — no full context).
- Service tests use `@ExtendWith(MockitoExtension.class)` + `@InjectMocks` / `@Mock` (pure unit, no Spring context).
- Model tests are plain JUnit 5 (no Spring annotations).
- `ObjectMapper` is injected via `@Autowired` in controller tests (provided by `@WebMvcTest`).
- JaCoCo coverage report is generated automatically during `mvn test` (bound to the `test` phase); report lands in `target/site/jacoco/`.

## CI/CD

- GitHub Actions (`.github/workflows/main.yml`): build → push Docker image to DockerHub → SSH deploy to AWS EC2.
- Required GitHub Secrets: `DB_URL`, `DB_USERNAME`, `DB_PASSWORD`, `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`, `EC2_SSH_KEY`, `EC2_HOST`, `USER_SERVER`.
- The pipeline uses `aws-session-token` (temporary credentials), meaning secrets rotate and may need refreshing.
