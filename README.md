# Microplastic Degradation Research Data Extractor AI

> **This system is an AI-assisted scientific data extraction tool.**
> Extracted values should be verified against the original research paper before being used for
> publication, regulatory decisions, or scientific conclusions. The system does not independently
> establish that a reported process constitutes true polymer degradation or mineralization.

## Project overview

A self-hosted web application that turns research papers on microplastic **degradation,
biodegradation, fragmentation, weathering, oxidation, mineralization, mechanisms, products and
toxicity** into a structured, experiment-level dataset.

Give it a PDF, several PDFs, a folder, a DOI or a public paper URL. It extracts the text (with OCR
for scanned pages), finds the experimental sections and tables, asks a **local LLM running in
Ollama** to extract evidence-backed values, validates them, and lets a researcher accept, correct
or reject each experiment before exporting JSON, CSV or Excel.

Three design commitments run through the whole system:

1. **Traceability.** Every extracted value carries its page, section, table/figure number and a
   short verbatim quote. Quotes are searched for in the PDF text; values whose quote cannot be
   found are marked unverified and downgraded to low confidence.
2. **Scientific distinctions.** Fragmentation, particle-size reduction, surface erosion, surface
   oxidation, chain degradation, depolymerization, biodegradation and mineralization are recorded
   separately. Mass loss is never treated as mineralization; particle-size reduction is never
   treated as biodegradation. Rule checks flag records that blur these lines.
3. **No invented values.** Anything not reported is `null`. Ranges and qualified values
   (`25–30 °C`, `<5 %`) are preserved as text rather than collapsed to a number. Unusual values are
   flagged for review, never silently rejected or "fixed".

## Features

| Area | What it does |
|---|---|
| Inputs | Single or multiple PDFs, folder upload (browser) or server folder import (`data/inbox`), DOI, public URL |
| DOI | Format validation, Crossref metadata, PMID via Europe PMC, legal open-access PDF via Unpaywall / OpenAlex / Europe PMC. Paywalls are never bypassed; if no legal PDF exists you are asked to upload it |
| PDF processing | SHA-256, PyMuPDF layout-aware text (two-column reading order, running headers removed), section detection, captions, pdfplumber/Camelot tables, Tesseract OCR for scanned pages |
| Long papers | Relevance-scored passages → overlapping chunks (`CHUNK_SIZE`, `CHUNK_OVERLAP`, `MAX_CHUNKS`); references and back matter are never sent to the model |
| Two-stage AI | Stage 1: experiment roster + per-chunk evidence-backed facts. Stage 2: independent LLM review (warnings, suggested corrections, evidence level) |
| Multiple experiments | One record per experiment (e.g. PET + enzyme A, PET + enzyme B, PET + enzyme A + UV), shared conditions applied to all, timepoints kept as separate measurements |
| Validation | Pydantic schema, physical ranges, missing/incorrect units, internal consistency, contradictions preserved, duplicate experiments flagged, quote verification |
| Evidence level | AI-assigned *strong / moderate / weak / unclear* plus a deterministic rule-based cross-check; disagreements are flagged |
| Review | Accept / Edit / Reject per experiment, add/delete rows, full audit trail, every value marked *AI extracted* or *human corrected* |
| Results | Searchable, filterable, sortable, paginated table with column selection and multi-select export |
| Export | JSON (full records + evidence), CSV (seven table choices), XLSX (all tables + README sheet) |
| Model management | Ollama status, current model, installed models, one-click download of the configured model |
| Duplicates | SHA-256, DOI and title similarity; the same paper is never processed twice |
| Ops | Docker Compose, background worker with a crash-safe database queue, structured JSON logs, rate limiting, HTTPS via Caddy, nightly backups |

## Architecture

```mermaid
flowchart LR
    U[Researcher's browser] -->|HTTPS| RP[Caddy reverse proxy<br/>production only]
    RP --> FE[Frontend<br/>Next.js]
    U -. development .-> FE
    FE -->|/api/* proxy| BE[Backend API<br/>FastAPI]
    BE --> DB[(PostgreSQL)]
    W[Extraction worker] --> DB
    W --> OL[Ollama<br/>local LLM]
    BE --> OL
    BE --- V[(Volume: PDFs,<br/>processed text, exports)]
    W --- V
    BE -->|metadata, legal OA PDFs| EXT[Crossref · Unpaywall<br/>OpenAlex · Europe PMC]
```

Only the frontend (or Caddy in production) publishes a port. PostgreSQL, Ollama and the API are
reachable only on the private Docker network.

### Extraction pipeline

```mermaid
flowchart TD
    A[PDF] --> B[Validate, SHA-256, duplicate check]
    B --> C[Text extraction<br/>PyMuPDF + OCR fallback]
    C --> D[Section detection<br/>+ captions + tables]
    D --> E[Relevant passage retrieval]
    E --> F[Overlapping chunks]
    F --> G[Stage 1a: experiment roster]
    G --> H[Stage 1b: evidence-backed facts per chunk]
    H --> I[Assembly per experiment<br/>shared vs specific facts, contradictions]
    I --> J[Quote verification + rule checks]
    J --> K[Stage 2: LLM review]
    K --> L[Deduplication]
    L --> M[(Database)]
    M --> N[Researcher review]
    N --> O[JSON / CSV / XLSX]
```

The model never fills a giant JSON template. It returns **facts**, each with its own evidence:

```json
{"experiment": "E1", "field": "experimental_conditions.temperature", "value": 30, "unit": "°C",
 "page": 4, "section": "Materials and Methods", "quote": "flasks were incubated at 30 °C",
 "basis": "directly_measured", "confidence": "high"}
```

That makes source evidence a by-product of extraction, allows sparse answers from small chunks,
and lets the assembler detect conflicting statements across chunks.

## Tech stack

| Layer | Technology |
|---|---|
| Frontend | Next.js 15 (App Router), React 19, TypeScript, Tailwind CSS 4 |
| Backend | Python 3.12, FastAPI, Pydantic 2, SQLAlchemy 2, Alembic |
| AI | Ollama (any local model, set by `OLLAMA_MODEL`), provider abstraction for future cloud models |
| Documents | PyMuPDF, pdfplumber, Camelot, Tesseract (pytesseract), RapidFuzz |
| Database | PostgreSQL 16 |
| Export | openpyxl, csv, json |
| Infrastructure | Docker, Docker Compose, Caddy (HTTPS) |

## Installation

### Requirements

* Docker Engine 24+ with the Compose plugin (or Docker Desktop on Windows/macOS).
* Disk: ~10 GB for images and data plus the model (an 8B model is ~5 GB).
* See [Server requirements](#server-requirements) for CPU/RAM/GPU sizing.

### Docker setup (quick start)

```bash
# 1. Install Docker: https://docs.docker.com/get-docker/
# 2. Get the code
git clone <your-repo-url> microplastic-degradation-ai     # or unzip the archive
cd microplastic-degradation-ai

# 3. Create .env (set POSTGRES_PASSWORD; review OLLAMA_MODEL and CONTACT_EMAIL)
cp .env.example .env

# 4. Start everything
docker compose up -d

# 5. The model downloads automatically on first start (ollama-pull service).
#    Follow progress with:
docker compose logs -f ollama-pull

# 6. Open the application
#    http://localhost:3000
```

`./scripts/setup.sh` does steps 3–6 for you (and generates a random database password). Use
`./scripts/setup.sh --gpu` on a machine with an NVIDIA GPU.

What starts:

| Service | Purpose |
|---|---|
| `frontend` | Web UI on `FRONTEND_PORT` (default 3000); proxies `/api/*` to the backend |
| `backend` | REST API; runs database migrations on start |
| `worker` | Background extraction jobs |
| `postgres` | Database (volume `pgdata`) |
| `ollama` | Local LLM server (volume `ollama_models`) |
| `ollama-pull` | One-shot: downloads `OLLAMA_MODEL` |

Uploaded PDFs, processed text and exports live in the `appdata` volume (`/data` in the containers).

### Ollama setup

The model is configured only through `.env`:

```env
OLLAMA_MODEL=llama3.1
```

Change models:

```bash
# edit OLLAMA_MODEL in .env, then
docker compose exec ollama ollama pull qwen2.5:14b
docker compose up -d            # restarts backend/worker with the new setting
docker compose exec ollama ollama list
```

The **Model settings** page shows the connection status, the current model, all installed models,
and can download the configured model if it is missing.

**GPU (recommended).** With an NVIDIA GPU and the
[NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html):

```bash
docker compose -f docker-compose.yml -f docker-compose.gpu.yml up -d
```

On Windows, Docker Desktop with the WSL 2 backend and a current NVIDIA driver provides GPU access
to containers.

**Model choice.** Extraction quality depends heavily on the model. Instruction-tuned models of
7–8B parameters (`llama3.1`, `qwen2.5:7b`) work; 14B+ models (`qwen2.5:14b`, `mistral-small`) are
noticeably more reliable at keeping experiments apart and quoting verbatim. Keep
`OLLAMA_NUM_CTX` ≥ 8192 so a prompt, a chunk and the answer fit.

## Environment variables

All variables are documented in [`.env.example`](.env.example). The most important:

| Variable | Default | Meaning |
|---|---|---|
| `POSTGRES_PASSWORD` | *(required)* | Database password |
| `FRONTEND_PORT` / `FRONTEND_BIND` | `3000` / `0.0.0.0` | Where the UI is published |
| `OLLAMA_MODEL` | `llama3.1` | Model used for extraction |
| `OLLAMA_TIMEOUT` | `600` | Seconds per model call |
| `OLLAMA_NUM_CTX` | `8192` | Context window (tokens) |
| `CHUNK_SIZE` / `CHUNK_OVERLAP` | `6000` / `600` | Characters per chunk / shared between chunks |
| `MAX_CHUNKS` | `24` | Max chunks per paper (most relevant kept) |
| `VALIDATION_LLM_ENABLED` | `true` | Stage-2 LLM review |
| `EXTRACTION_TIMEOUT_SECONDS` | `3600` | Hard limit per paper |
| `OCR_ENABLED` / `OCR_LANGUAGE` | `true` / `eng` | Tesseract OCR for scanned pages |
| `TABLE_EXTRACTOR` | `auto` | `auto`, `pdfplumber`, `camelot`, `none` |
| `CONTACT_EMAIL` | *(empty)* | Polite-pool e-mail for Crossref/OpenAlex; required for Unpaywall |
| `MAX_UPLOAD_MB` | `50` | Upload size limit |
| `RATE_LIMIT_PER_MINUTE` / `UPLOAD_RATE_LIMIT_PER_MINUTE` | `120` / `30` | Per-client limits |
| `WORKER_CONCURRENCY` | `1` | Parallel extractions per worker |
| `DOMAIN` | *(empty)* | Public domain for HTTPS (production) |

## Usage

1. **Add papers** on the dashboard: drop PDFs (or a folder), enter a DOI, or paste a public URL.
   *Paper information* (title, authors, journal, year, DOI) appears immediately.
2. **Extraction status** shows the six steps live: document processed → text extracted → tables
   detected → experimental sections identified → AI extraction completed → data validated.
3. **Review** each experiment on the paper page. Click a page reference such as `p.4` to see the
   highlighted source sentence and open the PDF at that page. Flags, contradictions and the AI
   reviewer's suggested corrections are listed above the data. Use **Accept**, **Edit** or
   **Reject**.
4. **Results** lists all experiments with search, filters (polymer, evidence level, degradation
   type, review status, flags), sorting, pagination and column selection. Select rows to export.
5. **Export** JSON, CSV or XLSX: everything except rejected experiments, only reviewed ones, or a
   selection.

Re-running an extraction (for example with a larger model) replaces only unreviewed experiments.
Experiments you accepted, corrected or rejected are kept as they are.

For a large collection, copy PDFs into the `data/inbox` folder of the backend volume
(`docker compose cp ./my-papers/. backend:/data/inbox/`) and press **Import folder** on the
Upload page.

## API documentation

Interactive OpenAPI docs: `http://localhost:3000/api/docs` (proxied from the backend).

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/papers/upload` | Multipart upload (`files`, many allowed; `auto_extract`) |
| POST | `/api/papers/doi` | `{"doi": "...", "auto_extract": true}` |
| POST | `/api/papers/url` | `{"url": "...", "auto_extract": true}` |
| POST | `/api/papers/import-folder` | Import PDFs from `DATA_DIR/inbox` |
| POST | `/api/papers/{id}/pdf` | Attach a PDF to a metadata-only paper |
| GET | `/api/papers` | Library (`search`, `status`, `page`, `page_size`) |
| GET | `/api/papers/{id}` | Paper with experiments and runs |
| GET | `/api/papers/{id}/pdf` | The stored PDF |
| GET | `/api/papers/{id}/text?page=N` | Extracted text of one page |
| DELETE | `/api/papers/{id}` | Delete paper, files and experiments |
| POST | `/api/extraction/start` | `{"paper_id": 1, "force": false, "model": null}` |
| GET | `/api/extraction/{id}` | Run status, progress, stages, warnings (`include_log=true` for model I/O) |
| GET | `/api/experiments/{id}` | Full experiment record |
| PATCH | `/api/experiments/{id}` | `{"changes": {"experimental_conditions.temperature": 30, "degradation_outcomes[id=4].mass_loss_percent": 12.4}, "note": "..."}` |
| POST | `/api/experiments/{id}/review` | `{"action": "accept" \| "reject" \| "reset"}` |
| POST / DELETE | `/api/experiments/{id}/rows/{section}[/{row_id}]` | Add / delete list rows |
| GET | `/api/experiments/{id}/history` | Review audit trail |
| GET | `/api/results` | Flat table (`search`, `polymer`, `evidence_level`, `review_status`, `degradation_type`, `flagged`, `sort`, `order`, `page`, `page_size`) |
| GET | `/api/export/json` · `/csv` · `/xlsx` | `ids`, `paper_id`, `accepted_only`, `include_rejected`; CSV also `table` |
| GET | `/api/health` | Database and Ollama status |
| GET | `/api/settings/llm` | Model status and pipeline configuration |
| POST | `/api/settings/llm/pull` | Download the configured model |
| GET | `/api/schema` | Field catalogue (used by the UI) |

Errors are returned as `{"detail": "<human-readable message>", "code": "<machine code>"}`, e.g.
`invalid_pdf`, `corrupted_pdf`, `ocr_failed`, `doi_not_found`, `paper_inaccessible`,
`llm_unavailable`, `model_not_installed`, `malformed_llm_json`, `extraction_timeout`,
`database_unavailable`, `file_too_large`, `rate_limited`.

## Database schema

```mermaid
erDiagram
    papers ||--o{ extraction_runs : has
    papers ||--o{ experiments : contains
    extraction_runs ||--o{ experiments : produced
    experiments ||--|| microplastics : "particles + pretreatment"
    experiments ||--|| degradation_agents : uses
    experiments ||--|| experimental_conditions : "medium + environment"
    experiments ||--o{ analytical_methods : measured_by
    experiments ||--o{ degradation_results : "measurements (timepoints)"
    experiments ||--o{ degradation_products : forms
    experiments ||--o{ toxicity_results : tested_for
    experiments ||--o{ source_evidence : supported_by
    experiments ||--o{ review_events : audited_by
```

* `papers` – bibliographic metadata, file hash/size, OCR flags, status, duplicate links.
* `extraction_runs` – job queue and history: status, progress, stage checklist, model, settings
  snapshot, warnings, model call log, timing.
* `experiments` – one per experiment: experimental design and mineralization columns, mechanism
  (JSON), AI and rule-based evidence levels, contradictions, validation report, per-field
  provenance (`field_meta`), review status.
* `microplastics`, `degradation_agents`, `experimental_conditions` – one-to-one typed columns.
* `analytical_methods`, `degradation_results`, `degradation_products`, `toxicity_results` –
  one-to-many rows, each with confidence, origin, page and quote.
* `source_evidence` – every value's page/section/table/figure/quote, confidence, basis,
  `verified` and origin.
* `review_events` – who changed what (old/new value, note, time).

Typed columns are generated from the Pydantic schema, so database, API and exports always agree.
The concrete DDL is in `database/migrations/versions/0001_initial_schema.py`.

## Extraction schema

Each experiment record has the structure below (see `backend/app/schemas/extraction.py` for all
fields):

```text
paper_metadata          title, authors, journal, publisher, publication_year, doi, pmid, url, abstract, keywords
microplastic            polymer_type … particle_size_min/max/mean + unit … crystallinity, additives, color
pretreatment            pretreatment_used/type/method, uv_*, chemical/oxidation/thermal/mechanical, aging_*, weathering
degradation_agent       degradation_type, biological_or_abiotic, microorganism/species/strain, enzyme (+conc./activity), catalysts
experimental_conditions medium, water type, pH, salinity …, temperature + unit, incubation_time + unit, light, oxygen, agitation
experimental_design     masses/concentrations/volumes, replicates, sampling intervals, controls, statistical_analysis
analytical_methods[]    method_name, instrument, measurement_target, result, unit, initial_value, final_value, change
degradation_outcomes[]  per timepoint: mass loss %, degradation %, biodegradation %, mineralization %, Mw/Mn, particle size,
                        carbonyl index, rate, half-life, measurement_method, statistics
mineralization          mineralization_observed, CO2 production + unit, O2 consumption, TOC/DOC change, carbon mineralization %
mechanism               entries{chain_scission, hydrolysis, …: status + basis (directly_demonstrated | proposed_by_authors |
                        ai_inference) + evidence}, proposed_mechanism, mechanism_evidence, mechanism_confidence
degradation_products[]  product_name, type, formula, concentration + unit, formation_observed, analytical_method, toxicity
toxicity[]              test_performed, organism, cell_line, endpoint, exposure_time, concentration, before/after/change
evidence_quality        evidence_level (AI-assigned), rationale, rule_based_level, assessment_label
source_evidence[]       field, value, unit, source{page, section, table_number, figure_number, quote}, confidence, basis, verified, origin
```

## Example output

Abridged JSON export of one experiment from the synthetic test paper used by the test suite
(empty fields removed):

```json
{
  "experiment_label": "E1",
  "description": "PET + purified cutinase; polymer: PET; agent: cutinase",
  "review_status": "pending",
  "paper_metadata": {"title": "Enzymatic and microbial degradation of PET microplastics: a synthetic test document",
                     "authors": ["A. Tester", "B. Fixture"], "doi": "10.1234/mpai.test.2026.001"},
  "microplastic": {"polymer_type": "PET", "particle_size_min": 100.0, "particle_size_max": 300.0,
                   "particle_size_mean": 180.0, "particle_size_unit": "µm"},
  "degradation_agent": {"degradation_type": "enzymatic_degradation", "enzyme": "cutinase",
                        "enzyme_concentration": "0.5 mg/mL"},
  "experimental_conditions": {"pH": 8.0, "temperature": 30.0, "temperature_unit": "°C",
                              "incubation_time": 14.0, "incubation_time_unit": "days", "agitation_speed": "150 rpm"},
  "experimental_design": {"replicates": 3, "abiotic_control": "without enzyme or cells"},
  "analytical_methods": [
    {"method_name": "FTIR", "measurement_target": "carbonyl index", "initial_value": 0.21, "final_value": 0.35, "change": "increase"},
    {"method_name": "HPLC", "measurement_target": "terephthalic acid"}
  ],
  "degradation_outcomes": [
    {"timepoint": 14.0, "timepoint_unit": "days", "mass_loss_percent": 12.4, "carbonyl_index_initial": 0.21,
     "carbonyl_index_final": 0.35, "measurement_method": "gravimetric", "statistics": "mean ± SD, n = 3"}
  ],
  "mineralization": {},
  "mechanism": {"entries": {"enzymatic_hydrolysis": {"status": "suggested", "basis": "proposed_by_authors",
                                                     "evidence": "Terephthalic acid was detected by HPLC", "page": 2}}},
  "degradation_products": [{"product_name": "terephthalic acid", "product_type": "monomer",
                            "formation_observed": true, "analytical_method": "HPLC"}],
  "evidence_quality": {"evidence_level": "moderate", "rule_based_level": "strong",
    "rationale": "FTIR shows carbonyl increase and HPLC detects terephthalic acid; no CO2 evolution was measured, so mineralization is not shown.",
    "assessment_label": "AI-assigned evidence assessment. This is a model judgement of the evidence reported in the paper, not an objective scientific determination."},
  "source_evidence": [
    {"field": "experimental_conditions.temperature", "value": 30.0, "unit": "°C",
     "source": {"page": 2, "section": "Materials and Methods", "quote": "incubated at 30 °C for 14 days"},
     "confidence": "high", "basis": "directly_measured", "verified": true, "origin": "ai_extracted"}
  ],
  "validation": {"valid": true, "warnings": [{"code": "evidence_level_disagreement", "severity": "info",
    "message": "AI assessment 'moderate' differs from the rule-based assessment 'strong'. Review recommended."}],
    "corrections": []}
}
```

Note that mass loss is recorded, but `mineralization` stays empty because the paper reports no
CO₂, O₂ or TOC data.

## Development

Run the stack in Docker as above, or run pieces locally:

```bash
# Backend (Python 3.12, PostgreSQL running locally, Tesseract installed)
cd backend
python -m venv .venv && . .venv/bin/activate
pip install -r requirements-dev.txt
export DATABASE_URL=postgresql+psycopg://mpai:mpai@localhost:5432/mpai DATA_DIR=../data \
       OLLAMA_BASE_URL=http://localhost:11434 EMBEDDED_WORKER=true LOG_JSON=false
alembic upgrade head
uvicorn app.main:app --reload --port 8000

# Frontend
cd frontend
npm install
BACKEND_INTERNAL_URL=http://localhost:8000 npm run dev
```

With `EMBEDDED_WORKER=true` the API process also runs the extraction worker (convenient locally;
in Docker the worker is a separate service).

Project layout:

```text
backend/app/
  main.py, config.py, worker.py, database.py
  api/          papers, extraction, experiments, exports (+results), settings, health
  models/       SQLAlchemy models (typed columns generated from the schema)
  schemas/      extraction.py (the dataset schema), api.py (request bodies)
  services/     pdf_service, table_service, doi_service, url_service, paper_service,
                retrieval, ollama_service, llm/ (provider abstraction), extraction_service,
                validation_service, storage_service, review_service, results_service,
                export_service, job_service
  prompts/      system_prompt.txt, roster_prompt.txt, extraction_prompt.txt, validation_prompt.txt
  utils/        logging, security, errors, json_repair, values
backend/tests/  pytest suite (synthetic PDFs generated on the fly)
database/migrations/  Alembic environment and versions
frontend/       Next.js app (app/, components/, lib/, types/)
deploy/         Caddyfile
```

**Adding a cloud LLM provider.** Implement `LLMProvider` (`generate_json`, `health`) in
`backend/app/services/llm/`, register it in `registry.py` (`openai`, `gemini` and `grok` names are
reserved), and set `LLM_PROVIDER`. The extraction pipeline does not change.

**Schema changes.** Edit `backend/app/schemas/extraction.py`, then
`alembic revision --autogenerate -m "..."` and `alembic upgrade head`.

## Testing

```bash
# inside Docker (SQLite by default)
docker compose build --build-arg INSTALL_DEV=true backend
docker compose run --rm backend test

# against PostgreSQL
docker compose run --rm -e TEST_DATABASE_URL=postgresql+psycopg://mpai:<password>@postgres:5432/mpai_test backend test
# (create the mpai_test database first: docker compose exec postgres createdb -U mpai mpai_test)

# locally
cd backend && pytest -q
cd frontend && npm run typecheck && npm run build
```

The suite (60 tests) covers:

* **PDF** – normal paper (sections, metadata, DOI), multi-column reading order, tables, scanned PDF
  via OCR, OCR failure, corrupted and non-PDF files.
* **DOI** – valid DOI with open-access PDF, invalid/unknown DOI, inaccessible paper (login page
  instead of PDF), paywall status codes, landing-page metadata.
* **AI extraction** – missing values stay null, multiple experiments, conflicting values preserved,
  incorrect units flagged, hallucinated quotes downgraded, malformed JSON repaired or rejected,
  LLM review stored but not applied, prompt placeholders copied by the model ignored.
* **Validation** – invalid percentage, invalid pH, missing unit, unusual temperature,
  mineralization without carbon evidence, biodegradation from size reduction only, evidence levels,
  lossless value coercion.
* **API** – upload, duplicates, file type and size limits, filename sanitisation, full extraction
  run through the worker queue, results filtering/sorting, edit/accept/reject/rows/history,
  JSON/CSV/XLSX exports, DOI metadata-only flow, SSRF protection, model-not-installed failure,
  settings, deletion, re-runs that keep reviewed experiments.
* **Ollama provider and security** – request format, missing model, unreachable server, health,
  path traversal, SSRF guard, rate limiting, log redaction.

The model itself is replaced by a scripted provider in tests, so results are deterministic; real
extraction quality must be evaluated with your chosen model on papers you know well.

## Deployment

### Server requirements

Local LLMs need real compute. Rough guide for one extraction at a time:

| Setup | CPU / RAM | GPU | Model | Time per typical paper* |
|---|---|---|---|---|
| Minimum (CPU only) | 8 vCPU, 16 GB RAM | none | 7–8B, Q4 | 15–45 min |
| Recommended | 8 vCPU, 32 GB RAM | 8–12 GB VRAM (RTX 3060/4060 class) | 7–8B (or 14B with 12 GB) | 1–5 min |
| High quality | 16 vCPU, 64 GB RAM | 24–48 GB VRAM | 14–32B (70B needs ~48 GB) | 2–8 min |

\*10–20 chunks plus one review call per experiment; depends strongly on paper length, model and
hardware. Small free hosting tiers cannot run a local LLM reliably. Add ~10 GB disk for the
application and the size of each model you install.

### Microsoft Azure (one command)

`deploy/azure/azure.sh`, run from Azure Cloud Shell, creates an Ubuntu VM, installs Docker,
copies the project, generates the secrets, adds a website login, starts the production stack
with HTTPS on a free `*.cloudapp.azure.com` address and switches the VM off every night.
Step-by-step guide, costs and Azure for Students notes: **[DEPLOY_AZURE.md](DEPLOY_AZURE.md)**.

```bash
ACME_EMAIL=you@example.org ./deploy/azure/azure.sh up
```

### Production with HTTPS (any server)

```bash
# .env: DOMAIN=extractor.example.org, ACME_EMAIL=you@example.org, a strong POSTGRES_PASSWORD,
#       CORS_ORIGINS=https://extractor.example.org
# DNS A/AAAA record → server; ports 80 and 443 open
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
# with an NVIDIA GPU:
docker compose -f docker-compose.yml -f docker-compose.gpu.yml -f docker-compose.prod.yml up -d
```

The production override adds:

* **Caddy** reverse proxy with automatic HTTPS (Let's Encrypt, falling back to ZeroSSL) and HSTS;
  the frontend's own port is no longer published.
* **Optional login**: a Caddy `basic_auth` block placed in `deploy/auth/*.caddy` protects every
  page and API call (the Azure script creates it; by hand: `caddy hash-password`).
* **Resource limits** for every service (`OLLAMA_MEMORY_LIMIT` for the model server).
* **Nightly backups**: gzip `pg_dump` into `./backups`, kept `BACKUP_KEEP_DAYS` days.
  Restore with `gunzip -c backups/mpai_<date>.sql.gz | docker compose exec -T postgres psql -U mpai mpai`.
* `restart: unless-stopped`, rotated JSON-file logs (10 MB × 5) and persistent volumes for the
  database, models, PDFs and Caddy certificates.

Back up the `appdata` volume (PDFs and processed text) as well, for example
`docker run --rm -v microplastic-degradation-ai_appdata:/data -v "$PWD/backups":/b alpine tar czf /b/appdata.tgz -C /data .`.

### Security

* File type, magic-number and size validation; streamed uploads; sanitised filenames; files stored
  under their SHA-256 name; path-traversal protection on every file access.
* SSRF protection for DOI/URL downloads (private, loopback and link-local addresses refused, every
  redirect re-checked).
* Per-client rate limiting (stricter for ingestion), request validation, CORS allow-list, security
  headers, CSV/XLSX formula-injection neutralisation.
* Database credentials only in `.env`; secrets and connection-string passwords are redacted from
  logs. PostgreSQL and Ollama are never published.
* The app has no user accounts of its own. Never expose it to the public internet without the
  login in front: the production setup's Caddy `basic_auth` (automatic on Azure), a VPN, or an
  identity proxy.

### Logging

Logs are JSON lines on stdout (`docker compose logs -f backend worker`) with `timestamp`,
`level`, `paper_id`, `extraction_id`, `processing_stage`, `model`, `processing_time` and `error`
where relevant. Set `LOG_JSON=false` for human-readable logs.

## Limitations

* Output quality is bounded by the local model. Small models miss values, merge experiments or
  misattribute shared conditions; the review step is essential, not optional.
* Values that appear only in figures (plots) are not read; only text, captions and tables are.
* Complex tables (merged cells, rotated pages) may extract imperfectly; scanned tables depend on
  OCR quality.
* Section detection uses headings and font cues; unusual layouts fall back to relevance scoring
  over the whole text.
* Supporting-information files must be uploaded as separate PDFs.
* Numbers are stored as floating-point values; trailing zeros of the reported precision
  (e.g. `8.0` vs `8`) are not preserved in numeric fields, though the verbatim quote is.
* Open-access retrieval depends on Unpaywall/OpenAlex/Europe PMC coverage. Paywalled papers must be
  uploaded manually — by design.
* Rate limiting is in-memory per process; use the reverse proxy for stronger limits on multi-node
  deployments.

## Scientific considerations

* **Fragmentation ≠ degradation.** Particle-size reduction and surface erosion are recorded as
  physical changes. A record claiming biodegradation whose only evidence is size reduction is
  flagged (`biodegradation_from_size_reduction`).
* **Mass loss ≠ mineralization.** Mineralization requires CO₂ evolution, O₂ consumption or TOC/DOC
  data. Records claiming mineralization without them are flagged
  (`mineralization_without_carbon_evidence`, `mass_loss_recorded_as_mineralization`).
* **Surface oxidation ≠ biodegradation.** A carbonyl-index increase is recorded as chemical change;
  the evidence level only reaches *strong* when several techniques directly demonstrate polymer
  degradation and/or mineralization.
* **Measured vs interpreted.** Every fact has a basis (*directly measured*, *author
  interpretation*, *AI inference*); every mechanism is labelled *directly demonstrated*, *proposed
  by authors* or *AI inference*.
* **Evidence level** is an AI-assigned assessment, cross-checked by a deterministic rule set.
  **Confidence** is the model's confidence in its extraction, not experimental certainty.
* **Contradictions are preserved.** When a paper states different values for the same quantity,
  the higher-confidence value fills the field and all values are kept under `contradictions`.
* **Controls, replicates and statistics** are extracted into the experimental design and per
  measurement, so effect sizes can be judged against controls.

## License

MIT — see [LICENSE](LICENSE). Third-party components (Ollama, models, PyMuPDF under AGPL-3.0,
Tesseract, etc.) are subject to their own licenses; model licenses in particular vary.
