# AGENTS.md

This file provides guidance to agents when working with code in this repository.

## Coding Rules (Non-Obvious)

- **No Lombok on model** — `Student` uses manual getters/setters even though Lombok is a dependency. If you add Lombok `@Data`/`@Getter`/`@Setter`, it will be excluded from the JAR (see `pom.xml` plugin exclusion) — Lombok is compile-only and works fine, but the exclusion is already configured.
- **Null return from updateStudent** — `StudentServiceImpl.updateStudent()` returns `null` when the ID is not found instead of throwing or returning `Optional`. Do not "fix" this to an exception without updating the controller and tests.
- **No `@Valid` on controller** — `@RequestBody` parameters in `StudentController` are **not** annotated with `@Valid`. Adding `@Valid` is required for Bean Validation to trigger on POST/PUT; without it, `@NotBlank`/`@Size`/`@Pattern` on the entity are silently ignored.
- **Entity as DTO** — `Student` is used directly as request/response body. There are no DTO classes. Don't introduce a DTO layer without updating all layers.
- **application.properties has placeholder literals** — `<db_url>`, `<db_username>`, `<db_password>` are literal strings that must be replaced by the CI pipeline (via `sed`). Running locally requires manually replacing them before `./mvnw spring-boot:run`.
- **Config Server on startup** — The app connects to a Spring Cloud Config Server at startup. In unit/slice tests (`@WebMvcTest`, `@ExtendWith(MockitoExtension.class)`) this is fine (context not loaded). `BdgetApplicationTests` (`@SpringBootTest`) will attempt this connection — it won't fail due to `fail-fast=false` but will log retries.
- **TNS_ADMIN env var required** — Oracle wallet path must be set as `TNS_ADMIN` environment variable before JVM starts. In Docker it's `ENV TNS_ADMIN=/app/wallet`. Locally, export `TNS_ADMIN` to point to `Wallet_N72BZHZWYZGTE7OH/`.
- **Test class naming** — Test classes use no public modifier (`class StudentControllerTest` not `public class`). Follow the same pattern.
- **Validation messages are in Spanish** — Keep validation messages in Spanish to match existing conventions (e.g., `"El nombre solo puede contener letras"`).
