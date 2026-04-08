---
name: database-migrations-with-golang-migrate
description: Production-grade database schema migration technique for Go backend projects using golang-migrate (go-migrate) with PostgreSQL. Directly inspired by the techschool/simplebank project (Makefile-driven CLI approach) while incorporating 2026 best practices: timestamp-based versioning to avoid merge conflicts, small focused migrations, always-reversible down scripts, idempotency where possible, and integration with the existing pgx + sqlx database layer. Migrations are CLI-first (via Makefile) for safety and control — never auto-run on app startup in production. Perfectly complements the database-access-with-pgx-and-sqlx skill.
---

# Database Migrations with golang-migrate

**Skill purpose**  
This skill defines the **official SemmiDev pattern** for managing PostgreSQL schema changes using `golang-migrate`.  

It follows the proven CLI + Makefile approach from `techschool/simplebank` while adding modern improvements:
- Timestamp versioning (instead of sequential) to prevent git merge conflicts in teams.
- Strict separation: migrations contain **only schema changes** (no seed data).
- Small, focused, reviewable migrations.
- Full support for `pgx` driver.
- Integration with your existing `internal/common/db` package.
- Docker + local dev workflow.

This keeps your database schema version-controlled, reproducible, and safe for production deploys.

## When to use this skill
- Any schema change: new tables, columns, indexes, constraints, views, functions.
- You need safe rollbacks (`down` migrations).
- You are setting up a new service or adding features that require DB changes.
- You want consistent local/CI/production migration workflow.

**Never**:
- Run migrations automatically on app startup in production (risky).
- Put data seeding or business logic inside migration files.
- Use `AutoMigrate` (GORM) or similar in production.
- Commit large, multi-purpose migrations.

## Installation

Install the CLI tool globally (or via Makefile):

```bash
go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest
```

(Use `postgres` tag for the official PostgreSQL driver. For native `pgx` support, you can also use the `pgx` driver, but `postgres` driver works perfectly with `pgxpool` via connection string.)

## Project Structure (recommended – matches simplebank style)

```
db/
├── migration/          ← All migration files live here
│   ├── 20260408100000_create_users_table.up.sql
│   ├── 20260408100000_create_users_table.down.sql
│   ├── 20260408103000_add_email_index.up.sql
│   └── ...
├── seed/               ← Optional: separate seed scripts (never in migrations)
└── query/              ← For sqlc or raw queries (if used)

internal/common/db/     ← Your existing DB setup (pgxpool + sqlx)
```

## Migration File Naming & Best Practices (2026)

Use **timestamp + descriptive name** (avoids conflicts when multiple developers work in parallel):

```
YYYYMMDDHHMMSS_description.up.sql
YYYYMMDDHHMMSS_description.down.sql
```

Examples:
- `20260408100000_create_users_table.up.sql`
- `20260408101500_add_balance_column.up.sql`

**Rules for writing migrations**:
- Keep each migration small and focused (one logical change).
- Always provide a working `.down.sql`.
- Use `IF NOT EXISTS` / `IF EXISTS` where safe for idempotency.
- Add comments explaining *why* the change is needed.
- For large tables: add column → backfill (separate migration) → add constraint.
- Use `CONCURRENTLY` for indexes in production to avoid locking.
- Test both `up` and `down` locally before committing.

## Makefile Targets (core of the skill – inspired by simplebank)

Add these to your root `Makefile`:

```makefile
DB_URL=postgresql://postgres:secret@localhost:5432/your_db_name?sslmode=disable

.PHONY: migrate-create
migrate-create: ## Create new migration files (timestamp + name)
	migrate create -ext sql -dir db/migration -seq -tz UTC $(NAME)

.PHONY: migrate-up
migrate-up: ## Apply all pending migrations
	migrate -path db/migration -database "$(DB_URL)" -verbose up

.PHONY: migrate-up1
migrate-up1: ## Apply only the next migration
	migrate -path db/migration -database "$(DB_URL)" -verbose up 1

.PHONY: migrate-down
migrate-down: ## Rollback all migrations
	migrate -path db/migration -database "$(DB_URL)" -verbose down

.PHONY: migrate-down1
migrate-down1: ## Rollback the last migration
	migrate -path db/migration -database "$(DB_URL)" -verbose down 1

.PHONY: migrate-status
migrate-status: ## Show migration status
	migrate -path db/migration -database "$(DB_URL)" -verbose status

.PHONY: migrate-goto
migrate-goto: ## Migrate to a specific version
	migrate -path db/migration -database "$(DB_URL)" -verbose goto $(VERSION)

.PHONY: migrate-force
migrate-force: ## Force set version (use carefully after manual fix)
	migrate -path db/migration -database "$(DB_URL)" -verbose force $(VERSION)
```

Usage examples:

```bash
# Create a new migration
make migrate-create NAME=create_users_table

# Run migrations locally
make migrate-up

# Rollback last change
make migrate-down1
```

## Local Development Workflow (Docker + Postgres)

Add these common targets too:

```makefile
.PHONY: postgres
postgres: ## Start Postgres in Docker
	docker run --name postgres -e POSTGRES_USER=postgres \
		-e POSTGRES_PASSWORD=secret -p 5432:5432 -d postgres:16-alpine

.PHONY: createdb
createdb: ## Create the database
	docker exec -it postgres createdb --username=postgres --owner=postgres your_db_name

.PHONY: dropdb
dropdb: ## Drop the database
	docker exec -it postgres dropdb --username=postgres your_db_name
```

Typical dev flow:
1. `make postgres`
2. `make createdb`
3. `make migrate-up`

## Integration with Your Existing DB Layer

Your `internal/common/db/db.go` remains unchanged — migrations run **outside** the application.

In CI/CD or deployment scripts, run `migrate-up` before starting the app.

For integration tests, you can run migrations programmatically if needed (optional helper):

```go
// internal/common/db/migration.go (optional)
import "github.com/golang-migrate/migrate/v4"

func RunMigrations(dsn string, migrationsPath string) error {
    m, err := migrate.New("file://"+migrationsPath, dsn)
    if err != nil {
        return err
    }
    defer m.Close()
    return m.Up()
}
```

**Recommendation**: Prefer CLI + Makefile in production/CI for visibility and control. Use code-based only in integration test setup.

## Best Practices & Decision Tree (2026 consensus)

**Always**:
- Commit migration files to git (schema is part of the codebase).
- Review migrations like production code (PRs).
- Test `down` migration locally.
- Use timestamp versioning for team work.
- Separate schema migrations from data seeding (use seed scripts or outbox pattern).

**Decision Tree**:
1. New table/column/index? → Create migration with `make migrate-create`.
2. Need to change data? → Do it in application code or separate seed script (not migration).
3. Large table change? → Multiple small migrations (add → backfill → constrain).
4. Production deploy? → Run `migrate-up` as part of deployment pipeline before starting app.
5. Rollback needed? → Use `migrate-down` or `migrate-goto`.

**Never**:
- Put `INSERT` statements for seed data in `.up.sql`.
- Run migrations automatically on every app startup in production.
- Use sequential numbering (`-seq` with numbers) in team environments.
- Ignore down migrations.

## CI/CD Integration Example

In GitHub Actions / GitLab CI:

```yaml
- name: Run database migrations
  run: |
    migrate -path db/migration \
      -database "${{ secrets.DB_URL }}" \
      -verbose up
```

## Integration Notes for Your Backend Projects
- Place migrations in `db/migration`.
- Use the same `DB_URL` format as your `pgxpool` config.
- Combine with `database-access-with-pgx-and-sqlx` (migrations create the schema your repos query).
- For zero-downtime: use blue-green or canary deploys + reversible migrations.
- Monitor migration history via the `schema_migrations` table created automatically.

Follow this skill and every schema change in your Go backends will be:
- Version-controlled
- Reproducible
- Safe to rollback
- Team-friendly
- Production-ready
