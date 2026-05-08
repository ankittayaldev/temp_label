# Audit Log Immutability Report

## Executive Summary

The current `audit_logs` table is a normal mutable PostgreSQL table. To present it to an external auditor as "immutable", we should be precise:

> The target design is append-only, partitioned, cryptographically tamper-evident audit logging.

Inside one PostgreSQL database, no design is perfectly immutable against a superuser or someone who can run arbitrary SQL as the database owner. But we can make unauthorized changes hard, noisy, and detectable.

The strongest design available with Alembic `op.execute()` is:

1. Native PostgreSQL monthly range partitions.
2. Database triggers that reject `UPDATE`, `DELETE`, and `TRUNCATE`.
3. Database-generated hash chaining using `pgcrypto`.
4. A private HMAC secret stored outside the `audit_logs` table, ideally not readable by the app role.
5. Periodic chain checkpoints printed to application logs and stored in a checkpoint table.
6. Optional manual/offline checkpoint evidence for auditor-grade proof.

Hash chaining is still a good option, but deterministic UUIDs, timestamps, and random values do not solve the privileged-attacker problem by themselves.

## Important Reality Check

If someone can do all of the following, then pure database-only immutability is impossible:

- update old audit rows
- update stored hashes
- update later rows in the chain
- update or delete checkpoint rows
- modify trigger/function definitions
- access any hash or HMAC secret

No UUID version, timestamp, random value, or hash formula can fully fix that, because the attacker can recompute the new values after changing history.

What we can do is reduce who can perform those actions and create evidence that tampering occurred.

## UUID5, Timestamp, Random: What They Actually Provide

### UUID5

UUID5 is deterministic. It produces the same UUID for the same namespace and input.

Example:

```text
uuid5(namespace, "LOGIN_SUCCESS:user:101:2026-05-08T10:00:00Z")
```

This is useful for idempotency, but it is not useful as tamper protection. If an attacker changes the row, they can recompute the UUID5 value.

### Current Timestamp

A timestamp gives ordering evidence, but it is not enough. If the column is writable, it can be changed. Even if it is database-generated, a privileged attacker can still rewrite the row later.

Use database timestamps anyway, but treat them as part of the hash payload, not as the main security control.

### Random / Nonce

A random nonce prevents two identical events from having the same hash payload. That is useful.

But random values stored in the same row do not stop a rewrite. If an attacker changes both payload and nonce, they can recompute the hash.

### Stronger Pattern

Use all of these as ingredients, but do not rely on them alone:

- database-generated monotonic sequence number
- database-generated timestamp
- random nonce from `gen_random_bytes()`
- previous row hash
- canonical row payload
- HMAC secret not stored in the audit row

The HMAC secret is the important upgrade. If the attacker only has table write access but cannot read the secret, they cannot recompute valid hashes.

## Recommended Audit Table Shape

For PostgreSQL monthly partitioning, a unique or primary key on a partitioned table must include the partition key. Because we partition by `created_at`, the primary key should be either:

- composite: `(id, created_at)`, or
- no parent primary key, with local partition indexes.

Recommended columns:

```sql
CREATE TABLE audit_logs (
    id text NOT NULL,
    chain_seq bigint GENERATED ALWAYS AS IDENTITY,
    event_type text NOT NULL,
    entity_type text,
    entity_id text,
    project_id text,
    user_id integer,
    ip_address text,
    created_at timestamptz NOT NULL DEFAULT clock_timestamp(),
    details jsonb,
    nonce bytea NOT NULL DEFAULT gen_random_bytes(16),
    prev_hash text,
    row_hash text NOT NULL,
    hash_algo text NOT NULL DEFAULT 'HMAC-SHA256',
    chain_version integer NOT NULL DEFAULT 1,
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);
```

Note: generated identity on a partitioned table is supported in modern PostgreSQL versions. If your version has issues, use an explicit sequence and `DEFAULT nextval(...)`.

## 1) Append-Only Write Path

You are right to be skeptical. Application-level append-only rules are not enough.

The database should reject mutation directly.

### Block UPDATE and DELETE

```sql
CREATE OR REPLACE FUNCTION audit_logs_block_mutation()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
    RAISE EXCEPTION 'audit_logs is append-only: % is not allowed', TG_OP;
END;
$$;

CREATE TRIGGER trg_audit_logs_no_update
BEFORE UPDATE ON audit_logs
FOR EACH ROW
EXECUTE FUNCTION audit_logs_block_mutation();

CREATE TRIGGER trg_audit_logs_no_delete
BEFORE DELETE ON audit_logs
FOR EACH ROW
EXECUTE FUNCTION audit_logs_block_mutation();
```

### Block TRUNCATE

```sql
CREATE OR REPLACE FUNCTION audit_logs_block_truncate()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
    RAISE EXCEPTION 'audit_logs is append-only: TRUNCATE is not allowed';
END;
$$;

CREATE TRIGGER trg_audit_logs_no_truncate
BEFORE TRUNCATE ON audit_logs
FOR EACH STATEMENT
EXECUTE FUNCTION audit_logs_block_truncate();
```

### Revoke Table Mutation Privileges

If you know the app database role, revoke mutation and grant insert/select only:

```sql
REVOKE UPDATE, DELETE, TRUNCATE ON audit_logs FROM label_app_user;
GRANT INSERT, SELECT ON audit_logs TO label_app_user;
```

This can also be run from Alembic with `op.execute()`, but only if the migration user owns the table or has grant permissions.

### Alembic Example

```python
def upgrade():
    op.execute("""
    CREATE OR REPLACE FUNCTION audit_logs_block_mutation()
    RETURNS trigger
    LANGUAGE plpgsql
    AS $$
    BEGIN
        RAISE EXCEPTION 'audit_logs is append-only: % is not allowed', TG_OP;
    END;
    $$;
    """)

    op.execute("""
    CREATE TRIGGER trg_audit_logs_no_update
    BEFORE UPDATE ON audit_logs
    FOR EACH ROW
    EXECUTE FUNCTION audit_logs_block_mutation();
    """)

    op.execute("""
    CREATE TRIGGER trg_audit_logs_no_delete
    BEFORE DELETE ON audit_logs
    FOR EACH ROW
    EXECUTE FUNCTION audit_logs_block_mutation();
    """)
```

### Limitation

The table owner or superuser can drop the triggers. So this protects against application bugs, normal app-role compromise, and accidental admin operations, but not against full DBA-level compromise.

## 2) Monthly Range Partitioning

PostgreSQL has native declarative partitioning. FastAPI does not need to own this logic.

The database table can be declared as:

```sql
CREATE TABLE audit_logs (
    ...
) PARTITION BY RANGE (created_at);
```

Then each month is a child table:

```sql
CREATE TABLE audit_logs_2026_05
PARTITION OF audit_logs
FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE TABLE audit_logs_2026_06
PARTITION OF audit_logs
FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');
```

### Migrating Existing Non-Partitioned Table

PostgreSQL cannot simply convert every existing table into a partitioned table with one small `ALTER TABLE`. A practical migration is:

```sql
ALTER TABLE audit_logs RENAME TO audit_logs_old;

CREATE TABLE audit_logs (
    id text NOT NULL,
    chain_seq bigint GENERATED ALWAYS AS IDENTITY,
    event_type text NOT NULL,
    entity_type text,
    entity_id text,
    project_id text,
    user_id integer,
    ip_address text,
    created_at timestamptz NOT NULL DEFAULT clock_timestamp(),
    details jsonb,
    nonce bytea NOT NULL DEFAULT gen_random_bytes(16),
    prev_hash text,
    row_hash text NOT NULL,
    hash_algo text NOT NULL DEFAULT 'HMAC-SHA256',
    chain_version integer NOT NULL DEFAULT 1,
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

CREATE TABLE audit_logs_2026_05
PARTITION OF audit_logs
FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
```

Then migrate old rows after backfilling `prev_hash` and `row_hash`.

### Creating Future Monthly Partitions With SQL Function

PostgreSQL does not include a built-in scheduler like cron in core. There is an extension called `pg_cron`, but if DevOps changes are not available, do not depend on it.

Use a database function and call it from:

- Alembic migration for the next N months
- FastAPI startup
- a low-traffic API path such as create-project
- a manual admin endpoint

Function:

```sql
CREATE OR REPLACE FUNCTION ensure_audit_log_month_partition(month_start date)
RETURNS void
LANGUAGE plpgsql
AS $$
DECLARE
    partition_name text;
    from_ts timestamptz;
    to_ts timestamptz;
BEGIN
    from_ts := date_trunc('month', month_start)::timestamptz;
    to_ts := (from_ts + interval '1 month')::timestamptz;
    partition_name := format('audit_logs_%s', to_char(from_ts, 'YYYY_MM'));

    EXECUTE format(
        'CREATE TABLE IF NOT EXISTS %I PARTITION OF audit_logs FOR VALUES FROM (%L) TO (%L)',
        partition_name,
        from_ts,
        to_ts
    );

    EXECUTE format(
        'CREATE INDEX IF NOT EXISTS %I ON %I (chain_seq)',
        partition_name || '_chain_seq_idx',
        partition_name
    );

    EXECUTE format(
        'CREATE INDEX IF NOT EXISTS %I ON %I (project_id, created_at)',
        partition_name || '_project_created_idx',
        partition_name
    );
END;
$$;
```

Create the current and next 12 months from Alembic:

```sql
SELECT ensure_audit_log_month_partition((date_trunc('month', now()) + (n || ' month')::interval)::date)
FROM generate_series(0, 12) AS n;
```

### Closing Old Partitions

For an old partition, block mutation at the partition level too:

```sql
REVOKE INSERT, UPDATE, DELETE, TRUNCATE ON audit_logs_2026_05 FROM label_app_user;
GRANT SELECT ON audit_logs_2026_05 TO label_app_user;
```

You can also add the same no-update/no-delete triggers to each partition if needed.

## 3) Hash Chaining: Strongest Practical Design

The most robust design under your constraints is not UUID5. It is:

- chain sequence generated by DB
- row timestamp generated by DB
- random nonce generated by DB
- canonical JSON payload
- previous hash
- HMAC secret outside the audit row

### Why HMAC Is Better Than Plain Hash

Plain hash:

```text
row_hash = sha256(row_payload + prev_hash)
```

If an attacker knows the payload and previous hash, they can recompute it.

HMAC:

```text
row_hash = hmac(secret_key, row_payload + prev_hash)
```

If the attacker cannot read the secret key, they cannot recompute valid hashes even if they can see every row.

### Secret Storage Table

Best practical database-only version:

```sql
CREATE TABLE audit_chain_secret (
    id integer PRIMARY KEY CHECK (id = 1),
    secret bytea NOT NULL,
    created_at timestamptz NOT NULL DEFAULT now()
);

INSERT INTO audit_chain_secret (id, secret)
VALUES (1, gen_random_bytes(32));

REVOKE ALL ON audit_chain_secret FROM PUBLIC;
```

Important: the application role should not be able to `SELECT` this table. The trigger function can be `SECURITY DEFINER` so it can use the secret internally.

### Chain State Table

```sql
CREATE TABLE audit_chain_state (
    id integer PRIMARY KEY CHECK (id = 1),
    last_chain_seq bigint,
    last_hash text,
    updated_at timestamptz NOT NULL DEFAULT now()
);

INSERT INTO audit_chain_state (id, last_chain_seq, last_hash)
VALUES (1, 0, repeat('0', 64));
```

### Hash Trigger

This trigger computes the chain in the database before insert.

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE OR REPLACE FUNCTION audit_logs_hash_before_insert()
RETURNS trigger
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
DECLARE
    previous_hash text;
    canonical_payload text;
    secret_key bytea;
BEGIN
    SELECT last_hash
    INTO previous_hash
    FROM audit_chain_state
    WHERE id = 1
    FOR UPDATE;

    SELECT secret
    INTO secret_key
    FROM audit_chain_secret
    WHERE id = 1;

    NEW.prev_hash := previous_hash;
    NEW.hash_algo := 'HMAC-SHA256';
    NEW.chain_version := 1;

    canonical_payload := jsonb_build_object(
        'id', NEW.id,
        'chain_seq', NEW.chain_seq,
        'event_type', NEW.event_type,
        'entity_type', NEW.entity_type,
        'entity_id', NEW.entity_id,
        'project_id', NEW.project_id,
        'user_id', NEW.user_id,
        'ip_address', NEW.ip_address,
        'created_at', NEW.created_at,
        'details', COALESCE(NEW.details, '{}'::jsonb),
        'nonce', encode(NEW.nonce, 'hex'),
        'prev_hash', NEW.prev_hash
    )::text;

    NEW.row_hash := encode(hmac(canonical_payload::bytea, secret_key, 'sha256'), 'hex');

    UPDATE audit_chain_state
    SET last_chain_seq = NEW.chain_seq,
        last_hash = NEW.row_hash,
        updated_at = now()
    WHERE id = 1;

    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_audit_logs_hash_before_insert
BEFORE INSERT ON audit_logs
FOR EACH ROW
EXECUTE FUNCTION audit_logs_hash_before_insert();
```

### Why This Is Better

This prevents normal application code from supplying fake hashes. The database computes them.

It also prevents a database user without access to `audit_chain_secret` from recomputing valid hashes.

### Still Not Fool Proof

This is strong, but not magic.

If an attacker can become table owner/superuser, read the secret, alter the function, disable triggers, or update `audit_chain_state`, they can still rewrite history.

So the strongest honest claim is:

> Hash chaining with an HMAC secret makes audit logs tamper-evident and resistant to normal table-level tampering. It does not defeat a full database-owner compromise without external evidence.

## Three-Row Example

Assume initial state:

```text
last_hash = 000000...000
secret = hidden
```

### Row 1

```text
event_type = LOGIN_SUCCESS
entity_type = user
entity_id = 101
prev_hash = 000000...000
nonce = random_1
row_hash = HMAC(secret, row1_payload + prev_hash + random_1)
```

Call this hash `A1`.

### Row 2

```text
event_type = FILE_UPLOADED
entity_type = file
entity_id = 555
prev_hash = A1
nonce = random_2
row_hash = HMAC(secret, row2_payload + prev_hash + random_2)
```

Call this hash `B2`.

### Row 3

```text
event_type = APPROVAL_GRANTED
entity_type = claim
entity_id = 9001
prev_hash = B2
nonce = random_3
row_hash = HMAC(secret, row3_payload + prev_hash + random_3)
```

Call this hash `C3`.

If row 2 changes, `B2` changes. Then row 3's `prev_hash` no longer matches. Verification fails.

If someone changes row 2 and recomputes row 2 and row 3, they still need the HMAC secret. Without the secret, they cannot produce valid `B2` and `C3`.

## 4) External Checkpoint Anchoring Under Your Constraints

You said S3 object versioning/object lock is not available. That means S3 cannot be treated as auditor-grade immutable evidence.

Printing the last hash in logs is useful only if those logs are shipped to a place the database administrator cannot rewrite. If logs remain local to the same server or same admin boundary, they are weak evidence.

### Practical Options Without DevOps Requests

#### Option A: Application Log Checkpoint

At startup, and after low-volume APIs such as create-project, print:

```text
AUDIT_CHAIN_CHECKPOINT chain_seq=12345 row_hash=abc123... created_at=2026-05-08T10:00:00Z
```

This helps if application logs are already collected by a managed platform, container log system, or SIEM.

If logs are not externally retained, this is only operational evidence, not strong auditor proof.

#### Option B: Checkpoint Table

Create a checkpoint table:

```sql
CREATE TABLE audit_chain_checkpoints (
    id bigserial PRIMARY KEY,
    checkpoint_type text NOT NULL,
    chain_seq bigint NOT NULL,
    row_hash text NOT NULL,
    created_at timestamptz NOT NULL DEFAULT now(),
    note text
);
```

This is useful for internal verification, but it is in the same database. A privileged attacker can rewrite it.

#### Option C: Manual Auditor Evidence

For real auditor proof without DevOps support, the simplest external anchor is manual but effective:

1. Once per day or month, generate a checkpoint line.
2. Send it to a company email distribution list.
3. Attach it to a ticketing system record.
4. Include it in a signed PDF report.
5. Store it in any system outside the database administrator's control.

Even email or ticket history can be stronger than an S3 bucket without object lock, because it creates third-party timestamps and distribution copies.

#### Option D: Public Timestamping

If allowed later, publish only the chain head hash, not sensitive data, to a public timestamping service or immutable ledger. This gives very strong proof without exposing audit contents.

### Recommended Checkpoint Function

```sql
CREATE OR REPLACE FUNCTION create_audit_chain_checkpoint(p_checkpoint_type text, p_note text DEFAULT NULL)
RETURNS TABLE(chain_seq bigint, row_hash text, checkpoint_at timestamptz)
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO audit_chain_checkpoints (checkpoint_type, chain_seq, row_hash, note)
    SELECT p_checkpoint_type, last_chain_seq, last_hash, p_note
    FROM audit_chain_state
    WHERE id = 1
    RETURNING audit_chain_checkpoints.chain_seq,
              audit_chain_checkpoints.row_hash,
              audit_chain_checkpoints.created_at
    INTO chain_seq, row_hash, checkpoint_at;

    RETURN NEXT;
END;
$$;
```

FastAPI can call this at startup or after create-project and log the returned values.

## Verification SQL

Auditor-side verification should recompute the chain in order and compare stored values.

With HMAC, the verifier needs controlled access to the HMAC secret or a verification function that can use it without exposing it.

Basic continuity check:

```sql
SELECT
    chain_seq,
    id,
    prev_hash,
    lag(row_hash) OVER (ORDER BY chain_seq) AS expected_prev_hash,
    row_hash
FROM audit_logs
ORDER BY chain_seq;
```

Find broken links:

```sql
WITH ordered AS (
    SELECT
        chain_seq,
        id,
        prev_hash,
        row_hash,
        lag(row_hash) OVER (ORDER BY chain_seq) AS expected_prev_hash
    FROM audit_logs
)
SELECT *
FROM ordered
WHERE chain_seq > 1
  AND prev_hash IS DISTINCT FROM expected_prev_hash;
```

This only checks chain continuity. Full verification must recompute HMAC values from canonical row payloads.

## Alembic `op.execute()` Implementation Sketch

This is the kind of migration you can run without asking DevOps, assuming your migration user can create functions, triggers, and possibly `pgcrypto`.

```python
def upgrade():
    op.execute("CREATE EXTENSION IF NOT EXISTS pgcrypto")

    op.execute("""
    CREATE TABLE IF NOT EXISTS audit_chain_secret (
        id integer PRIMARY KEY CHECK (id = 1),
        secret bytea NOT NULL,
        created_at timestamptz NOT NULL DEFAULT now()
    )
    """)

    op.execute("""
    INSERT INTO audit_chain_secret (id, secret)
    VALUES (1, gen_random_bytes(32))
    ON CONFLICT (id) DO NOTHING
    """)

    op.execute("""
    CREATE TABLE IF NOT EXISTS audit_chain_state (
        id integer PRIMARY KEY CHECK (id = 1),
        last_chain_seq bigint,
        last_hash text,
        updated_at timestamptz NOT NULL DEFAULT now()
    )
    """)

    op.execute("""
    INSERT INTO audit_chain_state (id, last_chain_seq, last_hash)
    VALUES (1, 0, repeat('0', 64))
    ON CONFLICT (id) DO NOTHING
    """)

    op.execute("""
    CREATE TABLE IF NOT EXISTS audit_chain_checkpoints (
        id bigserial PRIMARY KEY,
        checkpoint_type text NOT NULL,
        chain_seq bigint NOT NULL,
        row_hash text NOT NULL,
        created_at timestamptz NOT NULL DEFAULT now(),
        note text
    )
    """)

    # Then add columns, triggers, partition creation function, and monthly partitions.
```

## Final Recommendation

For this codebase and your constraints, implement this in phases:

1. Add append-only triggers immediately.
2. Add `prev_hash`, `row_hash`, `nonce`, `hash_algo`, `chain_version`, and `chain_seq`.
3. Use a database trigger to compute HMAC-SHA256 hashes.
4. Add monthly PostgreSQL range partitions.
5. Create next 12 monthly partitions from Alembic.
6. Add checkpoint creation and print checkpoint hashes in app logs.
7. For auditor-grade proof, establish one manual or external checkpoint process outside the database.

The most honest audit statement is:

> The audit log is append-only at the database layer, partitioned by month, protected by HMAC-based hash chaining, and periodically checkpointed. Normal application users and table-level attackers cannot rewrite history without detection. Full proof against privileged database-owner compromise requires at least one checkpoint outside the database trust boundary.
