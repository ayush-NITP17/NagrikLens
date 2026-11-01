# NagrikLens V10 — Phase 2 Implementation Report

**Package:** `NagrikLens_V10_Phase2.zip`  
**Phase 2 status:** Implemented and locally tested; not production-certified.  
**Baseline supplied:** `NagrikLens_V10_Phase1_WIP.zip`

## 1. Input inspection and baseline

The supplied archive did **not** contain `PHASE_1_REPORT.md`. It contained `WIP_STATUS.md`, which explicitly labels the archive as an incomplete Phase 1 snapshot and identifies missing PDF page-selection/OCR, retrieval, grounding wiring, API/schema/migration, and Phase 1 test work. I did not assume a missing report existed or treat its stated plan as proof of completion.

The starting project contained the V9 application code and its prior regression tests. The pre-change baseline run recorded **116/116 legacy tests passing**. The Phase 2 working tree retains those tests and their expected behaviour.

Because the requested Phase 1 report was absent, this Phase 2 package completes several prerequisite Phase 1 work items where needed by the Phase 2 API: page-scoped PDF processing, OCR fallback and reports, compound/multi-page retrieval facets, and wiring of the existing grounding/privacy helpers into grounded synthesis. This does not assert that every item originally intended for Phase 1 is finished.

## 2. Changes implemented

### 2.1 PDF and extraction foundation carried into Phase 2

- PDF ingestion supports `max_page`, `specific`, and `all` page modes.
- The default selected range is pages 1–10 (or the entire document when it has fewer than 10 pages).
- Specific pages are validated, deduplicated and processed in ascending order while retaining original PDF page numbers.
- The processing report records total/selected/extracted/OCR/unreadable/failed/not-processed pages, text length, chunk count, duration, indexing status and completeness.
- A selected subset of a larger document is reported as `PARTIAL` overall even when the selected scope was processed successfully. For example, processing pages 1–10 of a 100-page PDF leaves pages 11–100 explicitly listed as not processed.
- OCR fallback uses a bounded Tesseract subprocess. OCR availability, page limits and timeout are configurable. OCR pages are identified separately.
- Heading carry-forward and page-aware chunking preserve section context and page references.
- PDF upload provenance remains `USER_UPLOAD` / `USER_UPLOADED_PUBLISHER_UNVERIFIED` and is not represented as authenticated government material.

### 2.2 Website URL ingestion

- HTTPS HTML and direct-PDF URLs use the same validated fetch pipeline.
- HTML extraction captures the title, main content, headings/sections, paragraphs, lists and table rows within the configured extraction cap.
- Direct PDF URLs support the same PDF page-selection and completeness reporting.
- Linked PDFs are only fetched when `include_linked_pdfs=true`, are same-host only, and are bounded by the configured maximum. There is no unrestricted website crawler.
- Source URL, title, provenance and extracted evidence are stored through the existing document store.
- Re-fetching a source uses the existing version history/diff mechanism.
- Fetch failures and partial extraction are surfaced rather than marked complete.

### 2.3 SSRF and host validation

- Only exact configured or administrator-approved hostnames can be fetched.
- HTTPS is required; userinfo and non-standard ports are rejected.
- DNS answers are checked and non-global addresses are rejected.
- Redirects are handled manually and every redirect destination is revalidated for scheme, exact allowlist membership, DNS/public-IP safety and port.
- A rejected host is recorded as `PENDING` where possible, and the error names the hostname and explains that administrator approval is required. It is never auto-approved.
- Response sizes are bounded using both declared and streamed byte counts; timeouts and redirect counts are bounded.
- DNS validation reduces SSRF exposure but does not fully eliminate DNS-rebinding/time-of-check/time-of-use risks; network egress controls are still required for a public deployment.

### 2.4 Source registry and administrator approval

- Additive SQLite migration creates persistent registry and query-session tables and extends document metadata without replacing an existing V9 database.
- `GET /api/sources/registry` returns the effective registry dynamically from configured hosts, administrator decisions and ingested records.
- Registry records distinguish allowed/pending/rejected host state from fetched/ingested document state and document authentication status.
- `POST /api/admin/sources/approve` and `/api/admin/sources/reject` require a separate administrator key (`NAGRIKLENS_ADMIN_KEY`). If no admin key is configured, administration endpoints are disabled.
- Approval permits exact-host fetching only. It does not authenticate the document itself, certify its legal authority or prove that it is current.
- User-uploaded files appear separately as user uploads and do not create synthetic hostname entries.

### 2.5 Selected-document isolation and query sessions

- `/api/query` requires an explicit active `document_id`, an explicit `document_ids` list, or a previously selected session.
- Retrieval is scoped to those selected documents only.
- Multiple documents are included only when explicitly selected; this supports intentional comparison without silently mixing unrelated sources.
- Opaque query sessions persist selected document IDs for follow-ups, switching and continuation. They do not store full transcripts and are not authentication.
- Deleting a document also removes its evidence, analyses, version rows and references from query sessions.

### 2.6 Independent answer-language selection

- `QueryRequest.answer_language` accepts `same_as_question`, `en` or `hi`.
- An explicit answer language takes precedence over detected question language.
- Legacy `language` remains for compatibility.
- Tests cover English→English, English→Hindi, Hindi→English, Hindi→Hindi, Hinglish→Hindi and Hinglish→English.
- Source evidence identifiers and citations remain source-language references; only the explanation language is selected independently.

### 2.7 Eligibility API contract

- The V9 deterministic `MET` / `NOT_MET` / `UNKNOWN` engine, boolean expression handling, exceptions, normalisation and Hindi numeral support are retained.
- The eligibility endpoint now returns documented counts for conditions found/met/not met/unknown, missing profile information, evidence citations, analysis coverage, processing report and document provenance.
- Incomplete document processing or incomplete condition extraction is explicitly reflected in the result and status.
- The result does not claim official eligibility; `UNKNOWN` is distinct from `NOT_MET`.

### 2.8 Security and privacy changes

- Request-body limits are enforced for both declared and streamed POST bodies, including chunked requests.
- PDF upload and source downloads have size limits.
- A shared API key may protect demo API routes; admin operations have a separate key.
- A per-process request limiter remains a local-demo safeguard, not a distributed production rate limiter.
- SQL remains parameterised and frontend output uses the existing safe rendering approach.
- Profile values are not sent in grounded answer synthesis. Identifier redaction and prompt-injection evidence handling are wired into synthesis.
- `.env` and local databases are excluded from `.gitignore` / `.dockerignore`; `.env.example` contains no secret.
- Document deletion is exposed via API.

## 3. Preserved behaviour

The following V9 behaviours and safeguards were preserved and covered by the combined regression suite:

- Unicode-aware English/Hindi tokenisation.
- BM25/inverted-index retrieval and cached retrieval behaviour.
- Conservative abstention and retrieval-strength labels.
- Word-boundary-safe query expansion and negative-match controls.
- Claim grounding and numeric/comparator/semantic safeguards.
- Deterministic tri-state eligibility evaluation and exception handling.
- Document version history and change review flags.
- Provenance labels and the distinction between allowlisted hosts and authenticated documents.
- Existing API/UI response compatibility fields where applicable.

The existing single-file frontend was not redesigned in Phase 2; the API contract is prepared for the Phase 3 frontend work.

## 4. API endpoint inventory

There are **19 method/route operations across 18 OpenAPI paths**. Full field-by-field request/response contracts and JSON examples are in `docs/PHASE_2_API_CONTRACT.md`; the machine-readable schema is `docs/openapi_phase2.json`.

| Method | Endpoint | Request | Response / purpose |
|---|---|---|---|
| GET | `/` | none | Existing application HTML shell |
| GET | `/api/health` | none | Version, health and non-secret configuration indicators |
| GET | `/api/documents` | none | Document list and provenance/processing metadata |
| POST | `/api/documents/upload` | Multipart: `file`, `jurisdiction`, `page_mode`, `max_page_number`, `selected_pages` | Document ID, provenance, page selection and processing report |
| POST | `/api/documents/ingest-url` | `IngestURLRequest`: `url`, `title`, `jurisdiction`, `include_linked_pdfs`, `max_linked_pdfs`, page-selection fields | HTML/direct-PDF ingestion result, version and completeness metadata |
| GET | `/api/documents/{document_id}` | path ID | Document metadata and page-aware evidence; raw text omitted |
| GET | `/api/documents/{document_id}/processing-report` | path ID | Extraction completeness, selected pages and failures |
| GET | `/api/documents/{document_id}/versions` | path ID | Stored version history and diffs |
| POST | `/api/documents/{document_id}/analyse` | `AnalyseRequest`: `language` (`en`/`hi`) | Structured policy analysis and coverage; requires configured LLM |
| POST | `/api/documents/{document_id}/eligibility` | `EligibilityRequest`: `profile`, `answer_language` | Deterministic condition outcomes, counts, missing info, citations and coverage |
| DELETE | `/api/documents/{document_id}` | path ID | Deletes document, evidence, analyses, versions and session references |
| GET | `/api/sessions/new` | none | New opaque selected-source session ID |
| POST | `/api/query` | `QueryRequest`: `question`, `answer_language`, legacy `language`, `document_id`, `document_ids`, `session_id`, `profile`, `use_llm`, `require_live_sources` | Evidence-grounded answer, selected docs, citations, retrieval strength and warnings |
| GET | `/api/sources/registry` | none | Dynamic configured/admin-approved hosts and ingested-source records |
| POST | `/api/admin/sources/approve` | `AdminSourceRequest`: `hostname`, `authority`, `verification_status`; admin header | Exact hostname approval record; document remains unauthenticated |
| POST | `/api/admin/sources/reject` | `AdminRejectSourceRequest`: `hostname`, `reason`; admin header | Exact hostname rejection record |
| GET | `/api/metrics` | none | Ingested document and evidence counts |
| GET | `/api/changes` | none | Recent stored version-change records |
| GET | `/api/llm-status` | none | Non-secret configured model/provider indicators; not a live connectivity guarantee |

Shared request access is optionally protected by `X-Nagriklens-API-Key` when `NAGRIKLENS_API_KEY` is configured. Administrator POST routes require `X-Nagriklens-Admin-Key` and a configured `NAGRIKLENS_ADMIN_KEY`. The full contracts describe response schemas and error conventions.

## 5. Test results

### Automated suite

- **Passed:** 179 tests.
- **Failed:** 0.
- **Not run:** 0 within the local automated test suite.
- **Legacy test baseline:** 116/116 prior tests passed before Phase 2 work; they remain included in the final 179-test suite.
- **Phase 2-specific test functions:** 53, with parametrized cases included in the 179 collected tests.

Executed commands:

```text
pytest -q
179 passed

python -m compileall -q app scripts tests
Passed (no syntax/compilation errors)
```

The tests use local fixtures, generated PDFs, mocked HTTP responses and local DNS/redirect simulations. They cover HTML/direct-PDF ingestion, requested linked-PDF extraction, redirects, unapproved/private hosts, limits and timeout behaviour, refresh/versioning, source registry/admin approval, selected-document isolation, sessions, all six language combinations, eligibility output and compound multi-page evidence with original-page citations.

### OCR

- Tesseract 5.5.0 was available locally with `eng` and `hin` language packs.
- English and Hindi scanned-PDF OCR tests were executed successfully.
- A simulated OCR timeout was verified to produce a partial/failure report rather than a false complete status.
- OCR on other machines requires installation of Tesseract and the configured language packs; it is not bundled in the ZIP.

### Retrieval benchmark (synthetic corpus)

The supplied V9 baseline and Phase 2 were compared on the same small synthetic multi-page corpus and held-out question set.

| Held-out metric | V9 baseline | Phase 2 | Difference |
|---|---:|---:|---:|
| Questions | 26 | 26 | — |
| Answerable questions | 23 | 23 | — |
| Recall at context | 0.6522 | 0.7065 | +0.0543 |
| Full-recall rate | 0.6087 | 0.6522 | +0.0435 |
| Precision at context | 0.4210 | 0.4065 | −0.0145 |
| Hit at 1 | 0.5652 | 0.5652 | No change |
| Answerable questions wrongly abstained | 5 | 4 | 1 fewer |
| Unsupported questions correctly abstained | 3/3 | 3/3 | No change |

On the development split, recall at context moved from 0.5769 to 0.5962; precision moved from 0.2846 to 0.2718; hit@1 moved from 0.5385 to 0.6154. These are small synthetic diagnostic sets, not independent real-document benchmarks. The recall gain comes with a small precision decrease. Vocabulary-gap performance remains weak in the benchmark. Do not present these numbers as real-world answer accuracy or legal correctness.

### Results not established

| Check | Status | Reason |
|---|---|---|
| Real Gemini API request | **Needs credentials / not run** | No live provider call was made for this package. Mocked provider/error paths are tested. |
| Live government HTML ingestion | **Needs live websites / not run** | Website fetching was tested through local mocked responses; no live government site was contacted. |
| Live government direct-PDF ingestion | **Needs live websites / not run** | Direct PDF URL logic was exercised with local fixtures/mocks. |
| Production deployment/authentication review | **Not run** | This is a prototype with shared-key gates, not a full user/tenant identity system. |
| Real-document retrieval/answer accuracy | **Not established** | Benchmark fixtures are synthetic; no independently labelled government-document corpus was supplied. |
| Legal correctness / official eligibility | **Not established** | No legal professional or issuing authority validated generated answers or extracted rules. |

## 6. Known gaps and limitations

1. The supplied archive had no `PHASE_1_REPORT.md`; `WIP_STATUS.md` said Phase 1 was incomplete. Phase 2 completed prerequisite pieces used by this API but cannot verify absent claims from the missing report.
2. There is no per-user authentication, account ownership or tenant-level document isolation. Optional shared API/admin keys are suitable only for a controlled demo/local deployment.
3. Query sessions store selected document IDs only. They do not store full chat transcripts or implement deep conversational pronoun resolution.
4. Static JavaScript-rendered sites may not yield content; extraction is static HTML parsing. Use an official PDF when the webpage is rendered client-side.
5. Linked PDFs are opt-in, same-host and bounded; general crawling is intentionally not supported.
6. Host allowlisting is not document authentication. A domain ending in `.gov.in` is still labelled as a host-level signal, not proof that a fetched file is legally authentic, current or in force.
7. DNS validation cannot fully eliminate DNS-rebinding/time-of-check/time-of-use attacks. Public deployment requires an egress proxy/firewall and stronger network isolation.
8. Rate limiting is process-local and keyed by client host/path. It is not a distributed production control and can behave poorly behind proxies unless configured at a trusted gateway.
9. OCR requires external Tesseract binaries and language packs and is bounded by configured page/time limits.
10. Gemini availability depends on a valid locally configured key, model access, network and quota. The package contains no key.
11. The existing frontend is preserved, not redesigned in Phase 2. Phase 3 should implement the registry, page-selection, completeness, suggestions and answer-language UI against the documented API.
12. Retrieval performance on synthetic fixtures does not guarantee correctness on unseen or real government documents.

## 7. Reproduction

From the extracted project root:

```powershell
py -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements-dev.txt
Copy-Item .env.example .env
pytest -q
python -m compileall -q app scripts tests
```

For the local server:

```powershell
python -m uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
```

See `README.md`, `docs/PHASE_2_API_CONTRACT.md` and `docs/openapi_phase2.json` for setup and Phase 3 integration.

## 8. Release statement

Phase 2 is delivered as a locally tested backend/API package. The 179 automated tests pass, including independent website/registry/language/isolation/eligibility tests and local English/Hindi OCR tests. No live government-site or real Gemini API integration is claimed as verified. This release is a hackathon prototype, not a production-secure or legally authoritative government decision system.
