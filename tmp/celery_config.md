# Celery OCR Processing

## Scope

Regulatory document processing and marketing asset OCR/claim extraction are fully owned by Celery. There is no FastAPI endpoint that directly executes either OCR workflow.

The only API entry point is metadata creation:

```text
POST /v2/files/add-file-metadata
```

When `doc_type` is `regulatory`, this endpoint creates or updates the regulatory document record, publishes `tasks.regulatory.process_regulatory_document`, and returns immediately with `processing_task_id`.

When `doc_type` is `marketing`, this endpoint creates the marketing asset record, publishes `tasks.asset_ocr.process_asset_ocr`, and returns immediately with `processing_task_id`. The former direct asset-processing endpoint has been removed.

## Components

| Component | Responsibility |
| --- | --- |
| `label/routes/files.py` | Validates metadata, persists regulatory document or marketing asset records, uploads the source to S3, and publishes JSON Celery tasks. |
| `label/celery_app.py` | Bootstraps environment variables and configures Redis, task routing, acknowledgement, visibility timeout, events, and worker logging. |
| `label/tasks/regulatory.py` | Owns the regulatory task and the complete execution workflow. |
| `label/tasks/asset_ocr.py` | Owns marketing asset OCR and claim extraction, including page image fallback, response IDs, progress, billing, audit, and output metadata. |
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
    API -->|JSON task: regulatory or asset_ocr queue| Redis[(Redis)]
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

    W->>R: Reserve task from regulatory or asset_ocr queue
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
  --concurrency=4 \
  --loglevel=info \
  --queues=regulatory,asset_ocr
```

`-A celery_app` imports `label/celery_app.py`. That module calls `bootstrap_environment()`, creates the Celery application, and imports `tasks.regulatory` and `tasks.asset_ocr`.

The worker listens to the `regulatory` and `asset_ocr` queues. Other future task types should use separate named queues and worker sizing appropriate to their workloads.

## How The Worker Actually Runs

There are four separate concepts that are easy to mix up:

| Term | Meaning | In this QA setup |
| --- | --- | --- |
| Docker container | The isolated Linux environment that runs the worker command. | One container: `label-celery`. |
| Celery worker | The Celery program started by the container command. It receives messages from Redis and manages task execution. | One worker program. |
| Pool child | A unit that can execute one Celery task at a time. It is either a process or a thread, depending on `--pool`. | Four Python processes because `--pool=prefork --concurrency=4`. |
| Queue | A named Redis-backed waiting line for tasks. A queue does not provide CPU or create a worker by itself. | `regulatory` and `asset_ocr`. |

The current QA command is:

```bash
celery -A celery_app worker \
  --pool=prefork \
  --concurrency=4 \
  --loglevel=info \
  --queues=regulatory,asset_ocr
```

It creates this shape:

```text
Docker container: label-celery
|
|-- Celery parent process
|   |-- receives messages from Redis
|   |-- manages acknowledgements, signals, and child processes
|   |
|   |-- Celery child process 1: can run one regulatory task
|   |-- Celery child process 2: can run one regulatory task
|   |-- Celery child process 3: can run one regulatory task
|   `-- Celery child process 4: can run one regulatory task
|
`-- consumes Redis queues: regulatory, asset_ocr
```

Therefore, at most four OCR tasks run simultaneously across regulatory and marketing assets. A fifth task waits in Redis until a child process becomes free.

### Process Versus Thread

A **process** is an independent running Python interpreter. It has its own memory, event loop, and Python global state. The operating system schedules it independently.

A **thread** is a smaller execution path inside one process. Threads in the same process share memory, globals, and interpreter state.

Python's Global Interpreter Lock (GIL) means normal Python CPU work in one process cannot run truly in parallel across multiple threads. Threads are still useful when work spends most of its time waiting for network or file I/O, such as sending requests to an external API.

Compare the two Celery pool choices:

```text
Prefork: --pool=prefork --concurrency=4

one Celery parent
|-- Python process 1: one active task
|-- Python process 2: one active task
|-- Python process 3: one active task
`-- Python process 4: one active task

Threads: --pool=threads --concurrency=8

one Python process
|-- thread 1: one active task
|-- thread 2: one active task
|-- ...
`-- thread 8: one active task
```

The reference translation project uses the second model: one container, one Celery process, and eight shared threads. That is reasonable when its tasks mostly wait on external services. It does not mean eight is a universal target or that threads are always better.

### Celery Pool Types

The `--pool` option selects what a Celery worker uses for its execution slots. `--concurrency` is the number of slots in that selected pool; it does not always mean the same thing.

| Pool | Example | A concurrency value means | Good fit | Important trade-off |
| --- | --- | --- | --- | --- |
| `prefork` | `--pool=prefork --concurrency=4` | Four independent Python child processes. | CPU-heavy work, native libraries, OCR, PDF processing, and most production tasks. | Each child needs its own memory. |
| `threads` | `--pool=threads --concurrency=10` | Ten threads inside one Python process. | Work that mostly waits for HTTP, database, or file I/O. | Threads share memory; normal Python CPU work is limited by the GIL. |
| `gevent` | `--pool=gevent --concurrency=100` | One process with many cooperative greenlets. | Very high volumes of compatible non-blocking network I/O. | Requires compatible monkey-patched libraries; CPU/blocking work can stall all greenlets. |
| `eventlet` | `--pool=eventlet --concurrency=100` | One process with many cooperative greenlets. | Older/eventlet-compatible high-volume network I/O systems. | Same blocking/compatibility concerns as gevent; CPU-bound work is a poor fit. |
| `solo` | `--pool=solo` | One task in the main worker process at a time. | Debugging, local reproduction, and very small workloads. | No parallel task execution. |

Celery documents `prefork` as the default, robust choice for CPU-bound work and most applications. Alternative pools can disable or limit features such as soft time limits and `max_tasks_per_child`; use them only when the workload and third-party libraries are known to fit.

### Reference Project: `gist_dev`

`/home/ubuntu/vscode/gist_dev/docker-compose.prod.yaml` uses one container named `semantic` for API, migration commands, and Celery. Its Celery portion is:

```bash
celery -A settings.celery_app worker \
  -Q default,embedding,summarisation,doc_chat,download_chat,doctran,embed_images,create_standalone_question,generate_slides,upload_audio,upload_video,embed_transcript \
  -E \
  -l info \
  --pool=threads \
  --concurrency=10
```

This means one shared ten-thread pool consumes thirteen queues:

```text
13 named queues -> one Celery worker -> 10 shared threads
```

It does not reserve threads for any one queue. Ten active `embedding` tasks can make a `download_chat` or `upload_audio` task wait, even though those tasks have distinct queue names.

Its `genai/settings.py` creates `settings.celery_app`, uses Redis as the broker and result backend, and includes task modules for documents, images, conversations, audio, and video. The production Compose command, not `settings.py`, chooses the threads pool and the list of consumed queues.

The `semantic` command backgrounds migrations and `python main.py`, then keeps Celery in the foreground. This is a compact deployment style, but it does not independently supervise the API, migrations, and worker. It is useful as a reference for a shared-thread worker, not a recommended replacement for this project's separate API and `label-celery` containers.

### Async LangChain Does Not Choose The Celery Pool

This project already uses LangChain and async provider calls:

- `utils/llm_factory.py` creates LangChain `ChatOpenAI` models and agents.
- Regulatory image annotation uses `await agent.ainvoke(...)`.
- Asset processing uses `AsyncOpenAI` and awaits `client.responses.create(...)`.

`await` is valuable because the Python code can do other work *inside the same task* while it waits for Azure OpenAI or another provider to respond. It does not make the Celery worker automatically run another Celery task in the freed time.

For example, the regulatory Celery task runs its async workflow with:

```python
_get_event_loop().run_until_complete(_run_regulatory_workflow(...))
```

The `await` calls make provider I/O efficient within that workflow, but the prefork child remains assigned to that document until the workflow completes. With `--pool=prefork --concurrency=4`, at most four Celery tasks are active at the same time.

The future asset/verification tasks should use LangChain's async interfaces such as `await agent.ainvoke(...)`. Where one task has several independent LLM calls, use `asyncio.gather(...)` or LangChain `abatch(...)` with a bounded `max_concurrency`. The bound must match Azure OpenAI quota and the application's shared rate limit; unrestricted `gather` can create a burst of provider requests and trigger throttling.

### Shared Prefork Versus Shared Threads For This Application

The following are two valid one-container, one-worker-pool options. Both can consume `regulatory` plus the future asset queues. Neither reserves a slot for one queue, so a busy queue can still make another queue wait.

| Choice | Example command | What `concurrency` means | Strength | Main risk |
| --- | --- | --- | --- | --- |
| Shared prefork | `celery -A celery_app worker --pool=prefork --concurrency=4 --queues=regulatory,asset_ocr,asset_claim_extraction,asset_reference,asset_verification` | Four independent Python processes, each running one task. | Robust isolation for the existing PDF/OCR/native-library work; true CPU parallelism. | Four full Python processes consume more memory; no queue-level capacity reservation. |
| Shared threads | `celery -A celery_app worker --pool=threads --concurrency=16 --queues=regulatory,asset_ocr,asset_claim_extraction,asset_reference,asset_verification` | Sixteen threads in one Python process, each able to run one task. | Many tasks can wait for remote LLM responses concurrently with modest extra memory. | PDF/image/Python CPU work still competes for the GIL; blocking or non-thread-safe code can stall or destabilize the whole worker. |

For a four-CPU container, `prefork --concurrency=4` is a sensible initial ceiling because there can be one CPU-capable Python task per allocated CPU. It is not a strict law: `prefork --concurrency=8` is valid even when Docker Compose sets `cpus: "4.0"`. Compose applies a CPU quota; it does not require the Celery process count to equal that quota. When more than four children are CPU-active at once, all eight compete for the same four CPUs, so each gets less CPU time and CPU-bound work usually takes longer. If most children are waiting on external APIs, a higher count can still improve throughput. Memory, provider quotas, connection limits, and task behavior ultimately determine a useful limit.

LangChain does not require `--pool=threads`. An async LangChain task can run correctly inside a prefork child. In fact, this is the safest starting point for mixed work:

```text
one worker, prefork=4
|-- regulatory task: PDF/OCR/native work plus Mistral waits
|-- asset OCR task: PDF/page work plus Azure OpenAI waits
|-- asset claim extraction/reference/verification: LangChain/OpenAI waits
`-- each task can use await internally without sharing Python process state with another task
```

Recommended first production topology while all task types share one worker:

```bash
celery -A celery_app worker \
  --pool=prefork \
  --concurrency=4 \
  --loglevel=info \
  --queues=regulatory,asset_ocr,asset_claim_extraction,asset_reference,asset_verification
```

Keep the native-library limits at one thread per prefork child. Add explicit Redis-backed provider semaphores or rate limits for Mistral, Azure OpenAI chat, and Azure embeddings. This prevents four Celery tasks, each using `abatch`, from exceeding a provider's request/token quota.

### Four-Slot Shared-Pool Timeline

The following is a time-based illustration for the requested burst. All seven tasks arrive at $t=0$: Regulatory 1, Asset OCR 1, Regulatory 2, Regulatory 3, Regulatory 4, Asset Verification 1, and Asset Verification 2. Each task lasts 30 seconds and consists entirely of three sequential 10-second external API calls. The Celery process remains occupied for the task's full 30 seconds, even while an async call is waiting on the provider.

```mermaid
gantt
    title Shared prefork=4: seven tasks arrive at t=0
    dateFormat  YYYY-MM-DD HH:mm:ss
    axisFormat  %S seconds
    section Prefork process 1
    Regulatory 1 API 1             :p1r1a1, 2026-01-01 00:00:00, 10s
    Regulatory 1 API 2             :p1r1a2, after p1r1a1, 10s
    Regulatory 1 API 3             :p1r1a3, after p1r1a2, 10s
    Regulatory 4 API 1             :p1r4a1, after p1r1a3, 10s
    Regulatory 4 API 2             :p1r4a2, after p1r4a1, 10s
    Regulatory 4 API 3             :p1r4a3, after p1r4a2, 10s
    section Prefork process 2
    Asset OCR 1 API 1              :p2ao1a1, 2026-01-01 00:00:00, 10s
    Asset OCR 1 API 2              :p2ao1a2, after p2ao1a1, 10s
    Asset OCR 1 API 3              :p2ao1a3, after p2ao1a2, 10s
    Asset Verification 1 API 1     :p2av1a1, after p2ao1a3, 10s
    Asset Verification 1 API 2     :p2av1a2, after p2av1a1, 10s
    Asset Verification 1 API 3     :p2av1a3, after p2av1a2, 10s
    section Prefork process 3
    Regulatory 2 API 1             :p3r2a1, 2026-01-01 00:00:00, 10s
    Regulatory 2 API 2             :p3r2a2, after p3r2a1, 10s
    Regulatory 2 API 3             :p3r2a3, after p3r2a2, 10s
    Asset Verification 2 API 1     :p3av2a1, after p3r2a3, 10s
    Asset Verification 2 API 2     :p3av2a2, after p3av2a1, 10s
    Asset Verification 2 API 3     :p3av2a3, after p3av2a2, 10s
    section Prefork process 4
    Regulatory 3 API 1             :p4r3a1, 2026-01-01 00:00:00, 10s
    Regulatory 3 API 2             :p4r3a2, after p4r3a1, 10s
    Regulatory 3 API 3             :p4r3a3, after p4r3a2, 10s
    Idle after 30 seconds           :p4idle, after p4r3a3, 30s
```

| Time | Processes 1-4 | Queued work |
| --- | --- | --- |
| $0$-$30$s | Regulatory 1, Asset OCR 1, Regulatory 2, Regulatory 3 | Regulatory 4, Asset Verification 1, Asset Verification 2 |
| $30$-$60$s | Regulatory 4, Asset Verification 1, Asset Verification 2, one idle process | None |

The second row assumes the shown arrival order, `worker_prefetch_multiplier=1`, and that the worker can immediately reserve a task from each queue. It demonstrates capacity, not a strict multi-queue ordering promise: named queues do not reserve the newly free process for a particular task type.

### Eight Prefork Processes With A Four-CPU Quota

This doubled burst has 14 tasks at $t=0$: eight regulatory, two asset OCR, and four asset-verification tasks. With `--pool=prefork --concurrency=8` and Docker Compose `cpus: "4.0"`, eight tasks start immediately and six wait for the first 30-second wave to finish. Under this diagram's assumption that each task is only three external API waits, the CPU is mostly idle while it waits; the remote provider, not this container's CPU, is doing the model inference.

```mermaid
gantt
  title Shared prefork=8 with cpus=4: fourteen tasks arrive at t=0
  dateFormat  YYYY-MM-DD HH:mm:ss
  axisFormat  %S seconds
  section Prefork process 1
  Regulatory 1 API wait          :p8r1, 2026-01-01 00:00:00, 30s
  Asset OCR 2 API wait           :p8ao2, after p8r1, 30s
  section Prefork process 2
  Asset OCR 1 API wait           :p8ao1, 2026-01-01 00:00:00, 30s
  Regulatory 6 API wait          :p8r6, after p8ao1, 30s
  section Prefork process 3
  Regulatory 2 API wait          :p8r2, 2026-01-01 00:00:00, 30s
  Regulatory 7 API wait          :p8r7, after p8r2, 30s
  section Prefork process 4
  Regulatory 3 API wait          :p8r3, 2026-01-01 00:00:00, 30s
  Regulatory 8 API wait          :p8r8, after p8r3, 30s
  section Prefork process 5
  Regulatory 4 API wait          :p8r4, 2026-01-01 00:00:00, 30s
  Asset Verification 3 API wait  :p8av3, after p8r4, 30s
  section Prefork process 6
  Asset Verification 1 API wait  :p8av1, 2026-01-01 00:00:00, 30s
  Asset Verification 4 API wait  :p8av4, after p8av1, 30s
  section Prefork process 7
  Asset Verification 2 API wait  :p8av2, 2026-01-01 00:00:00, 30s
  section Prefork process 8
  Regulatory 5 API wait          :p8r5, 2026-01-01 00:00:00, 30s
```

| Time | Eight prefork processes | Four-CPU quota effect |
| --- | --- | --- |
| $0$-$30$s | Eight tasks are active; each waits for its three external API responses. Six tasks remain queued. | Usually low CPU usage, because API/model inference happens on Azure OpenAI, Mistral, or another remote provider. No meaningful CPU contention in this idealized all-I/O interval. |
| $30$-$60$s | The remaining six tasks are active; two prefork processes are idle. | Usually low CPU usage under the all-I/O assumption. |
| When several tasks parse PDFs/images, serialize results, run OCR/native code, or execute other Python CPU work | Up to eight active processes can simultaneously need CPU. | The Docker scheduler gives all runnable processes a share of the same four CPUs. This is CPU contention: compute portions slow down and context-switch overhead increases. |

In the current non-streaming OpenAI path, `await client.responses.create(...)` waits for one completed remote response. While that request is in progress, its Celery child is occupied and cannot run a second Celery task, but it normally consumes very little CPU. In streaming mode, the remote model still performs the inference; this process additionally receives and handles response chunks as they arrive. Streaming can improve time-to-first-output for an interactive client, but it does not by itself free the Celery process to run another task or make the provider generation faster.

This project is not a pure OpenAI-wait workload. Regulatory processing has PDF/image work, Mistral/LlamaParse/LiteParse integration, result handling, database writes, and `asyncio.to_thread(...)` calls for blocking work. Therefore, `prefork=8` may improve throughput when external waits dominate, but it can also increase memory usage, provider pressure, and CPU contention during those local portions. Keep `prefork=4` as the QA baseline; test `prefork=8` with representative documents and measure CPU throttling, RSS memory, latency, and provider throttling before changing production.

Use a shared threads pool only after QA measurements establish all of the following:

1. The selected queues are overwhelmingly provider-wait time, not PDF/image/Python CPU time.
2. Every blocking operation is moved to an async SDK call or a bounded `asyncio.to_thread(...)` call.
3. The LangChain models, agent/checkpointer, database access, and shared application state are safe to use across worker threads.
4. Azure OpenAI and Mistral request-per-minute and token-per-minute limits are enforced globally, not merely per thread.

If those conditions are proven for *asset-only* work, a threads worker can be a later optimization. Do not switch the mixed regulatory-and-asset worker to high-concurrency threads merely because LangChain has `ainvoke`; regulatory OCR retains synchronous PDF/native sections and should stay on prefork.

### Why This Project Uses Prefork

Regulatory processing can use PDF/OCR/native libraries, substantial memory, blocking SDK calls, and process-local asyncio synchronization. For this workload, `prefork` is safer because each Celery child has isolated Python and asyncio state.

If one child crashes because of a native-library problem, Celery can replace that child without corrupting another task's in-memory state. Separate processes also allow CPU-heavy Python work to run in parallel across CPU cores, subject to the container CPU limit.

The QA default concurrency is `4`, matching the worker's `cpus: "4.0"` limit. This means four documents may run simultaneously and approximately multiplies the memory and provider demand compared with one task. `prefork=8` remains a supported configuration under this CPU limit, but eight children compete for four CPUs whenever they are CPU-active and require roughly twice the child-process memory. Keep four until measured CPU, memory, provider quotas, Redis semaphore limits, and document latency justify another change.

### Why Native Thread Limits Are Set To One

Each prefork child can call libraries such as NumPy, OpenBLAS, MKL, OpenMP, or tokenizers. Those libraries may quietly create their own thread pools.

Without limits, a configuration that appears to run two tasks can turn into many runnable operating-system threads:

```text
4 Celery child processes x 8 native-library threads each = up to 32 compute threads
```

On a container limited to four CPUs, that causes contention and unpredictable processing time. The worker service sets these values so each Celery child starts with one native compute thread:

| Setting | Effect |
| --- | --- |
| `OMP_NUM_THREADS=1` | Limits OpenMP worker threads. |
| `OPENBLAS_NUM_THREADS=1` | Limits OpenBLAS worker threads. |
| `MKL_NUM_THREADS=1` | Limits Intel MKL worker threads. |
| `NUMEXPR_NUM_THREADS=1` | Limits NumExpr worker threads. |
| `TOKENIZERS_PARALLELISM=false` | Prevents Hugging Face tokenizer threading issues after Celery forks child processes. |
| `PYTHONFAULTHANDLER=1` | Emits Python trace information when a native extension crashes; it does not add worker capacity. |

With the current defaults, the intended capacity is approximately four active documents and one main native compute thread per document. This is predictable and can be changed deliberately after measurement.

### Queues And Dedicated Capacity

A queue is a waiting line, not a worker pool. This command makes one pool consume two queues:

```bash
celery -A celery_app worker --pool=prefork --concurrency=4 --queues=regulatory,asset_ocr
```

```text
regulatory queue --\
                    > one shared pool with four process slots
asset_ocr queue ---/
```

If four regulatory tasks are running, an asset task waits. The queue names keep task types organized, but they do not reserve a process slot.

Dedicated capacity requires separate Celery worker programs, for example:

```text
regulatory worker: --queues=regulatory --concurrency=3
asset worker:      --queues=asset_ocr --concurrency=1
```

These workers are normally separate containers. They can also run in one container only when a process supervisor manages both worker programs. That is more complex and still requires an explicit capacity decision whenever a new dedicated queue is added.

For the current deployment, one `label-celery` container with one prefork worker and concurrency `4` consumes both OCR queues.

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
  celery -A celery_app worker --pool=prefork --concurrency=2 --loglevel=info --queues=regulatory,asset_ocr
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
