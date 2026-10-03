# rota-sync Agent Guide

## Project Goal

`rota-sync` is a web application for recording work hours and organizing schedules from user-provided images.

The intended first workflow is:

```text
Image upload
    -> Docling document processing
    -> deterministic domain parsing
    -> user review and correction
    -> confirmed work entries and schedule events
```

The repository is currently greenfield. Do not assume that a frontend, backend, database, or deployment setup already exists.

## Architectural Decisions

### Web application

- Use Elixir Phoenix as the main application framework.
- Use Phoenix LiveView for the user interface and interactive review workflow.
- Use Ecto for PostgreSQL access.
- Use Oban for durable background jobs, retries, and processing state transitions.
- Use Phoenix PubSub when processing progress must be pushed to a LiveView.

### Model runtime

- Use Docling as the primary document processing pipeline.
- Do not introduce a Python FastAPI service for the initial architecture.
- Use `Pythonx` to embed CPython in the Phoenix release and call Docling directly.
- Keep Docling calls behind an application boundary such as `Rota.ModelRunner` or `Rota.Docling`.
- Do not call Pythonx directly from LiveView callbacks or arbitrary request processes.
- Run inference through a supervised, dedicated worker, normally a `GenServer` or equivalent process.
- Load the Python interpreter, Docling pipeline, and model artifacts once during worker initialization where practical.
- Serialize inference initially. Pythonx embeds Python in the same OS process as BEAM, and Python's GIL means that multiple Elixir callers do not automatically provide useful Python concurrency.
- Keep the model runner replaceable so a future HTTP, gRPC, or separate inference deployment can be added without changing domain logic.

The future abstraction should conceptually support both implementations:

```text
Rota.ModelRunner.Pythonx
Rota.ModelRunner.Http
```

The first implementation is `Pythonx`.

### Docling pipeline

Start with Docling's standard pipeline rather than a downstream LLM or a VLM pipeline:

```text
image / PDF
    -> layout detection
    -> OCR
    -> table structure detection
    -> DoclingDocument
    -> domain parser
```

Docling is a pipeline, not one monolithic model. Treat these as separate model concerns:

- Layout model: finds document regions and semantic blocks.
- OCR engine: recognizes text inside regions.
- Table structure model: identifies rows, columns, cells, and relationships.
- Domain parser: converts recognized content into work and schedule candidates.

Do not use an LLM in the initial implementation. Use deterministic parsing, anchors, templates, regular expressions, and explicit validation rules until a concrete need for an LLM is established.

### Training and model improvement

- Model training is performed outside the Phoenix runtime using Python, PyTorch, Docling-compatible tooling, and GPU infrastructure as needed.
- The Phoenix application may record user corrections and create training candidates, but must not train models in web request processes or Oban jobs running inside the application.
- Model artifacts must be versioned and loaded by an explicit model manifest.
- Do not commit large model weights to Git. Store them in object storage, a model registry, a release artifact, or a mounted deployment volume.
- Every processing run must record Docling, pipeline, and model versions.
- User corrections are feedback data, not automatically trusted labels. They require filtering, annotation policy, and train/validation/test separation.

## Runtime Architecture

```text
Browser
  |
  v
Phoenix LiveView
  |
  +-- Accounts and authorization
  +-- Upload and review UI
  +-- Domain contexts
  +-- Ecto / PostgreSQL
  +-- Oban jobs
  +-- Supervised Pythonx Docling runner
  |
  +-- Object Storage for images and raw artifacts
```

The initial deployment may be a single Phoenix release containing the embedded Python runtime, Docling dependencies, and model artifacts. Keep the model runner isolated in code so it can later be moved to a separate process if native Python, CUDA, memory, or availability failures make in-process execution unsuitable.

## Responsibilities

### Phoenix contexts

Prefer domain contexts with clear ownership:

- `Accounts`: users, organizations, roles, and authorization.
- `Documents`: uploaded files, checksums, storage keys, and source metadata.
- `Processing`: processing runs, pipeline states, errors, and retries.
- `Docling`: Pythonx integration, pipeline configuration, and normalized output.
- `Reviews`: candidate values, field-level edits, confirmation, and audit history.
- `WorkEntries`: canonical work-hour records and work-hour rules.
- `ScheduleEvents`: canonical schedule and itinerary records.
- `Integrations`: future calendar or external-system synchronization.

Do not put business rules in LiveView modules. LiveViews should coordinate user interaction and call contexts.

### PostgreSQL

PostgreSQL is the source of truth for structured and transactional data:

- users, organizations, and permissions
- documents and processing metadata
- processing runs and job state
- Docling elements and normalized recognition results
- candidate records and review changes
- confirmed work entries and schedule events
- model manifests and inference metadata
- audit records

Use Ecto migrations for schema changes. Keep large images, model weights, and large raw files in object storage; PostgreSQL should store their keys, hashes, metadata, and relationships.

### Object storage

Use S3-compatible storage for:

- original uploaded images and PDFs
- derived page images or crops when needed
- raw Docling output snapshots
- model artifacts and evaluation datasets when appropriate

Validate file type, size, checksum, and storage ownership before processing.

### Oban

Use Oban for durable, retryable processing jobs. A typical flow is:

```text
CreateDocument
    -> RunDocling
    -> NormalizeCandidates
    -> CreateReview
    -> ConfirmCanonicalRecords
```

Jobs must be idempotent, have explicit timeouts, persist failure details, and avoid overwriting previous processing runs. Reprocessing a document creates a new run with a new pipeline and model version.

### Pythonx and Docling

Keep Python integration code small and explicit. Prefer a Python module under `priv/python/` over large inline Python strings. Use explicit Python globals and convert results at the boundary into application-owned Elixir data structures.

Avoid passing live Python objects between unrelated Elixir processes. Keep the Docling converter and Python objects inside the dedicated runner process unless Pythonx's safety and lifecycle are explicitly verified.

Pythonx, PyTorch, CUDA, and native dependencies must be tested in the same target environment used by the release. The build and deployment process must pin Python, Docling, PyTorch, CUDA-compatible wheels, and model versions.

## Data and Processing Rules

The pipeline must preserve the following layers instead of overwriting them:

```text
source document
    -> immutable processing run
    -> raw Docling result
    -> normalized candidate
    -> user corrections
    -> confirmed canonical record
```

A model result is never a confirmed work entry or schedule event by itself.

Each processing run should record at least:

- `docling_version`
- layout model version
- OCR model version
- table model version
- pipeline configuration hash
- input checksum
- processing status
- duration and error details

Prefer storing structured elements with page number, bounding box, text, confidence, reading order, and source relationship. Do not retain only Markdown when the review UI needs highlighting or provenance.

Domain validation, not Python or a model, owns rules such as:

- valid date and time formats
- end time after start time
- cross-day handling
- timezone requirements
- break-time rules
- overlapping work warnings or errors
- duplicate confirmation prevention

## Review UI Requirements

The review interface should show the original image alongside extracted values. When available, display OCR or layout bounding boxes and the source element for each candidate field.

The review flow must support:

- field-level editing
- confidence or review-needed indicators
- source provenance
- validation messages
- before/after change history
- explicit confirmation
- retry or reprocess with another model version

Confirmation should be transactional and idempotent. It must not create duplicate canonical records when the user retries or refreshes the page.

## Suggested Repository Layout

The exact layout may evolve with Phoenix setup, but the intended shape is:

```text
lib/
├── rota/
│   ├── accounts/
│   ├── documents/
│   ├── processing/
│   ├── docling/
│   ├── reviews/
│   ├── work_entries/
│   ├── schedule_events/
│   └── workers/
└── rota_web/
    ├── live/
    ├── components/
    └── controllers/

priv/
├── python/
└── repo/
    └── migrations/

ml/
├── datasets/
├── annotations/
├── train_layout/
├── train_ocr/
└── evaluate/
```

Model weights should not be stored under `priv/` unless they are intentionally bundled as a release artifact and the repository policy explicitly permits it.

## Testing Requirements

Add tests at the boundary where behavior is deterministic:

- context tests for work-hour and schedule rules
- parser tests using representative Docling fixtures
- Ecto migration and constraint tests
- Oban job tests for retries and idempotency
- model runner contract tests using a fake runner
- Pythonx integration tests in an environment with pinned dependencies
- LiveView tests for upload, review, correction, and confirmation
- golden dataset evaluations for layout, OCR, table, and end-to-end field accuracy

Do not make the full GPU model mandatory for ordinary unit tests. Use a fake or fixture-based runner and reserve real model execution for explicit integration or evaluation jobs.

## Non-goals for the Initial Version

Do not add these without a concrete requirement:

- a Python FastAPI service
- a separate microservice architecture
- a downstream LLM extraction stage
- Google Calendar or Outlook synchronization
- payroll calculations
- automatic model training from every user correction
- model weights committed to Git

## Implementation Guidance for Agents

- Read the existing code and status before changing files.
- Preserve unrelated user changes.
- Prefer Phoenix Contexts, Ecto schemas, migrations, and supervised processes over logic in LiveViews.
- Keep Pythonx behind a narrow boundary and avoid leaking `Pythonx.Object` into domain structs or database schemas.
- Keep raw model output and canonical business records separate.
- Add model and pipeline version metadata to every inference result.
- Use ASCII by default in source files and documentation unless the project already requires other characters.
- Add tests for behavior changes and run the smallest relevant formatter, test, and type checks available.
