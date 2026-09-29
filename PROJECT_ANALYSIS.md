# Rusty — Project Analysis

> Codebase review as of commit `a784b98` (2026-09-29). Covers architecture, how each feature actually works end-to-end, and a prioritised list of gaps between the code and the rules in [CLAUDE.md](CLAUDE.md). Findings reference `file:line` so they can be verified directly.

---

## 1. What the system is

Rusty is a closed-access, mobile-first PWA that lets KHEL Foundation students (Classes 5–10, Bihar Board) ask questions about their textbooks and take practice tests. Answers are produced by a RAG pipeline restricted to chunks of admin-uploaded PDFs for the student's class and subject.

Three roles exist in practice (plus `coordinator`, which the backend treats like a teacher):

| Role | What they can do |
|---|---|
| **Student** | Study chat (Q&A with textbook citations), chapter summaries, AI-generated 5-question practice tests (resume / delete), take teacher-published tests, view own progress |
| **Teacher / Coordinator** | Generate quiz packs (5 MCQ + 2 short answer), edit & publish them as tests for a class, "study as Class N", view class dashboard, reset a student's PIN, view RAG dashboard |
| **Admin** | Upload/delete textbook PDFs (triggers ingestion), RAG observability + eval results, user management (**frontend-only — see §6**) |

---

## 2. Repository layout

```
Rusty-AI-Study-Mentor/
├── backend/                 FastAPI app (Python 3.11, async SQLAlchemy)
│   ├── app/
│   │   ├── main.py          App factory, middleware, lifespan (create_all + TTL purges)
│   │   ├── api/             Routers: auth, study, test, quiz, teacher, student, admin, health
│   │   ├── core/            config (pydantic-settings), database, deps (auth/RBAC), firebase, limiter
│   │   ├── middleware/      security headers, request logging, safeguarding (tier1/tier2)
│   │   ├── models/          SQLAlchemy ORM (8 tables)
│   │   ├── repositories/    Data access (chunk, cache, test, attempt, score, summary, trace)
│   │   ├── schemas/         Pydantic request/response DTOs
│   │   └── services/        RetrievalService (RAG orchestration), circuit breaker, token budget, RAG trace
│   ├── eval/                Offline RAG eval harness + datasets + saved results (read by admin UI)
│   ├── migrations/          Alembic 0001–0003 (out of sync with models — see §6)
│   ├── rag, safeguarding    ⚠ symlinks to an absolute macOS path (broken on Windows/Linux)
│   └── tests/               9 pytest modules (unit-level only)
├── rag/                     Shared RAG package (imported by backend as `rag.*`)
│   ├── ingestion/           pdf_extractor → chunker → metadata_tagger → embedder → pipeline; summary_generator; cli
│   ├── retrieval/           pre_filter, rrf, prompt_builder, llm_client (Ollama|Vertex), gemini_client (dead code)
│   └── data/maths-08.pdf    Sample textbook
├── safeguarding/            phrases_v1.json (40 tier-1 phrases), test_cases.json (10 cases)
├── frontend/                React 18 + Vite + Tailwind PWA, MSW mocks in dev
├── zap/                     OWASP ZAP config + auth hook
├── docker-compose*.yml      pgvector PG16 + Firebase emulators; ZAP profiles
├── .github/workflows/ci.yml truffleHog, pip-audit, pytest, npm audit, tsc, vitest, build
└── test_api.sh, test_rag_e2e.py  Manual smoke scripts
```

Size: ~5.6k lines of Python (app + rag) and ~5.4k lines of TSX components.

---

## 3. Architecture

```
 Browser (PWA, React)                           Backend (FastAPI)                               Stores
 ────────────────────                           ─────────────────                               ──────
 LoginScreen ──POST /api/auth/login──► Vite proxy ─► auth.login ──────────────────────────────► Firestore users (pin_hash)
                                                     │  dev: hardcoded creds → "dev-*-token"
 StudyMode ──POST /study/query/stream (SSE)────────► study.study_query_stream
                                                     ├ allowlist subject, tier1_scan(query+history) → 451
                                                     ├ summary intent? → chapter_summaries (pre-computed)
                                                     └ RetrievalService.retrieve_streaming
                                                         embed(query) ───────────────────────► Ollama / Vertex
                                                         response_cache (cosine < 0.05) ─────► Postgres/pgvector
                                                         vector_search + bm25_search ────────► chunks (WHERE class_num, subject[, chapter, language])
                                                         RRF top-5 → build_prompt
                                                         llm_breaker → generate_structured ──► Ollama qwen2.5:3b / Gemini
                                                         ground-check key points, store cache, persist RAGTrace ► rag_traces
 TestMode ──/test/generate, /test/answer, /test/result, /test/resume, /test/my-tests──► tests, questions, test_attempts, attempt_answers
 QuizPack/CreateTest ──/quiz/generate, /quiz/publish──► tests/questions (teacher-owned)
 AssignedTests/TakePublishedTest ──/quiz/assigned──► (graded client-side, never persisted)
 PdfUpload ──/admin/upload-pdf──► BackgroundTask: ingest_pdf → chunks → generate_chapter_summaries (EN+HI)
 RAGDashboard ──/admin/rag/*, /admin/eval/*──► rag_traces, backend/eval/results/*.json
```

### Middleware order ([main.py](backend/app/main.py))
Security headers → request logging (hashes IP, binds `trace_id`) → CORS allowlist (`GET/POST/DELETE`, `Authorization`/`Content-Type`). Rate limiting uses the shared `slowapi` singleton in [limiter.py](backend/app/core/limiter.py) keyed by remote IP. Safeguarding is **not** middleware; each route calls `tier1_scan` explicitly. The module is imported at startup, so a missing phrases file prevents boot.

### Auth & RBAC ([deps.py](backend/app/core/deps.py))
- `get_current_user` verifies a Firebase **ID token** (`verify_id_token(check_revoked=True)`), then loads `users/{uid}` from Firestore for `role`, `centre_id`, `class_num`, `is_active`.
- In `APP_ENV=development`, the literal tokens `dev-student-token` / `dev-teacher-token` / `dev-admin-token` short-circuit to hardcoded users (student is Class 8).
- `StudentDep` = any role, `TeacherDep` = teacher/coordinator/admin, `AdminDep` = admin only.
- Login ([auth.py](backend/app/api/auth.py)): 5/min rate limit, in-memory lockout (5 failures / 15 min, bounded to 10k IDs), bcrypt PIN check, returns a Firebase **custom token**.

---

## 4. Feature walkthroughs

### 4.1 Study (Q&A)
1. `StudyMode.tsx` builds `history` from prior turns (falls back to `key_points.join("; ")` when `notes` is empty) and calls `streamSSE("/study/query/stream")`.
2. Route validates subject against the allowlist and runs `tier1_scan` on the query **and every history turn**. A hit returns HTTP 451 with a "talk to a trusted adult" message, which the UI renders specially.
3. If a chapter is selected and the query matches summary keywords (EN/HI), the pre-computed `chapter_summaries` row is returned (EN fallback for HI).
4. Otherwise `RetrievalService.retrieve_streaming` runs these steps, emitting SSE `stage` events (`embedding → retrieving → retrieved → generating`) and then `result` + `done`:
   - `trim_history` to ~2000 tokens (char-ratio estimate, 4 for EN / 2 for HI).
   - `build_filter` re-validates class 1–10 and subject. **This is the architectural textbook boundary.**
   - Embed the query and check `response_cache`: same class/subject/chapter/medium, cosine distance < 0.05, within TTL, same `PROMPT_VERSION`.
   - Run vector search (pgvector cosine, top 10) and BM25 (`ts_rank_cd` over `to_tsvector('simple')`, top 10), both pre-filtered and language-filtered. If fusion is empty, retry without the language filter (`lang_fallback`).
   - Reciprocal Rank Fusion (k=60) keeps the top 5 chunks.
   - Prompt: fixed system prompt (EN or HI) + JSON spec, then history, then a user message with the passages, then an "ack" assistant message, then the student question.
   - The LLM call runs inside the circuit breaker (5 failures opens it for 30 s; half-open allows 2 probes). Output is forced to JSON (`format: json` on Ollama, `response_schema` on Vertex).
   - Ground check: key points and misconceptions are dropped unless ≥2 content words appear in the retrieved text. The result is cached and a `RAGTrace` row is persisted.

### 4.2 Practice tests (student)
- `/test/generate` (10/min): RAG in `test` mode with chunks shuffled for variety. The LLM returns 5 MCQs, which `_validate_mcq` filters (A–D answer, 4 distinct options, no negative phrasing). Question text is tier1-scanned. The route persists `Test` + `Question`s + a `TestAttempt`, and returns questions without answers.
- `/test/answer` grades one answer, reveals the correct option and explanation, and auto-submits once every question has an answer.
- `/test/result`, `/test/resume`, `/test/my-tests`, `DELETE /test/{id}` (owner-only).

### 4.3 Quiz packs and published tests (teacher → student)
- `/quiz/generate` (teacher, 20/min): 5 MCQ + 2 short answers plus a `plain_text` export. Output is tier1-scanned.
- `/quiz/publish` stores the (possibly teacher-edited) MCQs as a `Test` for a class.
- `/quiz/assigned` returns up to 20 active tests for a class **including correct answers**. `TakePublishedTest.tsx` grades in the browser and records completion only in `localStorage`.

### 4.4 Ingestion (admin)
`/admin/upload-pdf` checks the extension, the `%PDF-` magic bytes, 50 MB max, class 5–10 and the subject allowlist, and rejects duplicate `source_pdf` names. It creates a `Textbook(status=processing)` row and starts a background task:
1. `extract_pages` (PyMuPDF block-level extraction, superscripts rendered as `^n`, heuristic table→markdown, skips pages under 50 chars).
2. `chunk_pages`: splits at section boundaries (numbered sections, Example/Exercise/Definition…, EN+HI), headings and blank lines. Segments under 300 chars are merged, anything over 2000 chars is split, and anything under 80 chars is dropped. Chapter comes from "Chapter N"/"अध्याय N" patterns.
3. `tag_chunk`: section type (always `explanation` in practice), a keyword-based `math_type`, and `$…$` LaTeX extraction. A contextual header (`Class 8 | Mathematics | Chapter… | Page N`) is prepended before embedding.
4. Embeds each chunk sequentially through the Ollama HTTP API, then deletes and re-inserts chunks for that `source_pdf`.
5. Marks the textbook `ready` (or `failed` with the error), then generates EN + HI chapter summaries with Ollama (≤15 sampled chunks × 400 chars per chapter).

### 4.5 Observability & evaluation
- Every RAG call emits one structured `rag_trace` log line and a `rag_traces` row: timings per stage, hit counts, top scores, token usage, JSON validity, ground-check and MCQ-drop counts, a 150-char chunk preview, a 500-char LLM preview, and the **query text** (intentional, for safeguarding oversight). Rows older than 180 days are purged at startup.
- `RAGDashboard.tsx` shows summary, per-subject and timeline aggregates, recent traces, and eval runs.
- `backend/eval/run_eval.py` runs a JSON dataset against a live server and scores scope handling, chapter match, keyword coverage, exact match and latency. It pulls groundedness from the latest trace. The two committed runs on `maths08_squares` scored 8/11, then 10/11.

### 4.6 Student progress
`/student/progress` combines `quiz_scores` (never written, see §6) and submitted practice attempts (each hardcoded as out of 5). It returns an overall %, per-subject breakdown, timeline and a daily streak.

---

## 5. Data model (PostgreSQL + pgvector)

| Table | Purpose | Notes |
|---|---|---|
| `chunks` | Textbook chunks + `vector(1024)` embedding | Pre-filter index `(class_num, subject, language, chapter)` |
| `textbooks` | Upload registry + ingestion status | `source_pdf = class{N}_{subject}_{filename}` (unique) |
| `chapter_summaries` | Pre-computed EN/HI summaries | Unique `(class_num, subject, chapter, language)` |
| `response_cache` | Semantic answer cache | 168 h TTL, keyed by prompt version |
| `rag_traces` | Per-query telemetry incl. query text | 180-day TTL |
| `tests` / `questions` | AI practice tests and teacher-published tests | Same table for both; distinguished by `created_by` |
| `test_attempts` / `attempt_answers` | Per-student attempt + answers | Unique `(test_id, student_id)` |
| `quiz_scores` | Intended score history | **No code writes to it** |

Firestore holds `users/{student_id}` (role, class, centre, `pin_hash`, `is_active`) and is meant to hold `safeguarding_flags`.

---

## 6. Findings

Severity reflects impact on the project's own non-negotiables (CLAUDE.md) and on production readiness. The local Ollama + dev-token path works (the eval results show this). Most issues surface in production mode or on a fresh checkout.

### 🔴 Critical — safeguarding rules

| # | Finding | Where |
|---|---|---|
| C1 | **The SSE study endpoint never scans LLM output.** `/study/query` scans `key_points + notes`, but `/study/query/stream`, the endpoint the UI actually uses, yields the result straight from `retrieve_streaming`. This violates "Never return an LLM response without first running the output safeguarding scan." | [retrieval.py:280](backend/app/services/retrieval.py#L280) |
| C2 | **Cache hits and chapter summaries are served without an output scan.** Cached responses (produced by the unscanned stream path) and LLM-generated summaries are returned directly on both study endpoints. | [retrieval.py:189](backend/app/services/retrieval.py#L189), [study.py:136](backend/app/api/study.py#L136), [study.py:195](backend/app/api/study.py#L195) |
| C3 | **Safeguarding flags and alerts never happen.** `tier1_flag_and_alert` (Firestore `safeguarding_flags` + alert) and `tier2_scan` are defined but never called. A tier-1 hit returns 451 and leaves no record for the centre lead. The email sender is also a log-only stub. | [safeguarding.py:74](backend/app/middleware/safeguarding.py#L74), [safeguarding.py:126](backend/app/middleware/safeguarding.py#L126) |
| C4 | **Tier-1 substring matching produces false positives on normal study questions.** `"rape"` matches *grape/scrape/drape*; `"abuse"` matches *drug abuse* chapters; `"कोई मुझे"` / `"koi mujhe"` match "someone explain this to me"; `"beaten"` matches *unbeaten*. Students get 451s on ordinary questions, and textbook-derived output gets 500s. `test_cases.json` has 10 cases; its own note requires 100 before the pilot. | [phrases_v1.json](safeguarding/phrases_v1.json), [safeguarding.py:60-71](backend/app/middleware/safeguarding.py#L60-L71) |

### 🔴 Critical — production path will not work

| # | Finding | Where |
|---|---|---|
| P1 | **Production login is broken end to end.** The backend returns a Firebase *custom* token. The frontend never imports the Firebase SDK (it's in `package.json`), so it never exchanges that for an *ID* token and sends the custom token as the Bearer. `verify_id_token` rejects custom tokens, so every call after login returns 401 outside dev. | [auth.py:91](backend/app/api/auth.py#L91), [deps.py:32-47](backend/app/core/deps.py#L32-L47), `LoginScreen.tsx:33-41` |
| P2 | **The Vertex LLM path would reject the prompt.** `build_prompt` emits `role: "assistant"` (the ack turn and history). `_vertex_generate` passes roles through unchanged, but Vertex only accepts `user`/`model`. Only the Ollama path remaps roles. | [prompt_builder.py:172](rag/retrieval/prompt_builder.py#L172), [llm_client.py:121](rag/retrieval/llm_client.py#L121) |
| P3 | **Embedding spaces would not match in production.** Ingestion always embeds with Ollama `snowflake-arctic-embed2`, while production queries embed with Vertex `text-multilingual-embedding-002`. Vector search would compare incompatible vectors. Summary generation is also Ollama-only. | [pipeline.py:81](rag/ingestion/pipeline.py#L81), [llm_client.py:243-247](rag/retrieval/llm_client.py) |
| P4 | **The Docker image cannot build.** `COPY ../rag/ ./rag/ 2>/dev/null \|\| true` is invalid (a path outside the build context, and shell syntax in COPY). `backend/safeguarding` is a dangling symlink. `pymupdf` (`fitz`) is missing from `requirements.txt`, so ingestion would fail anyway. | [Dockerfile:28](backend/Dockerfile#L28), [requirements.txt](backend/requirements.txt) |
| P5 | **Migrations are out of sync with the models.** On a fresh DB, `alembic upgrade head` (README step 4) fails at 0002 because `rag_traces` is only created by `create_all` at startup. Migrations never create `textbooks` or `chapter_summaries`, or `chunks.page_num`/`chunks.language`. `create_all` won't add columns to an existing `chunks` table, so an Alembic-built DB breaks ingestion and retrieval. The 0001 FTS index uses `'english'` but queries use `'simple'` (index unused), and the ivfflat index was built for `vector(768)`. | [0002:22](backend/migrations/versions/0002_production_hardening.py#L22), [0001:120](backend/migrations/versions/0001_initial_schema.py#L120) |
| P6 | **`backend/rag` and `backend/safeguarding` are symlinks to `/Users/tejasai…/Desktop/FFG/…`.** On any other machine (including this Windows checkout and Linux CI) they are broken or plain files. Running from `backend/` works only if `PYTHONPATH` includes the repo root and `PHRASES_FILE_PATH` points to `../safeguarding`. | `backend/rag`, `backend/safeguarding` |

### 🟠 High — functional gaps

| # | Finding | Where |
|---|---|---|
| F1 | **User management exists only in the frontend.** `/admin/users`, `/admin/next-user-id`, `/admin/onboard`, `/admin/reset-pin` and `/admin/offboard` have no backend routes; MSW mocks answer them in dev. Nothing can create a Firestore user, so there is no real onboarding path. | [api/index.ts](frontend/src/api/index.ts), [mocks/handlers.ts](frontend/src/mocks/handlers.ts) |
| F2 | **Teacher dashboard is always empty against the real backend.** It queries `quiz_scores WHERE student_id = ''`, and nothing writes `quiz_scores` anyway. In dev, MSW mocks mask this. | [teacher.py:26](backend/app/api/teacher.py#L26) |
| F3 | **Published tests leak answers and aren't recorded.** `/quiz/assigned` sends `correct_option` to students and grading happens client-side. Results never reach the backend, so teachers can't see who took a test. | [quiz.py:137](backend/app/api/quiz.py#L137), `TakePublishedTest.tsx` |
| F4 | **Duplicate answers corrupt scoring.** `/test/answer` doesn't reject a second answer to the same question (no unique `(attempt_id, question_id)`). The count reaches the total early and auto-submits with a wrong score. | [test.py:151](backend/app/api/test.py#L151) |
| F5 | **Students can read other classes' content.** `/study/chapters`, `/study/summary` and `/quiz/assigned` accept a `class_num` query param that overrides `user.class_num` for students. The RAG query endpoints correctly use `user.class_num`. | [study.py:26](backend/app/api/study.py#L26), [quiz.py:106](backend/app/api/quiz.py#L106) |
| F6 | **Summary-intent keywords are too broad.** `"बताओ"` (tell me) and `"समझाओ"` (explain) match most Hindi questions, and `"सार"` is a substring of many words. With a chapter selected, these questions get the canned chapter summary instead of an answer. | [study.py:229](backend/app/api/study.py#L229) |
| F7 | **`/ready` never returns 503.** `status_code` is computed but never used, and `checks.get("llm") != "error"` never matches the actual `"error: X"` value. | [health.py:53](backend/app/api/health.py#L53) |

### 🟡 Medium — hardening and correctness

- **Forged history turns.** The client sends `assistant` history turns, which are passed straight into the prompt. They are tier1-scanned, but a student can fabricate "assistant" content to steer the model (prompt injection).
- **Summary JSON is not validated.** `summary_generator` output is parsed with `json.loads` and not validated with Pydantic. Retrieval paths use `.get` on raw dicts; `QuizGenerateResponse`/`MCQOut` validation fails with a 500 if the model omits `explanation`.
- **Login lockout is per process.** The lockout dict isn't shared across the 2 gunicorn workers or Cloud Run instances. Rate limiting is per IP, and a whole centre may share one NAT IP.
- **Dev bypass depends on one setting.** The auth bypass and hardcoded credentials are active whenever `APP_ENV=development`, which is also the default in `config.py`. A misconfigured prod deploy would accept `dev-admin-token`.
- **PWA caches sensitive responses.** It caches `/study|test|admin/` API responses (`NetworkFirst`, 24 h) in Cache Storage. Study answers and admin data persist on shared devices after logout; `logout()` only clears localStorage keys.
- **Several SQL strings use f-strings.** They build from fixed or allowlisted fragments (`_update_textbook`, `bm25_search`, `find_similar`), so they aren't exploitable, but they break the letter of the "never raw string interpolation" rule.
- **External CSS breaks under CSP.** KaTeX CSS loads from `cdn.jsdelivr.net`. The backend CSP doesn't cover the frontend, which has no CSP of its own; if one mirrored the backend's, KaTeX styling would break.
- **Admin ingestion workarounds.** Ingestion creates a new engine per status update. `language` is detected once per PDF, not per chunk. The admin `chapter` field only fills chunks with no detected chapter.
- **Ingestion blocks the web process.** It runs as a FastAPI `BackgroundTask` inside the web process (sequential embedding of each chunk). On Cloud Run the instance can be throttled or recycled mid-ingestion, leaving `status=processing` forever, which also blocks deletion.
- **Class range checks disagree.** Admin upload allows 5–10, schemas and `build_filter` allow 1–10, and the CLI allows 1–10.
- **Dead code.** `rag/retrieval/gemini_client.py` (including a nonsensical `verify_numeric`), `CacheRepository.invalidate_subject` (the cache isn't invalidated when a textbook is deleted or re-uploaded, so stale answers can outlive their source for 7 days), `ScoreRepository.save`, and `tier2_scan`.
- **Doc drift.** `.env.template` still references Vertex credentials and a "diksha" email. `test_rag_e2e.py` uses `nomic-embed-text` (768-dim) against a 1024-dim column.

### 🟢 What's done well

- **Textbook boundary.** Every chunk query is pre-filtered by `class_num` + `subject` in SQL, and `build_filter` re-validates both.
- **Input validation.** Pydantic `max_length` on all user input, a subject allowlist on every route, PDF magic-byte and size checks, and path traversal protection on eval file reads.
- **Resilience.** Circuit breaker, per-route rate limits, and generic 500 messages with no internals leaked. API docs are disabled in production.
- **Privacy-aware logging.** Hashed IPs and student IDs in logs, no student names anywhere, and TTL purges on traces and cache.
- **RAG quality instrumentation.** Traces, prompt versioning tied to the cache, a keyword ground-check, MCQ validation and an eval harness with saved runs are unusually thorough for this stage.
- **Bilingual support throughout.** Hindi prompts, `medium` toggle, Devanagari-aware chunking and language fallback.
- **Security and CI hygiene.** Non-root container user, CORS wildcard rejected in production, and CI runs truffleHog, pip-audit, npm audit and type checks.

---

## 7. Testing status

- **Backend (9 modules):** pure unit tests for schemas, pre-filter, RRF, circuit breaker, token budget, prompt version, health and tier-1 phrases. There are **no tests for any route, the retrieval service, ingestion, or output-side safeguarding**, which is why C1/C2 went unnoticed.
- **Frontend (4 tests):** API client, LoginScreen, OfflineBanner and MyProgress, using MSW.
- CI runs pytest from `backend/` with `PHRASES_FILE_PATH=../safeguarding/...`, but modules that import `rag.*` depend on the broken `backend/rag` symlink (P6).
- Tests were **not executed** as part of this analysis.

---

## 8. Suggested priorities

1. **Safeguarding (C1–C4).** Add an output scan on every path that returns content: stream result, cache hit, summaries. Wire `tier1_flag_and_alert` into the 451 paths. Switch to word-boundary or token matching, remove over-broad phrases, and grow `test_cases.json` to 100+ cases, including false-positive cases like "grape". Add route-level tests that assert these guarantees.
2. **Make a fresh checkout runnable.** Replace the symlinks (P6), regenerate migrations from the models (P5), and fix the Dockerfile and requirements (P4).
3. **Production path.** Do the Firebase ID-token exchange in the frontend (P1), map `assistant→model` for Vertex (P2), and use one embedding model for ingestion and query per provider (P3).
4. **Close functional gaps.** Build the real admin user endpoints (F1), persist published-test attempts server-side and stop sending answer keys (F3, F2), add a unique constraint on answers (F4), and lock students to their own `class_num` (F5).
