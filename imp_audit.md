# Implement Audit Log Hash Chaining With Monthly Partitioning

## Scope

This plan is intentionally limited to **only one logical table**:

- `audit_logs`

Do **not** create these tables:

- `audit_chain_checkpoints`
- `audit_chain_state`
- `audit_chain_secret`

Rejected items:

- append-only trigger path
- external checkpoint anchoring
- separate chain state table
- separate secret table
- separate checkpoint table

Accepted items:

- monthly range partitioning
- hash chaining
- use application `jwt_secret` from `settings.py`
- print latest audit hash on app startup
- smooth migration with no data loss

## Important PostgreSQL Partitioning Clarification

PostgreSQL monthly range partitioning always creates child partition tables internally.

Application code still queries only the logical parent table:

```sql
SELECT * FROM label_schema.audit_logs;
```

PostgreSQL automatically routes queries and inserts to the correct monthly partition.

So the app sees one table: `audit_logs`.

Internally PostgreSQL will create physical child tables such as:

```text
audit_logs_2026_05
audit_logs_2026_06
audit_logs_2026_07
```

If the hard requirement is literally one physical PostgreSQL table, then native monthly partitioning is not possible. The practical interpretation should be:

> one logical query table: `audit_logs`, with PostgreSQL-managed monthly partitions underneath.

## Final Design

`audit_logs` will contain both the original audit data and the hash-chain fields.

Recommended final columns:

```sql
id text NOT NULL,
chain_seq bigint NOT NULL,
event_type text NOT NULL,
entity_type text,
entity_id text,
project_id text,
user_id integer,
ip_address text,
created_at timestamptz NOT NULL,
details jsonb,
prev_hash text,
row_hash text NOT NULL,
hash_algo text NOT NULL DEFAULT 'HMAC-SHA256',
chain_version integer NOT NULL DEFAULT 1
```

Primary key for partitioned table:

```sql
PRIMARY KEY (id, created_at)
```

Why not `PRIMARY KEY (id)`?

PostgreSQL requires every unique constraint or primary key on a partitioned table to include the partition key. Since the partition key is `created_at`, the primary key must include `created_at`.

## Hash Formula

Use HMAC-SHA256 with `settings.jwt_secret`.

Formula:

```text
row_hash = HMAC_SHA256(jwt_secret, canonical_payload)
```

Where `canonical_payload` includes:

```text
chain_seq
id
event_type
entity_type
entity_id
project_id
user_id
ip_address
created_at
details
prev_hash
hash_algo
chain_version
```

Do not include `row_hash` inside the payload because that would be recursive.

## Three-Row Example

Assume the first hash starts from a genesis value:

```text
GENESIS = 0000000000000000000000000000000000000000000000000000000000000000
```

### Row 1

```text
chain_seq = 1
event_type = LOGIN_SUCCESS
entity_type = user
entity_id = 101
prev_hash = GENESIS
row_hash = HMAC_SHA256(jwt_secret, canonical_row_1)
```

Call this hash:

```text
A1
```

### Row 2

```text
chain_seq = 2
event_type = FILE_UPLOADED
entity_type = file
entity_id = file_555
prev_hash = A1
row_hash = HMAC_SHA256(jwt_secret, canonical_row_2)
```

Call this hash:

```text
B2
```

### Row 3

```text
chain_seq = 3
event_type = PROJECT_CREATED
entity_type = project
entity_id = project_9001
prev_hash = B2
row_hash = HMAC_SHA256(jwt_secret, canonical_row_3)
```

Call this hash:

```text
C3
```

If row 2 data changes, `B2` changes. Then row 3 still points to old `B2`, so chain verification fails.

If someone changes row 2 and row 3 hashes together and also knows `jwt_secret`, they can recompute the chain. With this accepted design, that risk remains because external checkpointing and separate secret storage are rejected.

## Security Claim Under This Scope

This design supports this statement:

> `audit_logs` is hash-chained and partitioned by month. Data changes can be detected by recomputing the chain, provided the original `jwt_secret` has not been compromised and the latest startup-printed hash is available for comparison.

This design should **not** claim absolute immutability, because:

- append-only triggers are rejected
- external checkpointing is rejected
- the HMAC secret is the app JWT secret
- anyone with DB write access and `jwt_secret` can rewrite rows and recompute hashes

## SQLAlchemy Model Change

Update `AuditLog` in `label/DAL/models/tables.py`.

Current model:

```python
class AuditLog(Base):
    __tablename__ = "audit_logs"

    id = Column(String, primary_key=True)
    event_type = Column(String, nullable=False)
    entity_type = Column(String)
    entity_id = Column(String)
    project_id = Column(String, ForeignKey("projects.id"))
    user_id = Column(Integer, ForeignKey("users.id"))
    ip_address = Column(String)
    created_at = Column(DateTime(timezone=True), server_default=func.now(), nullable=False)
    details = Column(JSONB)
```

Recommended model after migration:

```python
class AuditLog(Base):
    __tablename__ = "audit_logs"

    __table_args__ = {
        "postgresql_partition_by": "RANGE (created_at)",
        "extend_existing": True,
    }

    id = Column(String, primary_key=True)
    chain_seq = Column(Integer, nullable=False)
    event_type = Column(String, nullable=False)
    entity_type = Column(String)
    entity_id = Column(String)
    project_id = Column(String, ForeignKey("projects.id"))
    user_id = Column(Integer, ForeignKey("users.id"))
    ip_address = Column(String)
    created_at = Column(DateTime(timezone=True), primary_key=True, server_default=func.now(), nullable=False)
    details = Column(JSONB)
    prev_hash = Column(String)
    row_hash = Column(String, nullable=False)
    hash_algo = Column(String, nullable=False, server_default=text("'HMAC-SHA256'"))
    chain_version = Column(Integer, nullable=False, server_default=text("1"))
```

Note: SQLAlchemy `Integer` maps to regular integer. For `bigint`, import `BigInteger` and use it for `chain_seq` if very large audit volume is expected.

Better version with `BigInteger`:

```python
from sqlalchemy import BigInteger

class AuditLog(Base):
    __tablename__ = "audit_logs"

    __table_args__ = {
        "postgresql_partition_by": "RANGE (created_at)",
        "extend_existing": True,
    }

    id = Column(String, primary_key=True)
    chain_seq = Column(BigInteger, nullable=False)
    event_type = Column(String, nullable=False)
    entity_type = Column(String)
    entity_id = Column(String)
    project_id = Column(String, ForeignKey("projects.id"))
    user_id = Column(Integer, ForeignKey("users.id"))
    ip_address = Column(String)
    created_at = Column(DateTime(timezone=True), primary_key=True, server_default=func.now(), nullable=False)
    details = Column(JSONB)
    prev_hash = Column(String)
    row_hash = Column(String, nullable=False)
    hash_algo = Column(String, nullable=False, server_default=text("'HMAC-SHA256'"))
    chain_version = Column(Integer, nullable=False, server_default=text("1"))
```

## Application Hash Utility

Create a small utility, for example:

```text
label/utils/audit_hash.py
```

Code:

```python
from __future__ import annotations

import hashlib
import hmac
import json
from datetime import datetime, timezone
from typing import Any


GENESIS_HASH = "0" * 64
HASH_ALGO = "HMAC-SHA256"
CHAIN_VERSION = 1


def _normalize_datetime(value: Any) -> Any:
    if isinstance(value, datetime):
        if value.tzinfo is None:
            value = value.replace(tzinfo=timezone.utc)
        return value.astimezone(timezone.utc).isoformat().replace("+00:00", "Z")
    return value


def canonical_audit_payload(*, row: dict[str, Any], prev_hash: str) -> str:
    payload = {
        "chain_seq": row.get("chain_seq"),
        "id": row.get("id"),
        "event_type": row.get("event_type"),
        "entity_type": row.get("entity_type"),
        "entity_id": row.get("entity_id"),
        "project_id": row.get("project_id"),
        "user_id": row.get("user_id"),
        "ip_address": row.get("ip_address"),
        "created_at": _normalize_datetime(row.get("created_at")),
        "details": row.get("details") or {},
        "prev_hash": prev_hash,
        "hash_algo": HASH_ALGO,
        "chain_version": CHAIN_VERSION,
    }
    return json.dumps(payload, sort_keys=True, separators=(",", ":"), ensure_ascii=False)


def compute_audit_hash(*, jwt_secret: str, row: dict[str, Any], prev_hash: str) -> str:
    if not jwt_secret:
        raise ValueError("JWT_SECRET is required to compute audit row hashes")

    canonical_payload = canonical_audit_payload(row=row, prev_hash=prev_hash)
    return hmac.new(
        jwt_secret.encode("utf-8"),
        canonical_payload.encode("utf-8"),
        hashlib.sha256,
    ).hexdigest()
```

## Insert Flow

When inserting a new audit row:

1. lock the latest audit row
2. get last `chain_seq` and `row_hash`
3. set `chain_seq = last_chain_seq + 1`
4. set `prev_hash = last_row_hash` or genesis hash
5. compute `row_hash` using `settings.jwt_secret`
6. insert the row
7. commit transaction

Sample SQLAlchemy helper:

```python
from __future__ import annotations

from datetime import datetime, timezone
from typing import Any
from uuid import uuid4

from sqlalchemy import text
from sqlalchemy.orm import Session

from DAL.models.tables import AuditLog
from settings import get_settings
from utils.audit_hash import GENESIS_HASH, HASH_ALGO, CHAIN_VERSION, compute_audit_hash


def create_audit_log(
    db: Session,
    *,
    event_type: str,
    entity_type: str | None = None,
    entity_id: str | None = None,
    project_id: str | None = None,
    user_id: int | None = None,
    ip_address: str | None = None,
    details: dict[str, Any] | None = None,
) -> AuditLog:
    settings = get_settings()
    created_at = datetime.now(timezone.utc)

    last_row = db.execute(
        text("""
        SELECT chain_seq, row_hash
        FROM audit_logs
        ORDER BY chain_seq DESC
        LIMIT 1
        FOR UPDATE
        """)
    ).mappings().first()

    last_chain_seq = int(last_row["chain_seq"]) if last_row else 0
    prev_hash = str(last_row["row_hash"]) if last_row else GENESIS_HASH

    row_data = {
        "id": str(uuid4()),
        "chain_seq": last_chain_seq + 1,
        "event_type": event_type,
        "entity_type": entity_type,
        "entity_id": entity_id,
        "project_id": project_id,
        "user_id": user_id,
        "ip_address": ip_address,
        "created_at": created_at,
        "details": details or {},
    }

    row_hash = compute_audit_hash(
        jwt_secret=settings.jwt_secret,
        row=row_data,
        prev_hash=prev_hash,
    )

    audit_log = AuditLog(
        **row_data,
        prev_hash=prev_hash,
        row_hash=row_hash,
        hash_algo=HASH_ALGO,
        chain_version=CHAIN_VERSION,
    )
    db.add(audit_log)
    return audit_log
```

Important: call this inside the same transaction as the business operation when possible.

Example:

```python
project = Project(...)
db.add(project)

create_audit_log(
    db,
    event_type="PROJECT_CREATED",
    entity_type="project",
    entity_id=project.id,
    project_id=project.id,
    user_id=current_user.id,
    ip_address=request.client.host if request.client else None,
    details={"project_name": project.project_name},
)

db.commit()
```

## Startup Latest Hash Logging

On app startup, print the latest hash.

In this project, `create_app()` in `label/main.py` already calls `init_db()` during startup. The cleanest wiring is:

1. keep `init_db()` as-is for migrations and admin seeding
2. after `init_db()`, open a short DB session
3. query latest `audit_logs` hash
4. log it once

The startup log should not fail app startup if `audit_logs` does not exist yet during first bootstrap. Log a warning and continue.

Example function:

```python
import logging

from sqlalchemy import text
from sqlalchemy.orm import Session

logger = logging.getLogger(__name__)


def log_latest_audit_hash(db: Session) -> None:
    row = db.execute(
        text("""
        SELECT chain_seq, row_hash, created_at
        FROM audit_logs
        ORDER BY chain_seq DESC
        LIMIT 1
        """)
    ).mappings().first()

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

Example FastAPI startup usage:

```python
from contextlib import asynccontextmanager

from fastapi import FastAPI

from DAL.queries.database import SessionLocal
from utils.audit_startup import log_latest_audit_hash


@asynccontextmanager
async def lifespan(app: FastAPI):
    db = SessionLocal()
    try:
        log_latest_audit_hash(db)
    finally:
        db.close()

    yield


app = FastAPI(lifespan=lifespan)
```

## Monthly Partition Creation SQL

PostgreSQL parent table:

```sql
CREATE TABLE label_schema.audit_logs (
    id text NOT NULL,
    chain_seq bigint NOT NULL,
    event_type text NOT NULL,
    entity_type text,
    entity_id text,
    project_id text,
    user_id integer,
    ip_address text,
    created_at timestamptz NOT NULL DEFAULT now(),
    details jsonb,
    prev_hash text,
    row_hash text NOT NULL,
    hash_algo text NOT NULL DEFAULT 'HMAC-SHA256',
    chain_version integer NOT NULL DEFAULT 1,
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);
```

Example monthly partitions:

```sql
CREATE TABLE IF NOT EXISTS label_schema.audit_logs_2026_05
PARTITION OF label_schema.audit_logs
FOR VALUES FROM ('2026-05-01 00:00:00+00') TO ('2026-06-01 00:00:00+00');

CREATE TABLE IF NOT EXISTS label_schema.audit_logs_2026_06
PARTITION OF label_schema.audit_logs
FOR VALUES FROM ('2026-06-01 00:00:00+00') TO ('2026-07-01 00:00:00+00');
```

Partition helper function:

```sql
CREATE OR REPLACE FUNCTION label_schema.ensure_audit_log_month_partition(month_start date)
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
    partition_fqn := format('label_schema.%I', partition_name);

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
        'CREATE INDEX IF NOT EXISTS %I ON %s (event_type, created_at)',
        partition_name || '_event_created_idx',
        partition_fqn
    );
END;
$$;
```

Create current and next 12 months:

```sql
SELECT label_schema.ensure_audit_log_month_partition(
    (date_trunc('month', now()) + (n || ' month')::interval)::date
)
FROM generate_series(0, 12) AS n;
```

## Smooth Migration Strategy With No Data Loss

Because existing `audit_logs` is not partitioned, the safest path is:

1. rename old table to `audit_logs_old`
2. create new partitioned parent table named `audit_logs`
3. create partitions covering old data and future months
4. backfill old rows into the new partitioned table
5. compute `chain_seq`, `prev_hash`, and `row_hash` during backfill
6. verify row counts
7. keep `audit_logs_old` temporarily for rollback
8. drop `audit_logs_old` only after production verification

### Backfill Problem

The migration needs `jwt_secret` to compute `row_hash`.

Alembic can read environment variables. So the migration can use:

```python
import os

jwt_secret = os.environ.get("JWT_SECRET", "").strip()
if not jwt_secret:
    raise RuntimeError("JWT_SECRET is required to backfill audit log hashes")
```

Do not hardcode `jwt_secret` in the migration file.

## Alembic Migration Skeleton

Create a new migration file under:

```text
label/alembic/versions/
```

Example skeleton:

```python
from __future__ import annotations

import hashlib
import hmac
import json
import os
from datetime import date, datetime, timezone
from typing import Any

from alembic import op
import sqlalchemy as sa


GENESIS_HASH = "0" * 64
HASH_ALGO = "HMAC-SHA256"
CHAIN_VERSION = 1
try:
    from settings import resolve_schema_name
    SCHEMA = resolve_schema_name()
except Exception:
    SCHEMA = "label_schema"


def _normalize_datetime(value: Any) -> Any:
    if isinstance(value, datetime):
        if value.tzinfo is None:
            value = value.replace(tzinfo=timezone.utc)
        return value.astimezone(timezone.utc).isoformat().replace("+00:00", "Z")
    return value


def _canonical_payload(row: dict[str, Any], prev_hash: str) -> str:
    payload = {
        "chain_seq": row.get("chain_seq"),
        "id": row.get("id"),
        "event_type": row.get("event_type"),
        "entity_type": row.get("entity_type"),
        "entity_id": row.get("entity_id"),
        "project_id": row.get("project_id"),
        "user_id": row.get("user_id"),
        "ip_address": row.get("ip_address"),
        "created_at": _normalize_datetime(row.get("created_at")),
        "details": row.get("details") or {},
        "prev_hash": prev_hash,
        "hash_algo": HASH_ALGO,
        "chain_version": CHAIN_VERSION,
    }
    return json.dumps(payload, sort_keys=True, separators=(",", ":"), ensure_ascii=False)


def _compute_hash(jwt_secret: str, row: dict[str, Any], prev_hash: str) -> str:
    payload = _canonical_payload(row, prev_hash)
    return hmac.new(jwt_secret.encode("utf-8"), payload.encode("utf-8"), hashlib.sha256).hexdigest()


def upgrade():
    jwt_secret = os.environ.get("JWT_SECRET", "").strip()
    if not jwt_secret:
        raise RuntimeError("JWT_SECRET is required to backfill audit log hashes")

    bind = op.get_bind()

    # 1. Rename existing table.
    op.execute(f"ALTER TABLE {SCHEMA}.audit_logs RENAME TO audit_logs_old")

    # Important: renaming a table does not automatically rename every old
    # constraint/index. Rename the old primary key if it exists, otherwise the
    # new partitioned table may fail to create audit_logs_pkey.
    op.execute(f"""
    DO $$
    BEGIN
        IF EXISTS (
            SELECT 1
            FROM pg_constraint c
            JOIN pg_namespace n ON n.oid = c.connamespace
            WHERE n.nspname = '{SCHEMA}'
              AND c.conname = 'audit_logs_pkey'
        ) THEN
            ALTER TABLE {SCHEMA}.audit_logs_old
            RENAME CONSTRAINT audit_logs_pkey TO audit_logs_old_pkey;
        END IF;
    END $$;
    """)

    # 2. Create new partitioned parent table.
    op.execute(f"""
    CREATE TABLE {SCHEMA}.audit_logs (
        id text NOT NULL,
        chain_seq bigint NOT NULL,
        event_type text NOT NULL,
        entity_type text,
        entity_id text,
        project_id text,
        user_id integer,
        ip_address text,
        created_at timestamptz NOT NULL DEFAULT now(),
        details jsonb,
        prev_hash text,
        row_hash text NOT NULL,
        hash_algo text NOT NULL DEFAULT 'HMAC-SHA256',
        chain_version integer NOT NULL DEFAULT 1,
        PRIMARY KEY (id, created_at)
    ) PARTITION BY RANGE (created_at)
    """)

    # 3. Create helper function for monthly partitions.
    op.execute(f"""
    CREATE OR REPLACE FUNCTION {SCHEMA}.ensure_audit_log_month_partition(month_start date)
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
        partition_fqn := format('{SCHEMA}.%I', partition_name);

        EXECUTE format(
            'CREATE TABLE IF NOT EXISTS %s PARTITION OF {SCHEMA}.audit_logs FOR VALUES FROM (%L) TO (%L)',
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
            'CREATE INDEX IF NOT EXISTS %I ON %s (event_type, created_at)',
            partition_name || '_event_created_idx',
            partition_fqn
        );
    END;
    $$;
    """)

    # 4. Create partitions for all existing data months.
    months = bind.execute(sa.text(f"""
        SELECT DISTINCT date_trunc('month', created_at)::date AS month_start
        FROM {SCHEMA}.audit_logs_old
        ORDER BY month_start
    """)).scalars().all()

    for month_start in months:
        bind.execute(
            sa.text(f"SELECT {SCHEMA}.ensure_audit_log_month_partition(:month_start)"),
            {"month_start": month_start},
        )

    # 5. Create current and next 12 months.
    op.execute(f"""
    SELECT {SCHEMA}.ensure_audit_log_month_partition(
        (date_trunc('month', now()) + (n || ' month')::interval)::date
    )
    FROM generate_series(0, 12) AS n
    """)

    # 6. Read old rows in deterministic order and backfill hashes.
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
        FROM {SCHEMA}.audit_logs_old
        ORDER BY created_at ASC, id ASC
    """)).mappings().all()

    prev_hash = GENESIS_HASH
    chain_seq = 0

    for old_row in old_rows:
        chain_seq += 1
        row = dict(old_row)
        row["chain_seq"] = chain_seq

        row_hash = _compute_hash(jwt_secret, row, prev_hash)

        bind.execute(sa.text(f"""
            INSERT INTO {SCHEMA}.audit_logs (
                id,
                chain_seq,
                event_type,
                entity_type,
                entity_id,
                project_id,
                user_id,
                ip_address,
                created_at,
                details,
                prev_hash,
                row_hash,
                hash_algo,
                chain_version
            ) VALUES (
                :id,
                :chain_seq,
                :event_type,
                :entity_type,
                :entity_id,
                :project_id,
                :user_id,
                :ip_address,
                :created_at,
                :details,
                :prev_hash,
                :row_hash,
                :hash_algo,
                :chain_version
            )
        """), {
            **row,
            "prev_hash": prev_hash,
            "row_hash": row_hash,
            "hash_algo": HASH_ALGO,
            "chain_version": CHAIN_VERSION,
        })

        prev_hash = row_hash

    # 7. Verify row counts.
    old_count = bind.execute(sa.text(f"SELECT count(*) FROM {SCHEMA}.audit_logs_old")).scalar_one()
    new_count = bind.execute(sa.text(f"SELECT count(*) FROM {SCHEMA}.audit_logs")).scalar_one()
    if old_count != new_count:
        raise RuntimeError(f"audit_logs migration count mismatch: old={old_count}, new={new_count}")

    # 8. Keep audit_logs_old temporarily for rollback. Drop in a later migration after verification.


def downgrade():
    bind = op.get_bind()

    # Downgrade restores the old table only if it still exists.
    op.execute(f"DROP TABLE IF EXISTS {SCHEMA}.audit_logs CASCADE")
    op.execute(f"ALTER TABLE {SCHEMA}.audit_logs_old RENAME TO audit_logs")
```

## Important Migration Notes

### Schema Name

This repository supports a dynamic schema through `SCHEMA_NAME` and `resolve_schema_name()` in `label/settings.py`.

Do not hardcode `label_schema` in the final migration unless the deployment always uses that schema. Prefer:

```python
try:
    from settings import resolve_schema_name
    SCHEMA = resolve_schema_name()
except Exception:
    SCHEMA = "label_schema"
```

### Constraint and Index Name Collisions

When PostgreSQL renames `audit_logs` to `audit_logs_old`, existing constraints and indexes may still have names like:

```text
audit_logs_pkey
audit_logs_project_id_fkey
audit_logs_user_id_fkey
```

The new table may fail to create constraints with the same names. Rename old constraints after the table rename:

```sql
ALTER TABLE label_schema.audit_logs_old
RENAME CONSTRAINT audit_logs_pkey TO audit_logs_old_pkey;
```

Do the same for old FK constraints if needed.

### Existing Foreign Keys

The current `audit_logs` table has foreign keys:

```sql
project_id REFERENCES projects(id)
user_id REFERENCES users(id)
```

The migration skeleton above omits FK constraints for simplicity. Production migration should recreate them:

```sql
ALTER TABLE label_schema.audit_logs
ADD CONSTRAINT audit_logs_project_id_fkey
FOREIGN KEY (project_id)
REFERENCES label_schema.projects(id);

ALTER TABLE label_schema.audit_logs
ADD CONSTRAINT audit_logs_user_id_fkey
FOREIGN KEY (user_id)
REFERENCES label_schema.users(id);
```

### Existing Indexes

Add indexes for expected query patterns:

```sql
CREATE INDEX IF NOT EXISTS audit_logs_chain_seq_idx
ON label_schema.audit_logs (chain_seq);

CREATE INDEX IF NOT EXISTS audit_logs_project_created_idx
ON label_schema.audit_logs (project_id, created_at);

CREATE INDEX IF NOT EXISTS audit_logs_entity_idx
ON label_schema.audit_logs (entity_type, entity_id, created_at);

CREATE INDEX IF NOT EXISTS audit_logs_event_created_idx
ON label_schema.audit_logs (event_type, created_at);
```

On partitioned tables, indexes created on the parent become partitioned indexes and are applied to child partitions.

## Verification Query

Check continuity:

```sql
WITH ordered AS (
    SELECT
        chain_seq,
        id,
        prev_hash,
        row_hash,
        lag(row_hash) OVER (ORDER BY chain_seq) AS expected_prev_hash
    FROM label_schema.audit_logs
)
SELECT *
FROM ordered
WHERE chain_seq > 1
  AND prev_hash IS DISTINCT FROM expected_prev_hash;
```

Expected result:

```text
0 rows
```

Get latest hash:

```sql
SELECT chain_seq, row_hash, created_at
FROM label_schema.audit_logs
ORDER BY chain_seq DESC
LIMIT 1;
```

Check monthly partitions:

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
WHERE parent.relname = 'audit_logs'
ORDER BY child.relname;
```

## Chain Verification Script

Create a script if needed:

```text
scripts/verify_audit_chain.py
```

Code:

```python
from __future__ import annotations

from sqlalchemy import create_engine, text
from sqlalchemy.orm import Session

from settings import get_settings
from utils.audit_hash import GENESIS_HASH, compute_audit_hash


def main() -> None:
    settings = get_settings()
    engine = create_engine(settings.database_url)

    with Session(engine) as db:
        rows = db.execute(text("""
            SELECT
                id,
                chain_seq,
                event_type,
                entity_type,
                entity_id,
                project_id,
                user_id,
                ip_address,
                created_at,
                details,
                prev_hash,
                row_hash
            FROM audit_logs
            ORDER BY chain_seq ASC
        """)).mappings().all()

        prev_hash = GENESIS_HASH
        for row in rows:
            row_dict = dict(row)
            stored_prev_hash = row_dict.pop("prev_hash")
            stored_row_hash = row_dict.pop("row_hash")

            if stored_prev_hash != prev_hash:
                raise RuntimeError(
                    f"Broken prev_hash at chain_seq={row['chain_seq']}: "
                    f"expected={prev_hash} actual={stored_prev_hash}"
                )

            computed_hash = compute_audit_hash(
                jwt_secret=settings.jwt_secret,
                row=row_dict,
                prev_hash=prev_hash,
            )

            if computed_hash != stored_row_hash:
                raise RuntimeError(
                    f"Broken row_hash at chain_seq={row['chain_seq']}: "
                    f"expected={computed_hash} actual={stored_row_hash}"
                )

            prev_hash = stored_row_hash

    print(f"Audit chain verified successfully. rows={len(rows)} latest_hash={prev_hash}")


if __name__ == "__main__":
    main()
```

## New Insert Race Condition Warning

The app-level insert flow uses:

```sql
SELECT chain_seq, row_hash
FROM audit_logs
ORDER BY chain_seq DESC
LIMIT 1
FOR UPDATE
```

This locks the latest row, but with an empty table or high concurrency, two transactions can race.

Safer options:

1. Use PostgreSQL advisory transaction lock around audit insert.
2. Use serializable transaction isolation for audit insert.
3. Add a unique index on `chain_seq` and retry on conflict.

Recommended advisory lock:

```python
db.execute(text("SELECT pg_advisory_xact_lock(hashtext('audit_logs_hash_chain'))"))
```

Use it before selecting latest hash:

```python
db.execute(text("SELECT pg_advisory_xact_lock(hashtext('audit_logs_hash_chain'))"))

last_row = db.execute(
    text("""
    SELECT chain_seq, row_hash
    FROM audit_logs
    ORDER BY chain_seq DESC
    LIMIT 1
    """)
).mappings().first()
```

This keeps the one-table design and avoids a separate chain state table.

## Updated Insert Helper With Advisory Lock

Use this version in production:

```python
def create_audit_log(
    db: Session,
    *,
    event_type: str,
    entity_type: str | None = None,
    entity_id: str | None = None,
    project_id: str | None = None,
    user_id: int | None = None,
    ip_address: str | None = None,
    details: dict[str, Any] | None = None,
) -> AuditLog:
    settings = get_settings()
    created_at = datetime.now(timezone.utc)

    db.execute(text("SELECT pg_advisory_xact_lock(hashtext('audit_logs_hash_chain'))"))

    last_row = db.execute(
        text("""
        SELECT chain_seq, row_hash
        FROM audit_logs
        ORDER BY chain_seq DESC
        LIMIT 1
        """)
    ).mappings().first()

    last_chain_seq = int(last_row["chain_seq"]) if last_row else 0
    prev_hash = str(last_row["row_hash"]) if last_row else GENESIS_HASH

    row_data = {
        "id": str(uuid4()),
        "chain_seq": last_chain_seq + 1,
        "event_type": event_type,
        "entity_type": entity_type,
        "entity_id": entity_id,
        "project_id": project_id,
        "user_id": user_id,
        "ip_address": ip_address,
        "created_at": created_at,
        "details": details or {},
    }

    row_hash = compute_audit_hash(
        jwt_secret=settings.jwt_secret,
        row=row_data,
        prev_hash=prev_hash,
    )

    audit_log = AuditLog(
        **row_data,
        prev_hash=prev_hash,
        row_hash=row_hash,
        hash_algo=HASH_ALGO,
        chain_version=CHAIN_VERSION,
    )
    db.add(audit_log)
    return audit_log
```

## Rollout Plan

1. Add `utils/audit_hash.py`.
2. Add audit insert helper with advisory lock.
3. Add startup latest-hash logging.
4. Create Alembic migration.
5. In migration, rename `audit_logs` to `audit_logs_old`.
6. Create new partitioned `audit_logs` parent.
7. Create partitions covering existing data.
8. Create current and next 12 monthly partitions.
9. Backfill all old rows with `chain_seq`, `prev_hash`, and `row_hash`.
10. Verify old and new row counts match.
11. Deploy app code that writes new hash fields.
12. Keep `audit_logs_old` for one release.
13. After verification, drop `audit_logs_old` in a later migration.

## Final Accepted Architecture

```text
FastAPI app
  |
  | uses settings.jwt_secret
  | computes HMAC-SHA256 row hash
  | logs latest hash at startup
  v
PostgreSQL logical table: audit_logs
  |
  | partitioned by created_at month
  v
Physical monthly partitions managed by PostgreSQL
```

No separate audit chain tables are used.
