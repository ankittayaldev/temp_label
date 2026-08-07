# Celery Queue Configuration — What Changed

**Date:** 7 Aug 2026 · **Branch:** `feat/new-asset-pipeline`

## The problem

Queue names were written in four places: the app code, two docker-compose files, and the
deploy workflow. They drifted apart.

QA was listening to four of our five queues — `asset_html` was missing. So any HTML
marketing asset submitted on QA was accepted by the API, given a task ID, and then never
processed. Nothing errored. The job just sat in a queue with nothing reading it.

Production had the full list, so this only affected QA.

## What we changed

Queue names now live in exactly one place: `TASK_ROUTES` in `label/celery_app.py`.
Two things are now derived from it automatically:

- which queues a worker listens to
- which task modules a worker loads at startup

We removed the `--queues` flag from every worker command — local, QA, and production. A
worker without that flag now listens to every queue the app defines, so the environments
can't fall out of sync again.

We also enabled a safety check: publishing to a queue that isn't defined now fails
immediately with a clear error, instead of quietly creating a queue nobody reads.

## For developers

Adding a Celery task used to mean editing four files. Now it's one:

```python
# label/celery_app.py
BRAND_SYNC_QUEUE = "brand_sync"
BRAND_SYNC_TASK_NAME = "tasks.brand_sync.sync_brand_library"

TASK_ROUTES = {
    ...
    BRAND_SYNC_TASK_NAME: {"queue": BRAND_SYNC_QUEUE},
}
```

That's the whole change — local, QA and production pick it up on the next restart.

Two rules to keep it working:

- **Don't add `--queues` back** to any worker command. It overrides the list and the
  worker silently stops consuming whatever you leave off. That's the original bug.
- **Keep task names as `tasks.<module>.<function>`**, matching the file the function
  actually lives in. That's how the import list is worked out.

## Before this goes to QA

QA has a backlog of unprocessed `asset_html` jobs that has been building up. The new
worker will pick all of them up at once on startup. Check the size first:

```bash
docker exec label-redis redis-cli -a "$REDIS_PASSWORD" LLEN asset_html
```

If it's large, drain or trim it rather than letting the whole backlog hit the workers.

Also delete any `CELERY_QUEUES=` line from QA and production `.env` files — it is no
longer read by anything.

## Files changed

| File | Change |
|---|---|
| `label/celery_app.py` | Queues and task modules derived from `TASK_ROUTES` |
| `docker-compose.yaml` | Removed `--queues` |
| `docker-compose-qa.yaml` | Removed `--queues` |
| `.github/workflows/deploy.yml` | Removed `--queues` |
| `README.md` | Rewrote the "Celery Tasks Configuration" section |

## Verified

Tested against Celery 5.6.3 (the version we pin). A worker started with no `--queues`
consumes all six queues, all six tasks route to the right queue, and the auto-derived
module list is identical to the one that was previously hardcoded — so this is a
behaviour-preserving change for the tasks we have today, and a bug fix for QA.
