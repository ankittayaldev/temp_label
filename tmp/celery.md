# Celery Regulatory Processing

## Scope

Regulatory document processing is fully owned by Celery. There is no FastAPI endpoint that directly executes the regulatory pipeline.

The only API entry point is metadata creation:

```text
POST /v2/files/add-file-metadata
```

When `doc_type` is `regulatory`, this endpoint creates or updates the regulatory document record, publishes a Celery task, and returns immediately with `processing_task_id`.

Marketing asset processing still uses FastAPI `BackgroundTasks`; it is outside this Celery migration.

## Components

| Component | Responsibility |
| --- | --- |
| `label/routes/files.py` | Validates metadata, persists the document record, uploads the source to S3, and publishes a JSON Celery task. |
| `label/celery_app.py` | Bootstraps environment variables and configures Redis, task routing, acknowledgement, visibility timeout, events, and worker logging. |
| `label/tasks/regulatory.py` | Owns the regulatory task and the complete execution workflow. |
| `label/processors/regulatory_pipeline.py` | Runs the provider chain: primary Mistral, fallback Mistral, LlamaParse, then LiteParse. |
| `label/processors/document_metadata.py` | Shared document metadata helpers for source SHA and first-page extraction. |
| Redis | Celery broker and result backend. |
| PostgreSQL | Project authorization, advisory locks, progress, processing runs, billing events, audit events, and document output metadata. |
| S3 | Durable source and generated output storage. |
| Shared Docker volume | Working files used by the API and worker while processing. |

## Architecture

```mermaid
flowchart LR
    Client[Client] -->|POST metadata| API[FastAPI: label-backend]
    API -->|source and metadata| S3[(S3)]
    API -->|document record| PG[(PostgreSQL)]
    API -->|JSON task: regulatory queue| Redis[(Redis)]
    API -->|success + processing_task_id| Client

    Redis -->|reserve task| Worker[Celery: label-celery]
    Worker -->|authorization, lock, progress, run records| PG
    Worker -->|read working PDF| Volume[(Shared data volume)]
    Worker --> Pipeline[Regulatory pipeline]
    Pipeline --> Mistral[Mistral primary/fallback]
    Pipeline --> Llama[LlamaParse fallback]
    Pipeline --> Lite[LiteParse fallback]
    Pipeline -->|generated files| Volume
    Worker -->|upload outputs| S3
    Worker -->|success/failure, billing, audit, output keys| PG
    Worker -->|task result state| Redis
```

## Request And Task Flow

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant A as FastAPI
    participant P as PostgreSQL
    participant S as S3
    participant R as Redis
    participant W as Celery Worker
    participant O as OCR Providers

    C->>A: POST /v2/files/add-file-metadata
    A->>P: Validate project access
    A->>S: Upload source PDF or converted Word PDF
    A->>P: Create/update RegulatoryDocument
    A->>R: Publish process_regulatory_document
    R-->>A: Celery task ID
    A-->>C: Metadata response + processing_task_id

    W->>R: Reserve task from regulatory queue
    W->>P: Revalidate project write access
    W->>P: Acquire document advisory lock
    W->>P: Set progress to 0
    W->>P: Check current/recent idempotent result

    alt Existing reusable Mistral result
        W->>P: Attach existing output metadata
    else New processing required
        W->>P: Insert processing run: started
        W->>O: Mistral primary
        alt Primary failed
            W->>O: Mistral fallback
        end
        alt Mistral providers failed
            W->>O: LlamaParse
        end
        alt LlamaParse failed
            W->>O: LiteParse
        end
        W->>S: Upload markdown and structured JSON
        W->>P: Complete run, document keys, billing, audit
    end

    W->>P: Clear progress and release advisory lock
    W->>R: Acknowledge task and store result
```

## Celery Task Payload

The API publishes only JSON-compatible scalar data:

```json
{
  "key": "Upload/example.pdf",
  "project_id": "project-id",
  "document_id": "document-id",
  "user_id": "123",
  "email": "user@example.com",
  "roles": ["project_manager"],
  "designation": "project_manager",
  "client_host": "10.0.0.10"
}
```

FastAPI `Request`, dependency objects, database sessions, and Pydantic request models are not serialized into Redis.

## Worker Startup

The worker starts from `/home/label`, which is the Docker image working directory:

```bash
celery -A celery_app worker \
  --pool=prefork \
  --concurrency=2 \
  --loglevel=info \
  --queues=regulatory
```

`-A celery_app` imports `label/celery_app.py`. That module calls `bootstrap_environment()`, creates the Celery application, and imports `tasks.regulatory`.

The worker listens only to the `regulatory` queue. Other future task types should use separate named queues and worker sizing appropriate to their workloads.

## Why Prefork

Use Celery's `prefork` pool, not the threads pool.

The regulatory pipeline contains process-local asyncio synchronization and runs blocking provider SDK calls through its own internal threads. Prefork gives each Celery child process an isolated event loop and semaphore state while preserving provider-level concurrency controls.

The default concurrency is `2` because regulatory OCR is memory-heavy. Raise it only after measuring memory, CPU, provider quotas, Redis semaphore limits, and processing latency.

## Reliability And Idempotency

- `task_acks_late=True`: Redis is acknowledged only after successful task completion.
- `task_reject_on_worker_lost=True`: a task is requeued when a worker process exits unexpectedly.
- `worker_prefetch_multiplier=1`: a worker child reserves one long-running document at a time.
- Visibility timeout is six hours. It must be longer than the maximum expected single-document execution time.
- A PostgreSQL advisory lock prevents concurrent execution for the same project and document.
- Current-document and recent-document result checks make duplicate delivery idempotent.
- Provider fallback is handled inside one task attempt. Celery automatic retries are intentionally not used for provider exhaustion.
- Progress is cleared and the advisory lock is released in a `finally` block.

## Provider Order

The worker attempts providers in this order:

1. Configured primary Mistral model.
2. Configured fallback Mistral model, when enabled.
3. LlamaParse.
4. LiteParse.

If every provider fails, the worker:

1. Writes the detailed regulatory error log.
2. Marks the processing run as failed.
3. Inserts a failure audit event.
4. Re-raises the exception so Celery records the task as failed.

## API Behavior

Successful enqueue response includes:

```json
{
  "success": true,
  "processing_task_id": "celery-task-uuid"
}
```

The response also contains the existing regulatory document metadata fields.

If Redis is unavailable while publishing, the API returns HTTP `503` with:

```text
Regulatory processing queue is unavailable.
```

The metadata record has already been created, so the metadata request can be retried. Duplicate task publication is protected by the worker's advisory lock and idempotency checks.

There is intentionally no endpoint such as:

```text
POST /v2/process/process-regulatory
```

Regulatory execution is asynchronous and Celery-only.

## QA Deployment

`docker-compose-qa.yaml` starts:

- `label-update`: FastAPI container.
- `label-celery`: Celery worker container.
- `label-redis`: Redis broker/result backend.

The API and worker use the same image, environment file, Docker network, and `label_app_data` volume. The worker exposes no HTTP port.

Start QA services with:

```bash
docker compose -f docker-compose-qa.yaml up --build -d
```

View worker logs with:

```bash
docker logs -f label-celery
```

## Dev EC2 Deployment

`.github/workflows/deploy.yml` deploys two containers from the same ECR image:

- `label-backend`: runs the Dockerfile API command.
- `label-celery`: overrides the image command with the Celery worker command.

Both containers mount:

```text
/home/ubuntu/label-update-backend/label/.env -> /home/label/.env
label-backend -> /home/label/data
```

## Local WSL Test

Build and start the API using the existing local command. Then start the worker from the same image, network, environment, and data volume:

```bash
GW=$(docker network inspect labelnew_default -f '{{(index .IPAM.Config 0).Gateway}}') && \
docker rm -f label-celery 2>/dev/null || true && \
docker run -d --init \
  --name label-celery \
  --cpus=2 \
  --memory=3g \
  --network labelnew_default \
  -v label-api-data:/home/label/data \
  -v /home/ubuntu/vscode/indegene/.env:/home/label/.env \
  -e AWS_EC2_METADATA_SERVICE_ENDPOINT="http://${GW}:1338" \
  -e AWS_SECRETS_NAME="development" \
  -e DATABASE_URL="postgresql://labelapp:postgres@postgres:5432/labeldb" \
  -e REDIS_URL="redis://:redis_pass@redis:6379/0" \
  --restart unless-stopped \
  label-api:latest \
  celery -A celery_app worker --pool=prefork --concurrency=2 --loglevel=info --queues=regulatory
```

View logs:

```bash
docker logs -f label-celery
```

## Smoke Checks

Run inside the API or worker image from `/home/label`:

```bash
celery -A celery_app report
celery -A celery_app inspect registered
celery -A celery_app inspect active_queues
celery -A celery_app inspect active
celery -A celery_app inspect reserved
```

Expected task:

```text
tasks.regulatory.process_regulatory_document
```

Expected queue:

```text
regulatory
```

## Operational Checks

Confirm the worker is running:

```bash
docker ps --filter name=label-celery
```

Confirm Redis connectivity from the worker:

```bash
docker exec label-celery celery -A celery_app inspect ping
```

Inspect recent worker logs:

```bash
docker logs --tail 200 label-celery
```

When diagnosing a stuck document, check:

1. Celery task state and worker logs.
2. PostgreSQL processing run and progress records.
3. PostgreSQL advisory lock ownership.
4. Redis availability and queue depth.
5. Provider timeout, quota, and fallback logs.
6. Shared volume input/output paths.
7. S3 upload failures and output keys.


----

**2. What happens during CI/CD while processing is running**

The short answer is:

> The Celery message should not be permanently lost, but an interrupted task does not resume from the page where it stopped. It reruns the workflow.

The current deployment runs `docker stop label-celery` in deploy.yml.

There are three cases:

1. **Task is still queued**

   It remains in Redis. Restarting the API has no effect because the API has already published the message. The new worker receives it normally.

2. **Task is active and finishes during shutdown**

   `docker stop` sends `SIGTERM`. Celery performs a warm shutdown and tries to finish active tasks before exiting. Since `task_acks_late=True` in celery_app.py, the message is acknowledged only after successful completion.

3. **Task takes longer than Docker’s stop timeout**

   Docker normally waits only about 10 seconds and then sends `SIGKILL`. A regulatory OCR task will usually exceed that.

   In that case:

   - The Python `except` and `finally` blocks do not run.
   - The unfinished processing work is terminated.
   - The processing run may remain incorrectly marked `started`.
   - The progress value may temporarily remain stale.
   - The PostgreSQL advisory lock is released when the dead process’s database connection closes.
   - The Redis message remains unacknowledged.
   - Because the configured Redis visibility timeout is six hours, the message can take **up to six hours** to become available again after a hard container kill.
   - Once redelivered, the new worker starts the regulatory workflow from the beginning.

This is **at-least-once execution**, not checkpoint-based resume.

On redelivery, the task first checks existing document results:

- If complete Mistral output metadata was already committed, it reuses that result.
- If processing was interrupted before the final database update, it reruns OCR.
- Current idempotency reuse is limited to `provider == "mistral"`. Completed LlamaParse or LiteParse results are not currently reused by these checks.

The files on the shared volume survive container replacement, but they are not a workflow checkpoint by themselves.

**Current deployment weakness**

The task queue is durable enough for worker replacement, assuming Redis remains available, but the deployment is not graceful enough for long-running jobs. The minimum correction is:

```bash
docker stop --time=21600 label-celery
```

A stronger production deployment would:

1. Start the new worker first.
2. Stop the old worker from consuming new work.
3. Send `SIGTERM` to the old worker.
4. Let its active tasks drain.
5. Remove the old worker only after it exits.

That avoids the six-hour redelivery delay and allows existing tasks to finish on the old code while newly queued tasks run on the new worker. Redis itself is not restarted by the current workflow, so queued messages survive the API and worker replacement.

