# NagrikLens V10 — Evidence-Grounded Government Document Intelligence

NagrikLens is a citizen-facing prototype for understanding government PDFs and approved HTTPS sources. It lets a user select a document, choose PDF pages, ask questions in English or Hindi, inspect original evidence citations, and review extracted eligibility conditions.

> **Important:** NagrikLens is not a government portal, legal adviser, official eligibility decision-maker, or production service for sensitive citizen data. A hostname being allowlisted does not authenticate an individual document or prove legal authority. Automated grounding is heuristic, not formal legal entailment.

## V10 UI final pass

The single existing frontend (`app/static/index.html`) was reorganised around the citizen workflow: **Upload PDF / Enter website → active document → ask → evidence-backed answer**.

- Compact header (server and Gemini status, optional demo-access key) and a shorter introduction.
- PDF upload and website ingestion side by side on desktop, stacked on mobile.
- **Currently Allowed Government Sources** is a collapsed disclosure directly below ingestion. It opens only when clicked, loads from `/api/sources/registry`, keeps its Refresh button and the administrator approve/reject controls.
- The active-document bar and the question box sit directly below ingestion. A successfully ingested PDF or website becomes the active document automatically; the question box stays disabled until a document is active.
- Example questions are grouped into category tabs beside the question box; the answer, retrieval status, warnings and page-level citations appear immediately below.
- Processing report, document library, full-document analysis, eligibility check and version comparison are in a compact "Document tools" grid.

See `UI_FINAL_CHANGELOG.md` for details and test results.

## V10 Phase 3 highlights

- Reworked the existing `app/static/index.html`; no second frontend was introduced.
- PDF metadata preview detects total pages before processing. The UI shows the server's configured upload-size and page-processing limits.
- PDF selection offers first N pages (default 10), specific pages, or the entire PDF. Original page numbers are retained in citations.
- Direct PDF URL ingestion has its own page-scope controls; HTML page extraction is not incorrectly treated as PDF pages.
- Website ingestion, document selector, processing reports, source registry, admin source approval, suggested questions, independent answer-language choice, evidence panel, structured analysis, eligibility form and version history are connected to the Phase 2 API.
- Status labels are conservative: the UI never marks a source COMPLETE unless the backend reports complete extraction and successful indexing.
- Citation and source text are inserted as text nodes; the frontend does not treat source content as HTML.
- The LLM status endpoint reports the model loaded by the backend configuration without returning a key.

## Preserved V9 / Phase 2 behaviour

V10 retains the existing BM25/inverted-index retrieval, multilingual query expansion, abstention, evidence selection, grounding validation, Gemini request handling, selected-document isolation, source allowlist/SSRF checks, PDF page-selection/OCR pipeline, version history, source registry, and deterministic tri-state eligibility engine. The original regression suite remains part of the full test run.

## Requirements

- Python 3.10 or newer. Use a Python version supported by all binary dependencies in your environment.
- Dependencies in `requirements.txt`.
- For tests and browser integration: `requirements-dev.txt`, Node.js, and a Chromium browser supported by Playwright.
- Optional for scanned PDFs: Tesseract OCR executable plus English/Hindi language packs (`eng+hin`).
- Optional for generated answers and structured LLM extraction: a valid Gemini API key, project/model access, network connectivity and available quota.

## Windows / VS Code installation

1. Download and extract `NagrikLens_Forge_2026_V10.zip`.
2. In VS Code, select **File → Open Folder** and open the extracted project root. The root must contain `requirements.txt`, `.env.example`, `app/` and `tests/`.
3. Open **Terminal → New Terminal** and create a virtual environment:

```powershell
py -m venv .venv
```

4. Activate it:

```powershell
.\.venv\Scripts\Activate.ps1
```

If PowerShell blocks activation, allow scripts for the current terminal only, then activate again:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\.venv\Scripts\Activate.ps1
```

5. Install development and runtime dependencies:

```powershell
python -m pip install --upgrade pip
pip install -r requirements-dev.txt
```

6. Create your private local environment file:

```powershell
Copy-Item .env.example .env
```

7. Open `.env` in VS Code. Put your newly created Gemini API key in `LLM_API_KEY`. The example file intentionally has an empty key. **Never paste a key into this README, source code, browser, screenshots, Git repository or submission ZIP.**

The current default model is `gemini-3.8-flash` using Google's OpenAI-compatible Chat Completions endpoint. It can be changed in `.env` if your Google project does not have access to the configured model. See [Google's OpenAI compatibility documentation](https://ai.google.dev/gemini-api/docs/openai) and [model list](https://ai.google.dev/gemini-api/docs/models).

8. Run the complete test suite:

```powershell
pytest -q
```

9. Optional: install the Playwright-managed Chromium browser if your machine does not already have a compatible Chromium installation:

```powershell
python -m playwright install chromium
```

10. Run the application:

```powershell
python -m uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
```

11. Open <http://127.0.0.1:8000> in your browser. Interactive API docs are available at <http://127.0.0.1:8000/docs>.

## Environment settings

The `.env.example` contains non-secret defaults for:

- `LLM_API_KEY`, `LLM_BASE_URL`, `LLM_MODEL`, `LLM_TIMEOUT_SECONDS` — Gemini configuration.
- `ALLOWED_SOURCE_HOSTS` — additional exact HTTPS hostnames. Built-in exact hosts are also defined in `app/config.py`.
- `MAX_UPLOAD_BYTES`, `MAX_SOURCE_BYTES`, `MAX_HTML_CHARS` — input limits.
- `DEFAULT_MAX_PAGES`, `MAX_PDF_PAGES`, `MAX_PROCESS_PAGES` — PDF page handling limits.
- `OCR_ENABLED`, `OCR_LANGUAGES`, `MAX_OCR_PAGES`, `OCR_DPI`, `OCR_PAGE_TIMEOUT_SECONDS` — optional OCR controls.
- `NAGRIKLENS_API_KEY` — optional shared demo access gate (not user authentication).
- `NAGRIKLENS_ADMIN_KEY` — separate administrator key for source approval/rejection.

When `NAGRIKLENS_API_KEY` is configured, the browser asks for that demo access key and sends `X-Nagriklens-API-Key` to protected API routes. The administrator actions additionally send `X-Nagriklens-Admin-Key`. Do not put either value in the frontend source.

## PDF page selection

When a PDF is selected, the frontend calls `POST /api/documents/inspect-pdf` to inspect page count and configured size/page limits without storing the PDF. The default page maximum is 10. The user can increase it up to the server's configured `MAX_PROCESS_PAGES`, select individual original page numbers such as `2, 8, 10, 7`, or choose the entire PDF. The UI blocks selections over the one-request processing limit and reports when pages outside a selected subset were not processed.

The upload-size limit is read from the backend response; the UI does not hardcode a potentially incorrect MB value.

## Website sources and trust

Direct server-side URL ingestion requires HTTPS and an exact approved hostname. Redirects are validated individually. Unknown hosts are rejected and require administrator approval; they are never automatically trusted because a user entered a URL. The registry UI displays the backend's effective source registry rather than a hardcoded list.

An approved host is only a fetch permission. It does not independently authenticate each document, verify that the issuing authority is correct, or decide which legal version is in force. User-uploaded PDFs retain user-upload provenance and are not represented as authenticated official publications.

HTML parsing is static; JavaScript-rendered content may not be extracted. Linked PDFs are bounded and opt-in. There is no unrestricted crawler.

## Tests and verification

```powershell
python -m compileall -q app scripts tests
pytest -q
```

The V10 UI final release ran **209 automated tests successfully** (the original 194 plus 15 in `tests/test_v10_ui_final.py`, including a real-backend Chromium journey). The Phase 3 release environment previously ran 194. The 15 Phase 3-specific tests include JavaScript syntax validation, Chromium browser integration using mocked API responses, PDF metadata preview, UI page selection, document switching, language selection, suggestion clicks, citations, eligibility rendering, admin headers, version history, configured LLM status and safe rendering of hostile citation text.

The Phase 2/website tests use mocks and local fixtures; they do not prove that external government websites are currently reachable. Gemini network calls were not made as part of the release checks. See `PHASE_3_REPORT.md` for the precise passed/not-run breakdown and known limitations.

## Troubleshooting

| Symptom | Check |
|---|---|
| `py` is not recognized | Install Python and enable the Windows Python Launcher, or use `python -m venv .venv`. |
| `Activate.ps1` is blocked | Run the process-scoped execution-policy command above. |
| `No module named fastapi` | Confirm `(.venv)` appears in the terminal, then `pip install -r requirements-dev.txt`. |
| PDF preview says invalid PDF | Use an unlocked, readable PDF with a `.pdf` extension and a valid `%PDF-` signature. |
| PDF is too large | Check `MAX_UPLOAD_BYTES` in `.env`. |
| Page selection rejected | Check the PDF's actual page count and `MAX_PROCESS_PAGES`; select fewer pages or increase the server setting within the configured limits. |
| Scanned pages are unreadable | Install Tesseract and the required `eng` / `hin` language packs; check OCR settings in `.env`. |
| Gemini is not configured | Check the private `.env` file and restart Uvicorn. The server's `/api/llm-status` endpoint does not test a real request. |
| Gemini returns model/quota/auth error | Verify key validity, project model access, billing/free-tier quota and configured model ID in Google AI Studio. |
| Source host rejected | Check the exact hostname in the registry. Administrator approval is required; do not add wildcards. |
| Port 8000 is occupied | Run Uvicorn with `--port 8001` and open `http://127.0.0.1:8001`. |

## Documentation

- `PHASE_2_REPORT.md` — original Phase 2 report, preserved.
- `PHASE_3_REPORT.md` — Phase 3 changes, exact verification results and limitations.
- `PHASE_2_API_CONTRACT.md` and `docs/PHASE_3_FRONTEND_API_CONTRACT.md` — frontend/API contract, including PDF preview.
- `docs/openapi_v10.json` — generated OpenAPI schema.
- `docs/ARCHITECTURE_V10.md` — system architecture and data flow.
- `docs/MODEL_CARD_V10.md` — system purpose, intended use, evaluation and limitations.
- `docs/SECURITY_AND_PRIVACY_V10.md` — security controls and limitations.
- `docs/TEST_REPORT_V10.md` — test inventory and results.
- `docs/DEMO_SCRIPT_V10.md` — repeatable demonstration sequence.
- `docs/KNOWN_LIMITATIONS_V10.md` — known unverified or incomplete areas.
- `PHASE_1_REPORT_STATUS.md` — records the missing Phase 1 report and the available WIP status source.
