# AGENTS.md

This file provides guidance to agents when working with code in this repository.

## Architecture Constraints (Non-Obvious)

- **Schema is immutable from code** — `ddl-auto=none` means Hibernate never touches the Oracle schema. Any new entity field, table, or index must be applied as a manual DDL script against the Oracle Autonomous DB before deploying code that references it.
- **No custom queries** — `StudentRepository` extends `JpaRepository` with zero custom methods. All data access uses Spring Data derived queries or `findAll`/`findById`/`save`/`deleteById`. Adding JPQL/native queries requires no extra infrastructure.
- **Single-microservice, no service discovery** — Despite Spring Cloud Config, there is no Eureka, Consul, or Gateway. This is a standalone microservice deployed as a single Docker container on one EC2 instance.
- **CORS is wide open** — `@CrossOrigin(origins = "*")` on the controller means any frontend origin can call the API. This is intentional for the current deployment but must be restricted if security requirements change.
- **CI produces a single long-lived image tag** — The workflow pushes only `:latest` to DockerHub; there is no versioned tagging. Rolling back requires manually pulling a previous layer or re-running an older commit's pipeline.
- **AWS credentials are session-based** — The pipeline uses `AWS_SESSION_TOKEN` (temporary STS credentials). If the pipeline fails with auth errors, the session token has likely expired and must be rotated in GitHub Secrets.
- **Wallet must travel with the artifact** — The Oracle mTLS wallet is `COPY`-ed into the Docker image at build time. The image is therefore tied to this specific Oracle DB instance. Changing the DB requires rebuilding the image with a new wallet.
