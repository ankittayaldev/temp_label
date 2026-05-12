# Implement `audit_logs` Hash Chain With Monthly Partitioning

## Final Reviewed Decisions

Approved:

- Application code can continue to query the logical parent table normally:

```sql
SELECT * FROM label_schema.audit_logs;
```

- PostgreSQL monthly range partitioning is accepted.
- The app should still read/write through the logical table `audit_logs`.
- PostgreSQL should calculate `row_hash` using `pgcrypto`.
- Python should calculate deterministic `nonce_uuid` using UUID5.
- The app should print the latest hash at startup.
- Migration must preserve all existing data.

Rejected:

- No `audit_chain_checkpoints` table.
- No `audit_chain_state` table.
- No `audit_chain_secret` table.
- No `prev_hash` column.
- No `hash_algo` column.
- No `chain_version` column.
- No external checkpoint anchoring.
- No append-only trigger enforcement for `UPDATE` / `DELETE`.

Final logical table:

```text
audit_logs
```

PostgreSQL will create child partition tables internally, but the application will use only the logical parent table.

---

## Final Table Structure

Old structure:

```sql
CREATE TABLE IF NOT EXISTS {schema_quoted}.audit_logs (
    id VARCHAR PRIMARY KEY,
    event_type VARCHAR NOT NULL,
    entity_type VARCHAR NULL,
    entity_id VARCHAR NULL,
    project_id VARCHAR NULL REFERENCES {schema_quoted}.projects(id),
    user_id INTEGER NULL REFERENCES {schema_quoted}.users(id),
    ip_address VARCHAR NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    details JSONB NULL
);
```

New structure:

```sql
CREATE TABLE IF NOT EXISTS {schema_quoted}.audit_logs (
    id VARCHAR NOT NULL,
    chain_seq BIGINT GENERATED ALWAYS AS IDENTITY,
    event_type VARCHAR NOT NULL,
    entity_type VARCHAR NULL,
    entity_id VARCHAR NULL,
    project_id VARCHAR NULL REFERENCES {schema_quoted}.projects(id),
    user_id INTEGER NULL REFERENCES {schema_quoted}.users(id),
    ip_address VARCHAR NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT clock_timestamp(),
    details JSONB NULL,
    nonce_uuid VARCHAR NOT NULL,
    row_hash TEXT NOT NULL,
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);
```

Only added columns:

```sql
chain_seq BIGINT GENERATED ALWAYS AS IDENTITY,
nonce_uuid VARCHAR NOT NULL,
row_hash TEXT NOT NULL
```

Why `row_hash` can still be `NOT NULL`:

- the app does not send `row_hash`
- PostgreSQL identity/default values are available to the row during `BEFORE INSERT`
- the `BEFORE INSERT` trigger assigns `NEW.row_hash`
- the `NOT NULL` check happens after the trigger has populated `NEW.row_hash`

Only changed existing column:

```sql
created_at TIMESTAMPTZ NOT NULL DEFAULT clock_timestamp()
```

Primary key change:

```sql
PRIMARY KEY (id, created_at)
```

Reason: PostgreSQL requires every primary key or unique constraint on a partitioned table to include the partition key. Since the partition key is `created_at`, `id` alone cannot remain the parent table primary key.

---

## Why `CREATE NEW + BACKFILL` Is Required

Preference: `ALTER TABLE` would be simpler.

Problem: PostgreSQL does not support directly converting an existing normal table into a partitioned table using a simple `ALTER TABLE audit_logs PARTITION BY RANGE (...)`.

So there are two options:

### Option A: ALTER only, no partitioning

This is possible:

```sql
ALTER TABLE label_schema.audit_logs
ADD COLUMN chain_seq BIGINT GENERATED ALWAYS AS IDENTITY;

ALTER TABLE label_schema.audit_logs
ADD COLUMN nonce_uuid VARCHAR;

ALTER TABLE label_schema.audit_logs
ADD COLUMN row_hash TEXT;

ALTER TABLE label_schema.audit_logs
ALTER COLUMN created_at SET DEFAULT clock_timestamp();
```

But this does **not** give monthly partitions.

### Option B: New partitioned table + backfill

This is required if monthly partitions are accepted.

Migration flow:

1. Rename existing `audit_logs` to `audit_logs_old`.
2. Create new partitioned `audit_logs` parent table.
3. Create monthly partitions.
4. Add PostgreSQL trigger to compute `row_hash`.
5. Backfill old rows into new `audit_logs`.
6. Preserve old `created_at` values.
7. Generate `nonce_uuid` in Python during backfill.
8. Let PostgreSQL compute `row_hash` during backfill.
9. Verify old and new row counts.
10. Keep `audit_logs_old` for one release as rollback safety.

Final recommendation: use Option B because partitioning is accepted.

---

## Hash Formula

Final formula:

```text
row_hash = HMAC_SHA256(jwt_secret, canonical_payload + previous_row_hash)
```

No `prev_hash` column is stored.

The previous row hash is found dynamically:

```sql
SELECT row_hash
FROM audit_logs
WHERE chain_seq < NEW.chain_seq
ORDER BY chain_seq DESC
LIMIT 1;
```

For the first row:

```text
previous_row_hash = 0000000000000000000000000000000000000000000000000000000000000000
```

---

## Who Calculates What

### Python Calculates `nonce_uuid`

Python already has the event values before insert.

Use UUID5, similar to the existing pattern in `label/routes/rag.py`:

```python
point_id = uuid.uuid5(uuid.NAMESPACE_URL, f"{id_seed}:{chunk['chunk_no']}").hex
```

For audit logs:

```python
nonce_uuid = uuid.uuid5(uuid.NAMESPACE_URL, seed).hex
```

### PostgreSQL Calculates `row_hash`

The app does **not** send `row_hash`.

The app inserts normal audit columns plus `nonce_uuid`.

A PostgreSQL `BEFORE INSERT` trigger fills `row_hash` using `pgcrypto.hmac()`.

Reason for `BEFORE INSERT`: the trigger assigns `NEW.row_hash` before PostgreSQL enforces `row_hash TEXT NOT NULL`.

---

## Secret Handling For PostgreSQL Hashing

PostgreSQL needs the secret to calculate HMAC.

Do **not** create a new secret table.

Do **not** use `users.password_hash` as the preferred design because:

- admin password hash can change
- user `id = 1` may not always be the admin in every environment
- password reset would make verification ambiguous
- it couples audit integrity to user lifecycle

Recommended approach:

1. Python reads `settings.jwt_secret`.
2. SQLAlchemy sets it as a PostgreSQL session variable on every DB connection.
3. PostgreSQL trigger reads it using `current_setting('app.jwt_secret', true)`.

This keeps the secret out of `audit_logs` and avoids a new table.

Important: do not rotate `JWT_SECRET` without a migration plan. Since `row_hash` is HMAC-based, old rows can only be recomputed with the same secret that created them. If `JWT_SECRET` changes, old hash verification will fail unless the old secret is retained for verification or the chain is intentionally re-baselined.

---

## Step 1: Update SQLAlchemy Model

File:

```text
label/DAL/models/tables.py
```

Update import:

```python
from sqlalchemy import BigInteger, Boolean, Column, DateTime, ForeignKey, Identity, Integer, MetaData, Numeric, String, text
```

Replace `AuditLog` with:

```python
class AuditLog(Base):
    __tablename__ = "audit_logs"
    __table_args__ = {
        "schema": SCHEMA_NAME,
        "extend_existing": True,
        "postgresql_partition_by": "RANGE (created_at)",
    }

    id = Column(String, primary_key=True)
    chain_seq = Column(BigInteger, Identity(always=True), nullable=False)
    event_type = Column(String, nullable=False)
    entity_type = Column(String)
    entity_id = Column(String)
    project_id = Column(String, ForeignKey("projects.id"))
    user_id = Column(Integer, ForeignKey("users.id"))
    ip_address = Column(String)
    created_at = Column(
        DateTime(timezone=True),
        primary_key=True,
        server_default=text("clock_timestamp()"),
        nullable=False,
    )
    details = Column(JSONB)
    nonce_uuid = Column(String, nullable=False)
    row_hash = Column(String, nullable=False)
```

Why `__table_args__` is repeated here:

The base class already has schema settings, but once `AuditLog` needs `postgresql_partition_by`, it should explicitly include:

```python
"schema": SCHEMA_NAME,
"extend_existing": True,
```

Otherwise the model can accidentally lose the configured schema.

---

## Step 2: Add UUID5 Helper In `db.py`

File:

```text
label/db.py
```

Add imports near the top:

```python
import json
import uuid
```

Keep existing:

```python
from uuid6 import uuid7
```

Add helper near `_new_id()`:

```python
def _audit_nonce_uuid(
    *,
    log_id: str,
    event_type: str,
    entity_type: Optional[str],
    entity_id: Optional[str],
    project_id: Optional[str],
    user_id: Optional[Any],
    ip_address: Optional[str],
    details: Optional[Dict[str, Any]],
) -> str:
    """Build deterministic UUID5 nonce from values known in Python before DB insert."""
    normalized_details = _strip_nul_in_value(details or {})
    details_seed = json.dumps(
        normalized_details,
        sort_keys=True,
        separators=(",", ":"),
        ensure_ascii=False,
        default=str,
    )
    seed = "|".join(
        [
            log_id,
            event_type or "",
            entity_type or "",
            entity_id or "",
            project_id or "",
            str(user_id or ""),
            ip_address or "",
            details_seed,
        ]
    )
    return uuid.uuid5(uuid.NAMESPACE_URL, seed).hex
```

Why include `log_id`?

Because two identical audit events should not accidentally produce the same `nonce_uuid`. Since `log_id` is generated before insert, UUID5 remains deterministic for that row while still being practically unique.

---

## Step 3: Update `insert_audit_log()`

Current function inserts only old fields.

Update it so Python sends `nonce_uuid`, but still does **not** send `chain_seq` or `row_hash`.

File:

```text
label/db.py
```

Replace `insert_audit_log()` with:

```python
def insert_audit_log(
    *,
    event_type: str,
    entity_type: Optional[str] = None,
    entity_id: Optional[str] = None,
    project_id: Optional[str] = None,
    user_id: Optional[Any] = None,
    ip_address: Optional[str] = None,
    details: Optional[Dict[str, Any]] = None,
    conn=None,
) -> str:
    log_id = _new_id()
    payload = details or {}
    resolved_user_id = _coerce_user_id(user_id)
    nonce_uuid = _audit_nonce_uuid(
        log_id=log_id,
        event_type=event_type,
        entity_type=entity_type,
        entity_id=entity_id,
        project_id=project_id,
        user_id=resolved_user_id,
        ip_address=ip_address,
        details=payload,
    )

    with _ensure_session(conn) as db:
        CRUD.create(
            session=db,
            model_name=AuditLog,
            commit=False,
            refresh=False,
            id=log_id,
            event_type=event_type,
            entity_type=entity_type,
            entity_id=entity_id,
            project_id=project_id,
            user_id=resolved_user_id,
            ip_address=ip_address,
            details=payload,
            nonce_uuid=nonce_uuid,
        )
    return log_id
```

PostgreSQL fills these automatically:

```text
chain_seq
created_at
row_hash
```

---

## Step 4: Set `jwt_secret` On Every PostgreSQL Connection

File:

```text
label/db.py
```

Current code registers the schema connection event after one initial connection is already used.

For the audit hash trigger, register the event before opening the first connection.

Update `get_engine()` like this:

```python
def get_engine():
    """Return the SQLAlchemy engine used for raw database connections."""
    global _ENGINE
    with _ENGINE_LOCK:
        if _ENGINE is None:
            try:
                s = get_settings()
            except ValueError as exc:
                raise RuntimeError(str(exc)) from exc
            if not s.database_url:
                raise RuntimeError("DATABASE_URL is not configured. Set DATABASE_URL to a valid PostgreSQL URI.")
            if not s.jwt_secret:
                raise RuntimeError("JWT_SECRET is required for audit log hash chaining.")

            _ENGINE = create_engine(
                s.database_url,
                pool_size=10,
                max_overflow=20,
                pool_timeout=30,
                pool_recycle=300,
                pool_pre_ping=True,
                connect_args={"connect_timeout": 5},
            )
            event.listen(_ENGINE, "connect", _configure_schema_connection(s.schema_name, s.jwt_secret))

            with _ENGINE.begin() as connection:
                connection.execute(CreateSchema(s.schema_name, if_not_exists=True))
                schema_quoted = connection.dialect.identifier_preparer.quote(s.schema_name)
                connection.execute(text(f"SET search_path TO {schema_quoted}"))
                connection.execute(
                    text("SELECT set_config('app.jwt_secret', :jwt_secret, false)"),
                    {"jwt_secret": s.jwt_secret},
                )
    return _ENGINE
```

Update `_configure_schema_connection()`:

```python
def _configure_schema_connection(schema_name: str, jwt_secret: str):
    def _on_connect(dbapi_connection, connection_record):
        cursor = dbapi_connection.cursor()
        try:
            cursor.execute(sql.SQL("SET search_path TO {}").format(sql.Identifier(schema_name)))
            cursor.execute("SELECT set_config('app.jwt_secret', %s, false)", (jwt_secret,))
            dbapi_connection.commit()
        finally:
            cursor.close()

    return _on_connect
```

This makes the JWT secret available to PostgreSQL trigger code as:

```sql
current_setting('app.jwt_secret', true)
```

---

## Step 5: PostgreSQL Trigger To Compute `row_hash`

PostgreSQL extension:

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
```

Trigger function:

```sql
CREATE OR REPLACE FUNCTION label_schema.audit_logs_set_row_hash()
RETURNS trigger
LANGUAGE plpgsql
AS $$
DECLARE
    previous_row_hash text;
    jwt_secret text;
    canonical_payload text;
BEGIN
    PERFORM pg_advisory_xact_lock(hashtext('audit_logs_hash_chain'));

    jwt_secret := current_setting('app.jwt_secret', true);
    IF jwt_secret IS NULL OR jwt_secret = '' THEN
        RAISE EXCEPTION 'app.jwt_secret is required to compute audit_logs.row_hash';
    END IF;

    SELECT row_hash
    INTO previous_row_hash
    FROM label_schema.audit_logs
    WHERE chain_seq < NEW.chain_seq
    ORDER BY chain_seq DESC
    LIMIT 1;

    previous_row_hash := COALESCE(previous_row_hash, repeat('0', 64));

    canonical_payload := jsonb_build_object(
        'chain_seq', NEW.chain_seq,
        'id', NEW.id,
        'event_type', NEW.event_type,
        'entity_type', NEW.entity_type,
        'entity_id', NEW.entity_id,
        'project_id', NEW.project_id,
        'user_id', NEW.user_id,
        'ip_address', NEW.ip_address,
        'created_at', NEW.created_at,
        'details', COALESCE(NEW.details, '{}'::jsonb),
        'nonce_uuid', NEW.nonce_uuid,
        'previous_row_hash', previous_row_hash
    )::text;

    NEW.row_hash := encode(
        hmac(canonical_payload::bytea, jwt_secret::bytea, 'sha256'),
        'hex'
    );

    RETURN NEW;
END;
$$;
```

Trigger:

```sql
CREATE TRIGGER trg_audit_logs_set_row_hash
BEFORE INSERT ON label_schema.audit_logs
FOR EACH ROW
EXECUTE FUNCTION label_schema.audit_logs_set_row_hash();
```

Why use advisory lock?

It serializes audit hash generation without a separate `audit_chain_state` table.

```sql
PERFORM pg_advisory_xact_lock(hashtext('audit_logs_hash_chain'));
```

This keeps the one-table design while avoiding two concurrent inserts reading the same latest hash.

---

## Step 6: Monthly Partition Helper

Create a PostgreSQL function to create partitions.

This is not an audit data table. It is only a helper function.

```sql
CREATE OR REPLACE FUNCTION label_schema.ensure_audit_logs_month_partition(month_start date)
RETURNS void
LANGUAGE plpgsql
AS $$
DECLARE
    partition_name text;
    partition_fqn text;
    from_ts timestamptz;
    to_ts timestamptz;
BEGIN
    from_ts := date_trunc('month', month_start)::timestamptz;
    to_ts := from_ts + interval '1 month';
    partition_name := format('audit_logs_%s', to_char(from_ts, 'YYYY_MM'));
    partition_fqn := format('%I.%I', 'label_schema', partition_name);

    EXECUTE format(
        'CREATE TABLE IF NOT EXISTS %s PARTITION OF label_schema.audit_logs FOR VALUES FROM (%L) TO (%L)',
        partition_fqn,
        from_ts,
        to_ts
    );

    EXECUTE format(
        'CREATE INDEX IF NOT EXISTS %I ON %s (chain_seq)',
        partition_name || '_chain_seq_idx',
        partition_fqn
    );

    EXECUTE format(
        'CREATE INDEX IF NOT EXISTS %I ON %s (project_id, created_at)',
        partition_name || '_project_created_idx',
        partition_fqn
    );

    EXECUTE format(
        'CREATE INDEX IF NOT EXISTS %I ON %s (entity_type, entity_id, created_at)',
        partition_name || '_entity_created_idx',
        partition_fqn
    );
END;
$$;
```

Create current and next 12 months:

```sql
SELECT label_schema.ensure_audit_logs_month_partition(
    (date_trunc('month', now()) + (n || ' month')::interval)::date
)
FROM generate_series(0, 12) AS n;
```

---

## Step 7: Alembic Migration

Create a new migration under:

```text
label/alembic/versions/
```

Example full migration skeleton:

```python
from __future__ import annotations

import json
import os
import uuid
from typing import Any

from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects import postgresql

try:
    from settings import resolve_schema_name
    SCHEMA_NAME = resolve_schema_name()
except Exception:
    SCHEMA_NAME = "label_schema"


def _details_seed(details: Any) -> str:
    return json.dumps(
        details or {},
        sort_keys=True,
        separators=(",", ":"),
        ensure_ascii=False,
        default=str,
    )


def _audit_nonce_uuid(
    *,
    log_id: str,
    event_type: str,
    entity_type: str | None,
    entity_id: str | None,
    project_id: str | None,
    user_id: int | None,
    ip_address: str | None,
    details: Any,
) -> str:
    seed = "|".join(
        [
            log_id,
            event_type or "",
            entity_type or "",
            entity_id or "",
            project_id or "",
            str(user_id or ""),
            ip_address or "",
            _details_seed(details),
        ]
    )
    return uuid.uuid5(uuid.NAMESPACE_URL, seed).hex


def upgrade():
    jwt_secret = os.environ.get("JWT_SECRET", "").strip()
    if not jwt_secret:
        raise RuntimeError("JWT_SECRET is required to migrate audit_logs hash chain")

    bind = op.get_bind()
    preparer = bind.dialect.identifier_preparer
    schema_quoted = preparer.quote_schema(SCHEMA_NAME)

    bind.execute(
        sa.text("SELECT set_config('app.jwt_secret', :jwt_secret, false)"),
        {"jwt_secret": jwt_secret},
    )

    op.execute("CREATE EXTENSION IF NOT EXISTS pgcrypto")

    # 1. Rename old table.
    op.execute(f"ALTER TABLE {schema_quoted}.audit_logs RENAME TO audit_logs_old")

    # 2. Rename old constraints to avoid name collisions with the new table.
    op.execute(f"""
    DO $$
    BEGIN
        IF EXISTS (
            SELECT 1
            FROM pg_constraint c
            JOIN pg_namespace n ON n.oid = c.connamespace
            WHERE n.nspname = '{SCHEMA_NAME}'
              AND c.conname = 'audit_logs_pkey'
        ) THEN
            ALTER TABLE {schema_quoted}.audit_logs_old
            RENAME CONSTRAINT audit_logs_pkey TO audit_logs_old_pkey;
        END IF;
    END $$;
    """)

    # 3. Create new partitioned parent table.
    op.execute(f"""
    CREATE TABLE {schema_quoted}.audit_logs (
        id VARCHAR NOT NULL,
        chain_seq BIGINT GENERATED ALWAYS AS IDENTITY,
        event_type VARCHAR NOT NULL,
        entity_type VARCHAR NULL,
        entity_id VARCHAR NULL,
        project_id VARCHAR NULL REFERENCES {schema_quoted}.projects(id),
        user_id INTEGER NULL REFERENCES {schema_quoted}.users(id),
        ip_address VARCHAR NULL,
        created_at TIMESTAMPTZ NOT NULL DEFAULT clock_timestamp(),
        details JSONB NULL,
        nonce_uuid VARCHAR NOT NULL,
        row_hash TEXT NOT NULL,
        PRIMARY KEY (id, created_at)
    ) PARTITION BY RANGE (created_at)
    """)

    # 4. Create partition helper function.
    op.execute(f"""
    CREATE OR REPLACE FUNCTION {schema_quoted}.ensure_audit_logs_month_partition(month_start date)
    RETURNS void
    LANGUAGE plpgsql
    AS $$
    DECLARE
        partition_name text;
        partition_fqn text;
        from_ts timestamptz;
        to_ts timestamptz;
    BEGIN
        from_ts := date_trunc('month', month_start)::timestamptz;
        to_ts := from_ts + interval '1 month';
        partition_name := format('audit_logs_%s', to_char(from_ts, 'YYYY_MM'));
        partition_fqn := format('%I.%I', '{SCHEMA_NAME}', partition_name);

        EXECUTE format(
            'CREATE TABLE IF NOT EXISTS %s PARTITION OF {schema_quoted}.audit_logs FOR VALUES FROM (%L) TO (%L)',
            partition_fqn,
            from_ts,
            to_ts
        );

        EXECUTE format(
            'CREATE INDEX IF NOT EXISTS %I ON %s (chain_seq)',
            partition_name || '_chain_seq_idx',
            partition_fqn
        );

        EXECUTE format(
            'CREATE INDEX IF NOT EXISTS %I ON %s (project_id, created_at)',
            partition_name || '_project_created_idx',
            partition_fqn
        );

        EXECUTE format(
            'CREATE INDEX IF NOT EXISTS %I ON %s (entity_type, entity_id, created_at)',
            partition_name || '_entity_created_idx',
            partition_fqn
        );
    END;
    $$;
    """)

    # 5. Create partitions for all old data months.
    old_months = bind.execute(sa.text(f"""
        SELECT DISTINCT date_trunc('month', created_at)::date AS month_start
        FROM {schema_quoted}.audit_logs_old
        ORDER BY month_start
    """)).scalars().all()

    for month_start in old_months:
        bind.execute(
            sa.text(f"SELECT {schema_quoted}.ensure_audit_logs_month_partition(:month_start)"),
            {"month_start": month_start},
        )

    # 6. Create current and next 12 monthly partitions.
    op.execute(f"""
    SELECT {schema_quoted}.ensure_audit_logs_month_partition(
        (date_trunc('month', now()) + (n || ' month')::interval)::date
    )
    FROM generate_series(0, 12) AS n
    """)

    # 7. Create hash trigger function.
    op.execute(f"""
    CREATE OR REPLACE FUNCTION {schema_quoted}.audit_logs_set_row_hash()
    RETURNS trigger
    LANGUAGE plpgsql
    AS $$
    DECLARE
        previous_row_hash text;
        jwt_secret text;
        canonical_payload text;
    BEGIN
        PERFORM pg_advisory_xact_lock(hashtext('audit_logs_hash_chain'));

        jwt_secret := current_setting('app.jwt_secret', true);
        IF jwt_secret IS NULL OR jwt_secret = '' THEN
            RAISE EXCEPTION 'app.jwt_secret is required to compute audit_logs.row_hash';
        END IF;

        SELECT row_hash
        INTO previous_row_hash
        FROM {schema_quoted}.audit_logs
        WHERE chain_seq < NEW.chain_seq
        ORDER BY chain_seq DESC
        LIMIT 1;

        previous_row_hash := COALESCE(previous_row_hash, repeat('0', 64));

        canonical_payload := jsonb_build_object(
            'chain_seq', NEW.chain_seq,
            'id', NEW.id,
            'event_type', NEW.event_type,
            'entity_type', NEW.entity_type,
            'entity_id', NEW.entity_id,
            'project_id', NEW.project_id,
            'user_id', NEW.user_id,
            'ip_address', NEW.ip_address,
            'created_at', NEW.created_at,
            'details', COALESCE(NEW.details, '{{}}'::jsonb),
            'nonce_uuid', NEW.nonce_uuid,
            'previous_row_hash', previous_row_hash
        )::text;

        NEW.row_hash := encode(
            hmac(canonical_payload::bytea, jwt_secret::bytea, 'sha256'),
            'hex'
        );

        RETURN NEW;
    END;
    $$;
    """)

    # 8. Attach trigger to partitioned parent table.
    op.execute(f"""
    CREATE TRIGGER trg_audit_logs_set_row_hash
    BEFORE INSERT ON {schema_quoted}.audit_logs
    FOR EACH ROW
    EXECUTE FUNCTION {schema_quoted}.audit_logs_set_row_hash()
    """)

    # 8.1. Recreate the existing project last_updated trigger on the new audit_logs table.
    # The old trigger moved with audit_logs_old during table rename.
    op.execute(f"""
    CREATE TRIGGER trg_projects_last_updated_audit
    AFTER INSERT OR UPDATE ON {schema_quoted}.audit_logs
    FOR EACH ROW
    WHEN (NEW.project_id IS NOT NULL)
    EXECUTE FUNCTION {schema_quoted}.update_project_last_updated()
    """)

    # 9. Backfill old rows. Python computes nonce_uuid; PostgreSQL trigger computes row_hash.
    old_rows = bind.execute(sa.text(f"""
        SELECT
            id,
            event_type,
            entity_type,
            entity_id,
            project_id,
            user_id,
            ip_address,
            created_at,
            details
        FROM {schema_quoted}.audit_logs_old
        ORDER BY created_at ASC, id ASC
    """)).mappings().all()

    insert_stmt = sa.text(f"""
        INSERT INTO {schema_quoted}.audit_logs (
            id,
            event_type,
            entity_type,
            entity_id,
            project_id,
            user_id,
            ip_address,
            created_at,
            details,
            nonce_uuid
        ) VALUES (
            :id,
            :event_type,
            :entity_type,
            :entity_id,
            :project_id,
            :user_id,
            :ip_address,
            :created_at,
            :details,
            :nonce_uuid
        )
    """).bindparams(sa.bindparam("details", type_=postgresql.JSONB))

    for old_row in old_rows:
        row = dict(old_row)
        row["nonce_uuid"] = _audit_nonce_uuid(
            log_id=row["id"],
            event_type=row["event_type"],
            entity_type=row["entity_type"],
            entity_id=row["entity_id"],
            project_id=row["project_id"],
            user_id=row["user_id"],
            ip_address=row["ip_address"],
            details=row["details"],
        )
        bind.execute(insert_stmt, row)

    # 10. Verify row counts.
    old_count = bind.execute(sa.text(f"SELECT count(*) FROM {schema_quoted}.audit_logs_old")).scalar_one()
    new_count = bind.execute(sa.text(f"SELECT count(*) FROM {schema_quoted}.audit_logs")).scalar_one()
    if old_count != new_count:
        raise RuntimeError(f"audit_logs migration count mismatch: old={old_count}, new={new_count}")

    missing_hash_count = bind.execute(
        sa.text(f"SELECT count(*) FROM {schema_quoted}.audit_logs WHERE row_hash IS NULL OR row_hash = ''")
    ).scalar_one()
    if missing_hash_count:
        raise RuntimeError(f"audit_logs migration produced rows without row_hash: {missing_hash_count}")

    # 11. Keep audit_logs_old for rollback. Drop later after production verification.


def downgrade():
    bind = op.get_bind()
    preparer = bind.dialect.identifier_preparer
    schema_quoted = preparer.quote_schema(SCHEMA_NAME)

    op.execute(f"DROP TABLE IF EXISTS {schema_quoted}.audit_logs CASCADE")
    op.execute(f"ALTER TABLE {schema_quoted}.audit_logs_old RENAME TO audit_logs")
```

---

## Step 8: Ensure Future Monthly Partitions At Startup

Note: the existing project `last_updated` trigger must also exist on the new partitioned `audit_logs` parent table. The migration above recreates:

```sql
CREATE TRIGGER trg_projects_last_updated_audit
AFTER INSERT OR UPDATE ON label_schema.audit_logs
FOR EACH ROW
WHEN (NEW.project_id IS NOT NULL)
EXECUTE FUNCTION label_schema.update_project_last_updated();
```

The migration creates current and next 12 months.

To avoid future insert failures, add a startup helper that ensures current and next month exist.

File:

```text
label/db.py
```

Add:

```python
def ensure_audit_log_partitions() -> None:
    settings = get_settings()
    engine = get_engine()
    schema_quoted = engine.dialect.identifier_preparer.quote(settings.schema_name)
    with engine.begin() as connection:
        connection.execute(
            text(
                f"""
                SELECT {schema_quoted}.ensure_audit_logs_month_partition(
                    (date_trunc('month', now()) + (n || ' month')::interval)::date
                )
                FROM generate_series(0, 2) AS n
                """
            )
        )
```

Call it from both successful `init_db()` paths.

Current code returns early when the admin user already exists:

```python
with db_session() as session:
    try:
        _ensure_admin_user(session)
        return
    except SQLAlchemyProgrammingError as exc:
        logger.warning("Admin lookup failed; running migrations.", exc_info=exc)
        session.rollback()
        _run_migrations()
```

Change that block to:

```python
with db_session() as session:
    try:
        _ensure_admin_user(session)
        _ensure_project_last_updated_triggers()
        ensure_audit_log_partitions()
        return
    except SQLAlchemyProgrammingError as exc:
        logger.warning("Admin lookup failed; running migrations.", exc_info=exc)
        session.rollback()
        _run_migrations()
```

Also keep it in the post-migration path:

```python
with db_session() as post_migration_session:
    try:
        _ensure_admin_user(post_migration_session)
        _ensure_project_last_updated_triggers()
        ensure_audit_log_partitions()
    except (SQLAlchemyProgrammingError, IntegrityError) as exc:
        post_migration_session.rollback()
        logger.warning(
            "Admin lookup failed after migrations for %s; verify schema, permissions, and admin config.",
            settings.admin_email,
            exc_info=exc,
        )
        raise
```

---

## Step 9: Print Latest Hash At App Startup

File:

```text
label/db.py
```

Add:

```python
def log_latest_audit_hash() -> None:
    settings = get_settings()
    engine = get_engine()
    schema_quoted = engine.dialect.identifier_preparer.quote(settings.schema_name)
    try:
        with engine.begin() as connection:
            row = connection.execute(
                text(
                    f"""
                    SELECT chain_seq, row_hash, created_at
                    FROM {schema_quoted}.audit_logs
                    ORDER BY chain_seq DESC
                    LIMIT 1
                    """
                )
            ).mappings().first()
    except Exception as exc:
        logger.warning("AUDIT_CHAIN_STARTUP_CHECKPOINT unavailable: %s", exc)
        return

    if not row:
        logger.info("AUDIT_CHAIN_STARTUP_CHECKPOINT empty=true")
        return

    logger.info(
        "AUDIT_CHAIN_STARTUP_CHECKPOINT chain_seq=%s row_hash=%s created_at=%s",
        row["chain_seq"],
        row["row_hash"],
        row["created_at"],
    )
```

File:

```text
label/main.py
```

Change import:

```python
from db import init_db, log_latest_audit_hash
```

Call after `init_db()`:

```python
def create_app() -> FastAPI:
    """Create and configure FastAPI application."""
    s = get_settings()
    ensure_dirs([s.data_dir, s.upload_dir, s.processed_dir, s.cache_dir, s.asset_dir])
    init_db()
    log_latest_audit_hash()

    app = FastAPI(
        title="Label Update Backend",
        version="1.0.0",
    )
```

This is not external checkpoint anchoring. It only prints the current latest hash.

---

## Step 10: Do Not Delete Audit Logs During Project Delete

Current code in `label/routes/project.py` deletes audit logs for a project:

```python
deleted_rows["audit_logs"] = (
    conn.query(AuditLog)
    .filter(AuditLog.project_id == project_id)
    .delete(synchronize_session=False)
)
```

If audit rows are deleted, the hash chain becomes unverifiable because later rows were computed using hashes that may no longer exist.

Even though append-only trigger enforcement is rejected, the application should not intentionally delete audit logs.

Change to:

```python
deleted_rows["audit_logs"] = 0
```

Or:

```python
# Do not delete audit logs. They are retained for hash-chain verification.
deleted_rows["audit_logs"] = 0
```

Then keep the existing delete audit event:

```python
insert_audit_log(
    event_type="v2_project_delete_full",
    entity_type="project",
    entity_id=project_id,
    project_id=None,
    user_id=user.user_id,
    ip_address=request.client.host if request.client else None,
    details={...},
    conn=conn,
)
```

---

## Step 11: Verification SQL

Before verification, the DB session must have `app.jwt_secret` set.

In app connections this is automatic from `_configure_schema_connection()`.

Manual session:

```sql
SELECT set_config('app.jwt_secret', '<JWT_SECRET_VALUE>', false);
```

Find broken hashes:

```sql
WITH ordered AS (
    SELECT
        chain_seq,
        id,
        event_type,
        entity_type,
        entity_id,
        project_id,
        user_id,
        ip_address,
        created_at,
        details,
        nonce_uuid,
        row_hash,
        COALESCE(
            lag(row_hash) OVER (ORDER BY chain_seq),
            repeat('0', 64)
        ) AS previous_row_hash
    FROM label_schema.audit_logs
), recomputed AS (
    SELECT
        chain_seq,
        id,
        row_hash,
        encode(
            hmac(
                jsonb_build_object(
                    'chain_seq', chain_seq,
                    'id', id,
                    'event_type', event_type,
                    'entity_type', entity_type,
                    'entity_id', entity_id,
                    'project_id', project_id,
                    'user_id', user_id,
                    'ip_address', ip_address,
                    'created_at', created_at,
                    'details', COALESCE(details, '{}'::jsonb),
                    'nonce_uuid', nonce_uuid,
                    'previous_row_hash', previous_row_hash
                )::text::bytea,
                current_setting('app.jwt_secret')::bytea,
                'sha256'
            ),
            'hex'
        ) AS computed_row_hash
    FROM ordered
)
SELECT *
FROM recomputed
WHERE row_hash IS DISTINCT FROM computed_row_hash
ORDER BY chain_seq;
```

Expected result:

```text
0 rows
```

Latest hash query:

```sql
SELECT chain_seq, row_hash, created_at
FROM label_schema.audit_logs
ORDER BY chain_seq DESC
LIMIT 1;
```

Partition list query:

```sql
SELECT
    nmsp_parent.nspname AS parent_schema,
    parent.relname AS parent_table,
    nmsp_child.nspname AS child_schema,
    child.relname AS child_table
FROM pg_inherits
JOIN pg_class parent ON pg_inherits.inhparent = parent.oid
JOIN pg_class child ON pg_inherits.inhrelid = child.oid
JOIN pg_namespace nmsp_parent ON nmsp_parent.oid = parent.relnamespace
JOIN pg_namespace nmsp_child ON nmsp_child.oid = child.relnamespace
WHERE nmsp_parent.nspname = 'label_schema'
  AND parent.relname = 'audit_logs'
ORDER BY child.relname;
```

---

## Step 12: Runtime Insert Example

Application code can still call the same helper:

```python
insert_audit_log(
    event_type="PROJECT_CREATED",
    entity_type="project",
    entity_id=project.id,
    project_id=project.id,
    user_id=current_user.id,
    ip_address=request.client.host if request.client else None,
    details={"project_name": project.project_name},
    conn=db,
)
```

Under the hood:

1. Python creates `id`.
2. Python creates `nonce_uuid` using UUID5.
3. App inserts row into `audit_logs`.
4. PostgreSQL assigns `chain_seq`.
5. PostgreSQL assigns `created_at` if not supplied.
6. PostgreSQL trigger reads previous `row_hash`.
7. PostgreSQL trigger computes new `row_hash`.
8. Insert completes.

---

## Final Rollout Order

1. Update `AuditLog` model.
2. Add Python UUID5 nonce helper.
3. Update `insert_audit_log()` to send `nonce_uuid` only.
4. Update DB connection setup to set `app.jwt_secret`.
5. Add latest hash startup logging.
6. Remove intentional audit-log deletion from project full delete.
7. Add Alembic migration.
8. Migration renames old table to `audit_logs_old`.
9. Migration creates new partitioned `audit_logs`.
10. Migration creates monthly partitions.
11. Migration creates PostgreSQL hash trigger.
12. Migration backfills old rows with Python-generated `nonce_uuid`.
13. PostgreSQL trigger computes `row_hash` for backfilled rows.
14. Migration verifies old/new row counts.
15. Deploy app.
16. Keep `audit_logs_old` for one release.
17. Drop `audit_logs_old` later after verification.

---

## Final Architecture

```text
Python / FastAPI
  |
  | insert_audit_log()
  | - creates id
  | - creates nonce_uuid with UUID5
  | - does not calculate row_hash
  v
PostgreSQL logical table: audit_logs
  |
  | GENERATED ALWAYS AS IDENTITY -> chain_seq
  | DEFAULT clock_timestamp() -> created_at
  | BEFORE INSERT trigger -> row_hash
  v
Monthly partitions by created_at
```

Final columns added to `audit_logs`:

```text
chain_seq
nonce_uuid
row_hash
```

No extra audit-chain tables.
