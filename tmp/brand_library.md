<!-- markdownlint-disable -->

# Brand Library

Feature implementation notes and technical architecture for the brand library: a project-independent library of brands (folders) and the files inside them, designed for enterprise scalability with disk-based streaming, automatic ZIP unpacking, and backend metadata autofill.

Revision: `a7b8c9d0e1f2_brand_library_23` (revises `9c0d1e2f3a4b`)

---

## 1. Core design decision: DB owns names, S3 owns bytes

Everything else follows from this separation of concerns.

| Concern | Where it lives | Why |
| --- | --- | --- |
| User-facing name (`user_foldername`, `user_filename`) | Postgres only | Renameable, may contain spaces/unicode/anything |
| Storage name (`s3_folder_stem`, `s3_file_stem`) | Postgres + S3 | Immutable, sanitized, never shown to the user |
| File bytes | S3 only | Local filesystem is used only as staging during streaming uploads |

Consequences:
- **Rename is DB-only.** S3 has no rename primitive; a "rename" is copy + delete, which for a folder means N non-atomic operations that can half-fail. Since the stem is never user-visible, there is nothing to rename in S3. One `UPDATE`, done.
- **Delete is DB-only** (soft delete). Same reasoning, plus soft delete keeps the audit trail and makes restore trivial and idempotent.
- **List is DB-only**, so the brand library screen never pays for an S3 `LIST`.

## 2. S3 layout

`utils/s3_utils.build_s3_key()` already prepends `{S3_ROOT_PREFIX}/data/`, so relative keys in the brand code start at `brand/`:

```text
label/data/brand/                                   <- brand root prefix
label/data/brand/{s3_folder_stem}/                  <- zero-byte folder marker object
label/data/brand/{s3_folder_stem}/{s3_file_stem}    <- file object (stem includes extension)
```

Real example:
```text
brand "Braftovi (US) 2024!"  ->  stem "Braftovi_US_2024"
file  "Report Final V2.PDF"  ->  stem "Report_Final_V2.pdf"

folder_s3_key : label/data/brand/Braftovi_US_2024/
file_s3_key   : label/data/brand/Braftovi_US_2024/Report_Final_V2.pdf
```

Stems are produced by `utils/naming.sanitize_object_stem()`: everything outside `[a-zA-Z0-9_-]` becomes `_`, runs collapse, leading/trailing `._-` are trimmed, capped at 120 chars, empty falls back to `brand`/`file`. No spaces, no special characters.

## 3. Uniqueness and the "no `_1` `_2`" rule

Enforced by **partial unique indexes scoped to live rows**:

```sql
CREATE UNIQUE INDEX ux_brands_user_foldername_active
  ON label_schema.brands (lower(user_foldername)) WHERE active;
CREATE UNIQUE INDEX ux_brands_s3_folder_stem_active
  ON label_schema.brands (lower(s3_folder_stem)) WHERE active;
CREATE UNIQUE INDEX ux_brand_files_user_filename_active
  ON label_schema.brand_files (brand_id, lower(user_filename)) WHERE active;
CREATE UNIQUE INDEX ux_brand_files_s3_file_stem_active
  ON label_schema.brand_files (brand_id, lower(s3_file_stem)) WHERE active;
```

Three things fall out of `WHERE active`:
1. **Duplicate name -> `409` with a clear message.** Never an auto-suffix.
2. **Archived rows never block a fresh create.** Delete `Braftovi` (row becomes `Braftovi_archive`, `active=false`), create `Braftovi` again, delete again -> a second `Braftovi_archive` is fine, because archived rows are outside the index. This prevents collisions without needing auto-suffixing.
3. **Case-insensitive** (`lower(...)`), so `Braftovi` and `braftovi` cannot coexist.

## 4. Delete / restore semantics

Delete (`DELETE /v2/brand/{brand_id}`, `DELETE /v2/brand/file/{brand_file_id}`):
- `active = false`
- Name renamed `X` -> `X_archive` (files: `report.pdf` -> `report_archive.pdf`)
- `archived_at`, `archived_by`, `last_modified_by` stamped
- Deleting a brand **cascades** to its live files in the same transaction
- **S3 objects are left in place** (`s3_objects_removed: false` in audit log)
- **Idempotency**: Deleting an already-archived brand or file is treated as a no-op success instead of returning an error.

Restore (via unified `PUT /v2/brand` or `PUT /v2/brand/file` with `active=True`) strips the suffix and re-checks both name and stem availability, returning `409` if the live name was taken in the meantime.

## 5. Tables

`label/DAL/models/tables.py` — `Brand` (24 columns) and `BrandFile` (46 columns). Both use `String` PKs holding `uuid7()` via `db._new_id()`.

Value sets are enforced by DB `CHECK` constraints:
```sql
ck_brand_files_document_type    -- NULL OR IN ('PI','ISI','ASSET','SCOPE','ZIP','CUSTOM')
ck_brand_files_ai_document_type -- same set
ck_brand_files_filetype         -- IN ('pdf','word','zip','txt','json','ppt')
```

## 6. Disk-Based Staging Pipeline & Metadata Autofill

### Why Disk-Based Streaming?
In initial drafts, file uploads used an in-memory stream (`upload_bytes` / `download_bytes`). In an enterprise setting where concurrent users upload large files (500MB+ presentations or high-resolution documentation), buffering whole payloads in RAM immediately leads to Out-Of-Memory (OOM) fatal crashes. 

To eliminate this memory hazard:
- In-memory buffering helpers (`upload_bytes` and `download_bytes`) have been completely stripped from `label/utils/s3_utils.py`.
- They are replaced by `upload_file_to_s3_key` / `upload_file` and `download_file_by_key`, which perform chunked disk-to-S3 streaming directly via `boto3`.
- The upload handler streams incoming files directly to temporary storage on disk while simultaneously calculating SHA-256 hashes and file sizes in small memory chunks (1MB blocks).
- Temporary directories are guaranteed to be cleaned up via strict `try...finally` teardown blocks using `shutil.rmtree(..., ignore_errors=True)`.

### Automatic ZIP Decompression & Ingestion
When a user uploads a `.zip` file, the backend automatically intercepts and decompresses it on disk:
- The archive is expanded inside a secure temporary workspace.
- The system walks the directory tree recursively (`rglob("*")`), ignoring system artifacts (such as `.DS_Store` or `__MACOSX` resource forks).
- Every supported document found inside the archive is ingested as an independent `brand_file` record under the target brand folder.
- The outer ZIP container file itself is discarded after decompression, avoiding duplicate disk or S3 consumption.

### LiteParse Extraction
`label/utils/brand_metadata.py` owns metadata extraction and hosts `get_brand_liteparse_parser()` (which was relocated from `llm_factory.py` to keep LLM factories strictly focused on agent models).
- `extract_file_metadata(file_path: Path, filetype: str)` operates strictly on local file paths.
- Parser config has expensive visual processing switched off while keeping complexity detection enabled:
```python
output_format="text", ocr_enabled=False, ocr_failure_fatal=False,
image_mode="off", extract_images=False, extract_links=False,
extract_annotations=False, extract_form_fields=False, extract_structure_tree=False,
extract_xfa_packets=False, extract_vector_graphics=False,
extract_text_metadata=False, emit_word_boxes=False,
include_complexity=True, max_pages=1000, quiet=True
```

### Inline vs Deferred Processing
| Filetype | Metadata Extraction | AI Classification |
| --- | --- | --- |
| `pdf` | inline (ms via PyMuPDF/LiteParse) | background task (`gpt-5-mini`) |
| `word` | **background task** (LibreOffice required) | background task (`gpt-5-mini`) |
| `ppt` | **background task** (LibreOffice required) | not classified |
| `txt`, `json` | inline (fast plain-text decode) | not classified |
| `zip` | uncompressed on disk; contents processed individually | not classified |

## 7. AI Document Type Classification

Background task -> `classify_brand_document_type()` -> `get_brand_document_type_agent()`, wrapping `get_regulatory_image_annotation_model()` (`gpt-5-mini`).
- Structured schema: `{ AI_document_type: enum, confidence: int }`.
- Results with confidence `< 50` are treated as low-confidence and discarded.
- `ai_document_type` is advisory only and never overwrites a user-specified `document_type`.

## 8. SHA-256 Deduplication

When an uploaded file matches the SHA-256 hash of an existing live file in the database (across any brand), a duplicate relationship is established (`duplicate_of_file_id`). Furthermore, any OCR local or S3 cache keys (`mistral_ocr_*`, `ocr_*`) are automatically **inherited** by the new record, saving downstream processing time.

## 9. RBAC

`admin` and `project_manager` roles have read/write/delete permissions. `reviewer` and `viewer` roles are read-only. Enforced strictly at the API route boundary via synchronous gating helpers (`_validate_brand_access_sync`, `_validate_brand_id_sync`, and `_validate_brand_file_id_sync`).

## 10. API Inventory (12 Streamlined RESTful CRUD Endpoints)

The original 19 fragmented action endpoints have been consolidated down to **12 standardized RESTful endpoints** under `/v2`, tag `v2-brand-library`, inside `label/routes/brand_lib.py`.

| Method | RESTful Endpoint | Access | Summary |
| --- | --- | --- | --- |
| `POST` | `/v2/brand` | Write | Create brand folder in DB and S3 marker |
| `GET` | `/v2/brand/list` | Read | Paginated list of brands with active filter & search |
| `GET` | `/v2/brand/{brand_id}` | Read | Get single brand details & live file count |
| `PUT` | `/v2/brand` | Write | Unified update: rename, metadata, owner change, or restore (`active=True`) |
| `DELETE` | `/v2/brand/{brand_id}` | Delete | Idempotent soft-delete; cascades to all live files |
| `POST` | `/v2/brand/file` | Write | Multipart upload; supports disk streaming & recursive ZIP extraction |
| `GET` | `/v2/brand/file/list` | Read | Paginated list of brand files with filters |
| `GET` | `/v2/brand/file/{brand_file_id}` | Read | Retrieve full metadata and processing flags |
| `PUT` | `/v2/brand/file` | Write | Unified file update: rename, set `document_type`, owners, or restore |
| `DELETE` | `/v2/brand/file/{brand_file_id}` | Delete | Idempotent soft-delete for file |
| `GET` | `/v2/brand/file/{brand_file_id}/download-url` | Read | Generate 30-min presigned GET download URL |
| `POST` | `/v2/brand/file/{brand_file_id}/reprocess` | Write | Re-run background metadata and AI classification |

## 11. Transactional Integrity

The disk-based upload pipeline strictly orders operations to ensure atomicity without memory strain:
1. Stream incoming upload directly to temporary disk workspace while calculating SHA-256 and enforcing file size caps.
2. If ZIP archive, unpack to disk and enumerate valid document contents.
3. For each file, perform inline metadata extraction from disk (outside database transactions so no locks are held).
4. Open transactional `db_connection()`:
   a. Verify name and storage stem uniqueness -> `409 Conflict` if existing.
   b. Identify SHA-256 duplicate sources and inherit OCR cache keys.
   c. Execute `INSERT` statement and `flush()`.
   d. Perform disk-to-S3 streaming upload via `upload_file`.
   e. Record audit log entry.
5. Commit database transaction and purge temporary disk staging workspace in `finally` block.

## 12. Files Touched

```text
label/routes/brand_lib.py                    Refactored to 12 RESTful CRUD endpoints & disk/zip streaming
label/utils/brand_metadata.py              Changed extraction to disk Paths; acquired get_brand_liteparse_parser
label/utils/s3_utils.py                      Removed upload_bytes/download_bytes; added disk streaming helpers
label/utils/llm_factory.py                   Removed brand-specific parser factory (moved to brand_metadata)
docs/temp/brand_library.md                   Updated documentation & frontend handover guide
```

---

## 16. Frontend Handover & Architectural Guide

This section acts as a complete handover and theoretical reference for frontend engineers integrating with the refactored Brand Library API.

### Endpoint Mapping Table (Old 19 -> New 12 RESTful API)

To simplify frontend logic and state management, fragmented RPC-style APIs have been replaced with Resource-Oriented CRUD methods:

| Legacy Endpoint (V1 Draft) | New Streamlined REST Endpoint | Implementation Guidance for Frontend |
| --- | --- | --- |
| `POST /v2/brand/create` | `POST /v2/brand` | Send JSON body with `user_foldername` and optional metadata. |
| `GET /v2/brand/list` | `GET /v2/brand/list` | Supports query params: `active=true/false`, `search`, `limit`, `offset`. |
| `GET /v2/brand/get?brand_id=...` | `GET /v2/brand/{brand_id}` | Pass `brand_id` directly in URL path. Returns live `file_count`. |
| `PUT /v2/brand/rename` | `PUT /v2/brand` | Send JSON body `{ "brand_id": "...", "user_foldername": "New Name" }`. |
| `PUT /v2/brand/update` | `PUT /v2/brand` | Send JSON body with `brand_id` and any subset of editable fields. |
| `PUT /v2/brand/change-owner` | `PUT /v2/brand` | Send JSON body `{ "brand_id": "...", "current_owner": 123 }`. |
| `POST /v2/brand/restore` | `PUT /v2/brand` | Send JSON body `{ "brand_id": "...", "active": true }` to unarchive. |
| `DELETE /v2/brand/delete` | `DELETE /v2/brand/{brand_id}` | Idempotent soft delete via URL path parameter. |
| `POST /v2/brand/file/upload` | `POST /v2/brand/file` | Send `multipart/form-data` with `file`, `brand_id`, optional `user_filename`. |
| `GET /v2/brand/file/list` | `GET /v2/brand/file/list` | Query params: `brand_id`, `active`, `document_type`, `limit`, `offset`. |
| `GET /v2/brand/file/get` | `GET /v2/brand/file/{brand_file_id}` | Pass `brand_file_id` in URL path. Used by metadata screen. |
| `PUT /v2/brand/file/rename` | `PUT /v2/brand/file` | Send JSON body `{ "brand_file_id": "...", "user_filename": "doc.pdf" }`. |
| `PUT /v2/brand/file/update-metadata` | `PUT /v2/brand/file` | Send JSON body with `brand_file_id` and fields (e.g. `document_type`). |
| `PUT /v2/brand/file/change-owner` | `PUT /v2/brand/file` | Send JSON body `{ "brand_file_id": "...", "current_owner": 456 }`. |
| `POST /v2/brand/file/restore` | `PUT /v2/brand/file` | Send JSON body `{ "brand_file_id": "...", "active": true }`. |
| `DELETE /v2/brand/file/delete` | `DELETE /v2/brand/file/{brand_file_id}` | Idempotent soft delete via URL path parameter. |
| `GET /v2/brand/file/download-s3-url` | `GET /v2/brand/file/{brand_file_id}/download-url` | Returns JSON containing presigned `url` and filename. |
| `POST /v2/brand/file/reprocess-metadata` | `POST /v2/brand/file/{brand_file_id}/reprocess` | Triggers background extraction and AI classification re-run. |
| `GET /v2/brand/s3-folders` | *(Removed / Consolidated)* | Internal diagnostic tool removed from external public API surface. |

### Key Theoretical Concepts for Frontend Integration

#### 1. Unified Partial Updates (`PUT` Endpoints)
You no longer need different API wrappers for renaming, assigning owners, or editing field values. 
Both `PUT /v2/brand` and `PUT /v2/brand/file` accept a **partial payload (PATCH-style semantic over PUT)**. 
- You MUST supply the primary ID (`brand_id` or `brand_file_id`).
- You ONLY need to include the exact fields changed by the user in the form screen. Unset fields remain untouched in the database.
- Example: To complete the metadata screen, send:
  ```json
  {
    "brand_file_id": "018f...-....",
    "document_type": "PI",
    "region": "US",
    "priority": "High"
  }
  ```
  Setting `document_type` automatically switches `file_metadata_set: true` in the backend response.

#### 2. Automatic ZIP Archive Expansion
When a user drops a `.zip` archive into the file uploader (`POST /v2/brand/file`), the backend automatically unfolds the archive and processes all internal valid documents (`.pdf`, `.docx`, `.json`, etc.) individually.
- Because a single upload request can generate multiple files, `BrandFileUploadResponse` provides an `items: List[BrandFileItem]` array alongside a backward-compatible `file: Optional[BrandFileItem]` (which defaults to the first item).
- **Recommendation**: Always bind your UI grid or file listing updates to the `response.items` array so uploaded ZIP contents instantly display in the folder view.
- The `duplicates` field lists any newly ingested file IDs that were recognized as exact SHA-256 matches of existing documents.

#### 3. Idempotent Operations & Error Resilience
- **Deletions (`DELETE`)**: Executing soft-delete against an already-archived brand or file returns `200 OK` with `success: true` rather than throwing a error. This ensures UI retry loops and cleanup routines do not get stuck on archived objects.
- **Name Conflicts**: If a user attempts to rename or create a file/brand to an active name, the API returns `409 Conflict` with a human-readable message suitable for direct rendering in UI toasts or modal error boxes.

#### 4. Background Processing & Polling Workflow
Because parsing 500MB Word documents or running AI classification models (`gpt-5-mini`) can take several seconds, these tasks execute asynchronously on disk in backend worker threads after the HTTP upload finishes.
- The upload response returns boolean indicator flags: `metadata_pending: true/false` and `ai_classification_pending: true/false`.
- If either flag is `true`, your UI should display a subtle progress spinner on the file row and poll `GET /v2/brand/file/{brand_file_id}` every 2–3 seconds until the fields populated by background extraction (`word_count`, `total_pages`, `ai_document_type`, etc.) settle.
