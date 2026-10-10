# Nexus Commerce backend

V1 is a Java 21 / Spring Boot application backed by PostgreSQL. Payments are local simulations and never charge real money. Phases 1–2 provide configuration, 14 mapped tables, Flyway migrations, restricted database roles and automated persistence tests. Authentication and business APIs begin in Phase 3.

## Prerequisites

- Java 21, selected explicitly; Maven wrapper 3.9.16 is included.
- Docker Desktop running, with Compose. PostgreSQL 15.16 is used locally and in tests.
- First build requires access to Maven Central and Docker Hub.
- React normally uses `http://127.0.0.1:8443`; backend uses port 8080.

Spring Boot 4.1.1 manages Flyway 12.4.0 and Testcontainers 2.0.5. No AI or AWS credentials are required.

## Local setup: new database

Run from `nexus-backend`:

```sh
export JAVA_HOME=$(/usr/libexec/java_home -v 21) # macOS; select JDK 21 on other OSes
cp .env.example .env
# Edit .env: choose three DIFFERENT random local-only passwords for
# DB_PASSWORD, MIGRATION_DB_PASSWORD and POSTGRES_ADMIN_PASSWORD.
set -a
. ./.env
set +a
docker compose up -d --wait postgres
./mvnw spring-boot:run
```

Only source a local file you control. Use shell-safe passwords (for example random hex), and keep `.env` untracked. Spring Boot does not automatically read `.env`; Compose does. Changing `POSTGRES_PORT` requires changing `DB_URL` too.

The bootstrap SQL in `ops/postgres/01-roles.sql` runs once when PostgreSQL initializes an empty volume. It creates schema `nexus` and two non-superuser accounts:

| Account | Environment | Purpose |
| --- | --- | --- |
| `nexus_bootstrap` | `POSTGRES_ADMIN_PASSWORD` | Local PostgreSQL administrator; never use for application connections |
| `nexus_migrator` | `MIGRATION_DB_USERNAME`, `MIGRATION_DB_PASSWORD` | Owns `nexus`; Flyway DDL and explicit data migrations |
| `nexus_runtime` | `DB_USERNAME`, `DB_PASSWORD` | Application reads/writes with table/column grants |

The local bootstrap uses these fixed names; set the example variables accordingly. In a deployed database, an operator must provision equivalent separate roles and a schema owned by the migrator before startup. Runtime must neither own tables nor inherit a privileged role. Role names are trusted deployment configuration, not request input; use simple PostgreSQL identifiers. Renaming the configured runtime role requires a planned privilege migration.

Flyway runs before Hibernate validates all 14 entity mappings. Never bypass migration failures with `ddl-auto=update`, automatic baselining, or a database reset. Flyway clean is disabled. Once released, add a new versioned migration instead of editing an applied one.

Check `http://localhost:8080/actuator/health` for `{"status":"UP"}`. Only health is exposed. Spring Security defaults still protect other routes; generated development authentication is not the planned JWT login and is not a deployable V1 authentication feature.

## Existing Phase 1 database or volume

**Do not delete your existing volume.** Docker initialization scripts do not run again on an existing database, and changing `.env` does not change existing account passwords. Phase 2 requires the new roles and `nexus` schema even if Phase 1 already connected successfully.

Back up any data first. Inspect the existing database as its administrator. For a Phase 1 database with no `nexus` schema or these roles, provision them once using the checked-in script. With Compose, after supplying the new password variables in `.env` and recreating the container with `docker compose up -d postgres`, run:

```sh
# Replace EXISTING_ADMIN with the actual administrator of the existing volume.
# Its identity/password are not changed by the new Compose configuration.
docker compose exec -T postgres psql -U EXISTING_ADMIN -d nexus_local -v ON_ERROR_STOP=1 < ops/postgres/01-roles.sql
```

For native PostgreSQL, export the two password variables and run `psql -U EXISTING_ADMIN -d YOUR_DATABASE -v ON_ERROR_STOP=1 -f ops/postgres/01-roles.sql`. Set DB_URL to that database afterward. The script intentionally fails if roles/schema already exist; inspect and reconcile rather than blindly rerunning or dropping objects. A database with legacy domain tables requires an explicit data/schema migration review: the new `nexus` schema does not import tables from `public`.

## IntelliJ run configuration

Open **Run → Edit Configurations → your existing NexusApplication configuration** (not Configuration Templates). Set Java 21 and these environment variables from your local `.env`:

```text
SPRING_PROFILES_ACTIVE=local
DB_URL=jdbc:postgresql://localhost:5433/nexus_local
DB_USERNAME=nexus_runtime
DB_PASSWORD=<your runtime password>
MIGRATION_DB_USERNAME=nexus_migrator
MIGRATION_DB_PASSWORD=<your migration password>
```

Use the environment-variable table to enter each separately. PostgreSQL must already be running and the roles/schema provisioned. The bootstrap administrator password belongs to Compose; the application does not need it.

## Profiles and optional fixtures

| Profile | Purpose |
| --- | --- |
| `local` | Developer startup with required runtime and migration credentials |
| `local,local-seed` | Explicit opt-in to synthetic catalog, disabled accounts, address, cart and opening stock |
| `test` | Integration tests inject disposable PostgreSQL credentials; no developer-database fallback |
| `prod` | External secrets, TLS-configured DB_URL, graceful shutdown and health probes; migrations only |

Activate exactly the intended profiles. Never combine `prod` with `local-seed` or point the seed profile at a deployed database. Default/prod Flyway locations contain only `db/migration`; fixtures are in `db/dev-seed`. A database seeded for development should remain a development database.

Seed accounts use `example.invalid`, are disabled and have unusable password hashes. They cannot log in. Opening inventory and its movement are inserted together. Reapplying seed SQL does not reset stock, change existing prices or duplicate opening movements. Registration/admin provisioning follows in Phase 3.

The application currently holds separate migration credentials at startup. Production migration execution can move to a deployment step in Phase 11; runtime credentials must retain their restricted grants.

## Persistence conventions

- UUID identifiers; UTC `Instant` timestamps; `@Version` on mutable user/address/product/inventory/cart/item rows.
- `created_at` and `updated_at` are managed for JPA writes. Future native update commands must update timestamps and increment versions explicitly.
- Monetary amounts are `bigint` INR paise. `Pricing` performs bounded integer calculations: unit price ≤100000000, quantity ≤99, ≤50 distinct lines, total ≤10000000000. Empty cart subtotal is zero; checkout must reject an empty cart.
- Foreign keys are mapped as UUIDs, avoiding implicit eager object graphs. Services resolve ownership and joins explicitly. Do not expose entity objects as API DTOs.
- Financial entries, inventory movements, failed attempts, order items and refunds are append only. Runtime cannot update/delete/truncate them. Order/payment updates are restricted to `status`; immutable columns are marked non-updatable in JPA.
- Products are deactivated, preserving historical references. Address/cart-item removal is allowed. Idempotency identity is immutable; only response/result fields can be filled by runtime.
- Composite foreign keys enforce cart/order ownership and payment/refund/order association. Partial unique indexes guard journal and stock movement duplicates.
- Cross-row totals, full refund amounts, stock-to-movement reconciliation, actor authorization and allowed status transitions require service transactions/locks in later phases. Database row constraints alone do not implement checkout.

Migration order: identity/catalog/cart → purchases/operation histories → runtime grants. There are no blanket privileges for future tables; future migrations must grant explicitly.

## Tests and build

```sh
# Unit tests only (no Docker required):
./mvnw test
# Complete build, including PostgreSQL integration tests:
./mvnw clean verify
```

Surefire runs `*Test`; Failsafe runs `*IT` during integration-test/verify and fails if no integration tests are discovered. Testcontainers creates a fresh PostgreSQL instance with separate migration/runtime roles and removes it on JVM exit. Docker unavailability fails the build; tests are not silently skipped. No TEST_DB_* variables or manually created test database are needed.

Integration fixtures use random IDs and rollback their transactions, including expected SQL failures via savepoints. The optimistic-lock test commits an isolated product that disappears with its disposable container. Tests never truncate existing databases or disable constraints.

Coverage includes fresh/repeated migrations, Hibernate validation of all entities, uniqueness/check/FK guards, runtime permissions, rollback, audit timestamps, optimistic conflicts and seed reapplication. Actual concurrent checkout/refund workflows are tested when their services exist.

To start the packaged build with the same application environment:

```sh
java -jar target/nexus-0.0.1-SNAPSHOT.jar
```

## Stop and intentional local reset

`docker compose stop` retains data. `docker compose down` removes containers/network and retains the named volume.

Only if you deliberately intend to discard this project's disposable local database:

```sh
docker compose down --volumes
docker compose up -d --wait postgres
```

This destroys that Compose database and reruns bootstrap on a new volume. It is not a migration or routine testing procedure.

## Repository and phase references

The backend is now its own Git repository, separate from the frontend. Shared documents remain in the adjacent `docs` directory; backend commits do not automatically include them. Keep real environment files, build output and local verification scripts/logs untracked.

See [backend phases](../docs/backend-development-plan.md), [database/API contract](<../docs/NexusCommerce V1 - Database and API Design.md>) and [system design](<../docs/NexusCommerce V1 - System Design.md>).

## Phase 2 verification — 8 October 2026

`JAVA_HOME=$(/usr/libexec/java_home -v 21) ./mvnw -B clean verify` passed: 3 unit tests and 11 PostgreSQL integration tests, no failures/errors/skips. Packaged local/prod startup passed with health UP and no seeds. Two consecutive local seed startups passed with exactly two disabled fixture accounts. The documented Compose bootstrap was exercised using a new disposable project/volume, then removed. Existing developer databases were untouched.
