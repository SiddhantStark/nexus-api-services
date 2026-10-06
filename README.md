# Nexus Commerce backend

V1 is a Java 21 / Spring Boot modular application backed by PostgreSQL. Payments are local simulations and never charge real money. Phase 1 establishes configuration, package boundaries and startup only. Registration/JWT, commerce endpoints, Flyway migrations and database integration fixtures are later phases.

## Prerequisites

- Java 21 (select it explicitly; a different system Java may otherwise be used).
- The checked-in Maven wrapper, pinned to Maven 3.9.16; first use downloads dependencies.
- Docker with Compose for the local PostgreSQL service, or an isolated PostgreSQL 15.16 instance.
- Existing React frontend normally served at `http://127.0.0.1:8443`; backend port 8080.

The existing Spring Boot 4.1.1 pin is retained and verified through compilation. Spring AI and Bedrock are removed from V1. No AWS/AI credentials are required for startup.

## Local setup

Run from `nexus-backend`:

```sh
export JAVA_HOME=$(/usr/libexec/java_home -v 21) # macOS; select JDK 21 on other OSes
cp .env.example .env
# Edit .env and set a new local-only DB_PASSWORD before continuing.
# Only source a local file you control; .env is ignored by Git.
set -a
. ./.env
set +a
docker compose up -d --wait postgres
./mvnw spring-boot:run
```

Spring Boot does not automatically load `.env`; export it as above or provide the variables through your IDE. Compose reads it for the database service. Use a shell-safe value such as a randomly generated hex password. Do not paste production credentials into this file.

The database publishes only on `127.0.0.1:5433` to avoid an existing service on 5432. `DB_URL` uses database `nexus_local`; changing `POSTGRES_PORT` also requires updating DB_URL and TEST_DB_URL. PostgreSQL first-start credentials are stored in its volume; changing `.env` does not change the password in an existing volume.

The host runs Java and React; Compose runs PostgreSQL only. This is the Phase 1 local-development variant of the system design. A backend image and cloud deployment follow in Phase 11.

Check `http://localhost:8080/actuator/health` for `{"status":"UP"}`. Only health is exposed. All other routes remain protected by Spring Security defaults; there are no business routes or JWT implementation yet. Default generated development authentication is not a V1 login feature and must not be deployed as one.

## Profiles and schema ownership

| Profile | Database configuration | Purpose |
| --- | --- | --- |
| local | DB_URL (local default), required DB_USERNAME and DB_PASSWORD | Developer-run application |
| test | Required TEST_DB_URL, TEST_DB_USERNAME, TEST_DB_PASSWORD | Isolated test database; no local DB fallback |
| prod | Required DB_URL, DB_USERNAME, DB_PASSWORD; optional DB_POOL_SIZE | External secrets/configuration, graceful shutdown and health probes |

Activate one of these profiles explicitly using SPRING_PROFILES_ACTIVE. No credentials are embedded in source. In deployment use a PostgreSQL TLS connection string and secret injection from the platform. The test database must not contain data you need to keep.

Hibernate uses `validate` in every profile and Open Session in View is disabled. There are no entities or Flyway migrations yet, so a successful Phase 1 startup validates connectivity only and creates no domain tables. Add Flyway and its migrations in Phase 2 before introducing entities. Do not switch back to `ddl-auto=update` to bypass missing migrations.

## Tests and build

Create an isolated test database on the disposable local Compose instance:

```sh
docker compose exec postgres sh -c 'createdb -U "$POSTGRES_USER" nexus_test'
# Run once; "already exists" means the test database was previously created.
# Set TEST_DB_PASSWORD to your local DB_PASSWORD in .env, then re-export .env.
./mvnw clean verify
```

The context test selects `test` explicitly. `verify` requires all TEST_DB_* variables and an available isolated database. Phase 2 will add dedicated PostgreSQL fixtures and migration tests; this context test alone does not validate transactions.

Compile without starting the database:

```sh
./mvnw -DskipTests compile
```

Start a packaged application after a verified build:

```sh
java -jar target/nexus-0.0.1-SNAPSHOT.jar
```

Use the same environment variables as local startup. Never include secrets in command-line flags, logs, screenshots or committed configuration.

## Stop and reset the local database

`docker compose stop` keeps data. `docker compose down` removes the local containers/network but keeps the named volume.

Only when intentionally discarding this project's disposable local data:

```sh
docker compose down --volumes
docker compose up -d --wait postgres
```

This deletes both local and test databases in the Compose volume. Never point this procedure at a shared or deployed database. Recreate `nexus_test` afterward. Do not use a reset as a schema migration strategy.

## Package boundaries

`auth`, `address`, `catalog`, `inventory`, `cart`, `order`, `payment`, `refund`, `history` and `common` each contain a package description. Controllers/DTOs/services/repositories will be added only as their phase needs them. Keep common code limited to shared technical conventions; commerce operations belong to their owning features.

See [backend phases](../docs/backend-development-plan.md), [database/API contract](<../docs/NexusCommerce V1 - Database and API Design.md>) and [system design](<../docs/NexusCommerce V1 - System Design.md>).

## Repository and secret boundary

On 6 October 2026 the backend and shared docs resolve to `/Users/siddhantprasad/Desktop`, whose branch has no commits; neither directory is tracked. The frontend is a separate Git repository. No repository was initialized and nothing was staged or committed during Phase 1.

Intended ownership: backend source and backend CI belong together, and shared `NexusCommerce/docs` must be explicitly tracked by the chosen project repository. Until a project-level or backend-plus-shared-docs layout is authorized, keep these local and do not install backend CI into the frontend-only repository or stage the Desktop parent broadly. Recheck boundaries before any commit.

The original inline database password was removed. No commit of this file was found in the current parent repository. New local databases must use a new password; this work does not change an existing database account or prove a credential was never shared elsewhere.

## Phase 1 verification record

Verified on 6 October 2026 with Java 21.0.10, Maven 3.9.16, Spring Boot 4.1.1 and native PostgreSQL 15.16:

- Compilation and `./mvnw -B clean verify` passed; 1 context test, no failures/errors/skips.
- Packaged `local` and `prod` profiles returned health UP against a temporary database.
- Dependency tree has no Spring AI or Bedrock artifacts.
- Startup created no application tables; all verification services were stopped afterward.
- Compose configuration validates. Docker daemon was stopped, so container startup/image pull was not tested.

The temporary database used port 55439 and random disposable credentials, separate from the normal local setup. These checks establish a runnable skeleton, not implemented authentication, commerce APIs or migrations.
