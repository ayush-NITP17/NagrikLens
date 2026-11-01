# NagrikLens V10: UI final changelog

This pass changed the existing frontend, `app/static/index.html`. It did not add a second frontend, and the backend code under `app/*.py` was not changed. Every original element ID, event handler and API call was kept.

The main user journey is now **Upload PDF / Enter website → active document (selected automatically) → ask → evidence-backed answer.**

## 1. Layout changes

| Area | Before | After |
|---|---|---|
| Header | Logo, server badge, version tag. A separate full-width "Demo access" card held the Gemini badge and the key input. | One compact row with the server status, the Gemini status, an **Access** pop-over (holding the optional demo key) and a refresh icon. The pop-over opens itself only when `/api/health` reports `auth_required`. |
| Introduction | Tall hero banner | Shorter banner with the requested heading and subtitle, plus a four-step indicator (Add a source → Select it → Ask → See evidence). |
| Ingestion | Cards 1 and 2 in a general grid | Two balanced side-by-side cards on desktop (the same height, checked by a test), stacked on mobile. The PDF picker is now a dropzone and also accepts drag and drop. The page-scope controls are visible from the start, and **Process** stays disabled until the server has inspected the PDF. |
| Allowed sources | Large permanent card 4 | A collapsed one-line disclosure directly below ingestion. See section 2. |
| Active document + question | Cards 3, 6 and 7, below the library, registry and report | One workspace directly below the disclosure. An active-document bar shows the name, origin, extraction summary and a COMPLETE / PARTIAL SCOPE badge, with the switcher (`#selectedDoc`). Below it are the question form and the example questions, then the answer and evidence. |
| Examples | A full-width grid of 9 groups | Category tabs beside the question box (stacked below it on narrower screens). All 20 original examples are kept, and clicking one fills the question box. |
| Answer | A single column | The answer and its warnings sit beside the evidence column on wide screens and stack on narrow ones. The evidence header shows the retrieval status ("N supporting passages · retrieval strong"). Each citation shows an "Original PDF page N" chip, the quote, the source URL and an "Open original source" link for HTTPS sources. "What the system did" is now a collapsible panel. |
| Secondary | Cards 5, 3, 8, 9 and 10 spread across the page | A "Document tools" grid: processing report, document library, full-document analysis, eligibility check, and version comparison (full width). |

## 2. Collapsible source registry

- The trigger is a `<button id="registryToggle">` with `aria-expanded` and `aria-controls="registryPanel"`. It shows the heading "Currently Allowed Government Sources", the line "View the government websites currently approved for document ingestion.", and a **View sources ⌄** / **Hide sources ⌃** chip. The chip includes the host count from the backend response.
- The panel is **collapsed on load**. It opens and closes **only** when the user clicks it or uses Enter / Space. Typing a URL, ingesting a source or having a URL rejected never opens it; tests cover each of these.
- The animation uses a `grid-template-rows: 0fr → 1fr` transition. While the panel is closed it also has `visibility:hidden`, so its contents cannot receive keyboard focus.
- The panel's contents still come from `GET /api/sources/registry`. No domain is hardcoded, and the existing test that checks `pib.gov.in` is absent still passes.
- The **Refresh** button stays visible in the collapsed row. The administrator approve/reject form is inside the panel and makes the same API calls with the same `X-Nagriklens-Admin-Key` header as before.
- When an unapproved host is rejected, the error shows the backend message with the exact hostname, then adds: "Open 'Currently Allowed Government Sources' below to see the approved hosts. NagrikLens never approves a host automatically."

## 3. Document-to-question workflow (JavaScript changes)

- `setActive(id)` now accepts only a document that is in the loaded list. When there is no valid document, it disables `#question`, `#askBtn` and the other tools, and shows a lock note.
- After a successful upload or URL ingestion, the new `document_id` becomes active. If the list refresh failed or raced, `ensureListed()` adds a minimal entry built from the successful ingestion response. The page then scrolls the workspace into view and focuses the question box.
- If ingestion fails, the active document does not change; a test covers this.
- Switching documents hides the previous answer, and the active-document bar briefly highlights the change. If the user switches while a query is still running, the late response is discarded so citations always match the active document.
- Ctrl/⌘ + Enter submits a question.
- The rendering helpers are the same text-node based ones as before (`appendText`, `textContent`). No source text is inserted as HTML.

## 4. Whitespace reductions

- Card padding went from 19px to 16px. Field gaps went from 11px to 8px, input padding from 10px to 7px, and the textarea minimum height from 100px to 84px.
- The hero padding went from 30px to 18px, and its heading from a 38px maximum to a 23px maximum.
- The full-width Demo access card was removed (folded into the header), and a lot of repeated explanatory text was shortened.
- On a 1440×900 screen, the upload card, the website card, the sources disclosure and the start of the question workspace are visible without scrolling.

## 5. Visual effects (all CSS, no new dependencies)

- A layered background of soft radial and linear gradients, refined card shadows, and a slight lift on hover.
- Button hover and press transitions. The primary buttons now have small inline SVG icons from a sprite in the page; no icon library was added.
- The animated source disclosure with a rotating chevron.
- A gentle highlight and fade-in when a document becomes active.
- A fade-up when the answer panel appears, and a shimmer loading card while the backend retrieves evidence.
- An indeterminate progress bar during PDF and website ingestion. This is honest: the backend reports no partial progress, so the bar does not claim a percentage.
- A green success message with an animated check mark after ingestion. Partial page scope is still stated in the message text.
- Hover motion on the example-question chips and status badges that change colour smoothly.
- `prefers-reduced-motion: reduce` turns off all transitions and animations and stops the hover lift; a test covers this.
- Responsive breakpoints at 1020, 860 and 560px. A test confirms there is no horizontal overflow at 1440, 1024, 820 and 390px.

## 6. Tests executed

All runs below were done in this environment: Python 3.13, the Chromium found on `PATH`, Node.js, and Tesseract with `hin` data.

| Check | Result |
|---|---|
| Baseline run of the original V10 suite before any change | 193 passed, **1 failed**. The failure was `test_hindi_scanned_pdf_uses_hindi_ocr…`, which needs the system font `/usr/share/fonts/truetype/noto/NotoSansDevanagari-Regular.ttf`. After installing `fonts-noto-core` in the test environment it passes; no code was changed for it. |
| `python -m compileall -q app scripts tests` | OK |
| Frontend JavaScript syntax (`node --check`, an existing test) | Passed |
| Full suite `pytest -q` (working copy) | **209 passed, 0 failed, 0 skipped** |
| Full suite from a clean extraction of the final ZIP | **209 passed** (see section 7) |

**Two existing tests needed a one-line edit, with nothing removed.** `test_frontend_dynamic_source_registry_and_document_switch` and `test_frontend_admin_approval_and_version_history_use_real_api_contract` assumed the registry was always open. Both now click `#registryToggle` first, as this assignment requires, and every original assertion is unchanged.

**New tests (`tests/test_v10_ui_final.py`, 15 tests):**

1. The registry is collapsed by default: `aria-expanded=false`, the list is hidden, the row is under 90px tall, and the data came from the API.
2. The registry opens on click and shows backend data.
3. The registry closes on a second click and works with the keyboard (Enter and Space).
4. Typing a URL or getting a rejection does not expand the registry; the rejected hostname is shown and no admin call is made.
5. The question panel unlocks after a successful upload, the new upload becomes active, and the card order is correct.
6. A website ingestion becomes the active document.
7. The question panel is unavailable when no document is active.
8. A failed upload does not make anything active.
9. The page-selection inputs work: the default is 10, the maximum equals the total page count, specific and entire modes work, and out-of-range values are rejected.
10. The answer language and document switching send the correct `document_id` and `answer_language` (Hindi question with English answer, and English question with Hindi answer).
11. The example questions fill the input across the category tabs, and all 20 are present.
12. Evidence citations render safely as plain text (XSS probe), with page numbers.
13. There is no horizontal overflow at four widths, and the ingestion cards align on desktop and stack on mobile.
14. Reduced motion is respected.
15. **A real-backend end-to-end test.** It starts a live Uvicorn server and a Chromium browser, uploads a generated 100-page PDF (the default of 10 is shown, the user changes it to 12), checks the PDF becomes active automatically, asks a question, and gets the answer with "Original PDF page" citations through real BM25 retrieval. It then uploads a second PDF, switches back and forth, and checks that each query's `selected_document_ids` and citations match only the active document. It also checks that a Hindi question with English requested returns `answer_language: "en"`.

## 7. Packaging check

`NagrikLens_V10_UI_Final.zip` was extracted into an empty directory. In that directory, `python -m compileall`, `pytest -q` and the README's `uvicorn app.main:app` start command were run, and `/` and `/api/health` returned 200. The ZIP contains no `.env`, no SQLite database, no `__pycache__` and no API key.

## 8. Known limitations (honest disclosure)

- **The Gemini answer language could not be tested live** (Scenario 5). No API key was available, and the release checks make no network LLM calls. The frontend sends `answer_language` correctly, and the extractive (no-LLM) backend path reports the requested language. Whether a *Gemini-generated* answer is actually written in Hindi or English still has to be checked by hand with a configured key.
- **Real government websites were not fetched** (Scenario 2 against live `.gov.in` hosts). The sandbox's network allowlist blocks them. Website ingestion was tested through the existing mocked backend tests and the mocked browser flow.
- **Upload progress is indeterminate.** The API returns a single response, so no true percentage is available.
- The "Use" button, the suggestion clicks and the scroll after ingestion use smooth scrolling. Smooth scrolling is turned off under reduced motion.
- The layout was checked with Chromium only (desktop 1440, tablet 820 and mobile 390, using screenshots and automated overflow checks). It was not checked in Safari or Firefox. The radio-pill styling uses `:has()`; in older browsers without it, the radios still work but the selected pill is not highlighted.
- The Hindi OCR test depends on a system Devanagari font being installed, as it did before this pass.
