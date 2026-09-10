# ContextHub POC — Implementation Plan

> **Status:** Draft for review · **Author:** hvohra@dataeconomy.ai · **Date:** 2026-08-31
> **Nothing in this document has been implemented.** This is the build specification.

---

## 1. Context & objective

`contexthub-web` today is a 12-screen React mockup that *describes* a governed context/memory
layer for GenAI agents. It is 100% static:

- `src/data/mock.ts` holds flat display strings (`'184k'`, `'2m ago'`) — not records
- `src/screens/Ask.tsx` returns one hardcoded paragraph; the `Ask` button has no `onClick`
- `src/components/GraphCanvas.tsx` is decorative physics with no hit-testing
- `ContextAsset` — the README's headline contract — is declared in `src/types.ts` and never instantiated
- Persona switching reshapes only `Overview.tsx` and `TopBar.tsx`; 10 screens ignore it

The mockup sells a thesis it cannot demonstrate:

> **One governed graph, many persona-scoped views, served to agents over MCP
> with ACL enforcement, PII masking and full audit.**

**Objective:** make that thesis executable end-to-end on synthetic data, so it can be
demonstrated rather than described.

### The demo this plan builds toward

Ask **the same question** as three different personas and watch the retrieved context, the
redactions, the answer and the audit trail visibly differ — then point Claude Code at the MCP
server, ask it the same question, and watch the ContextHub audit trail light up live as an
external agent consumes the hub.

### Decisions already locked

| Decision | Choice |
|---|---|
| Depth | Real backend + Claude synthesis + MCP server |
| Backend language | Python / FastAPI |
| Input sources | GitHub, Jira, Slack |
| Personas | Data Engineer, Digital Engineer, Citizen Developer |
| Synthetic org | Fintech / payments SaaS |

`docker-compose.yml` already reserves port 8080 with the comment *"8080 is used by another local
service, e.g. a uvicorn/FastAPI app"* — this plan takes that reservation.

---

## 2. Scope

### In scope — becomes real

| Screen | Becomes |
|---|---|
| `Ask.tsx` | Live query → governed retrieval → streamed Claude answer with resolvable citations |
| `Sources.tsx` | Real connectors, real record counts, real sync state |
| `Graph.tsx` | Real nodes/edges, click-to-inspect actually works |
| `Steward.tsx` | Real entity-resolution proposals; Approve/Reject mutate the graph |
| `Governance.tsx` | Real audit trail from real serve events |
| `Measure.tsx` | Counters derived from the audit table |
| `Overview.tsx` | Derived global counts; LiveFeed streams real serve events |

### Out of scope — stays mock, by design

`MarketPositioning.tsx`, `DomainProducts.tsx`, `Federation.tsx`, `Skills.tsx`.

These are positioning/vision surfaces, not POC surfaces. This will be stated in the README rather
than left for a reviewer to discover.

### Non-goals

- No real connector credentials or live SaaS API calls — the corpus is synthetic and committed
- No multi-tenancy, no auth/login — persona is selected in the UI, not authenticated
- No production deployment, autoscaling or HA
- No vector database (see §4.1 for the rationale)

---

## 3. The synthetic organization

**Meridian Pay** — a B2B payments platform, ~40 employees across 5 teams.

Deliberately built on entities the UI **already names**, so no existing screen copy becomes a lie:

| Already in the app | Where it appears today |
|---|---|
| `billing-service`, `identity-api`, `data-loader`, `gateway`, `scheduler` | `personas.ts` DevOps board |
| Payments Team, tech lead **A. Rivera** | `Ask.tsx:44` |
| **RFC-114** · Dunning & Retry Policy, **RFC-88** | `Ask.tsx:46`, `Steward.tsx:47` |
| **PAY-2871** "retries not honoring cap" → **PR #3182** | `Ask.tsx:8`, `mock.ts:101` |
| `Acme Corp` ↔ `Account` resolution pair | `mock.ts:50` |
| Domains: Billing & Payments, SDLC/Engineering, Customer 360 | `strategy.ts:42-44` |

### 3.1 Why Slack is deliberately dirty

Slack is the source that makes governance *visible*. The generator plants, on purpose:

| Planted artefact | Drives |
|---|---|
| Emails, phone numbers, Luhn-valid card fragments, customer names | PII detection + masking counts |
| An engineer stating "retries cap at 3" when RFC-114 says 5 | Trust scoring — the low-trust chunk must be visibly outranked |
| 2 prompt-injection payloads | The `blocked` row in the audit trail |
| One `#payments-security` channel classified `confidential` | The ACL denial that separates the three personas |

### 3.2 Cross-source identity collisions

The generator emits the same human and the same service under different names per source, and
**never writes the mapping into the corpus** — the ingestion pipeline has to earn it:

```
Person:    arivera (GitHub)        · Ana Rivera (Jira)          · @ana (Slack)
Service:   billing-service (repo)  · "Billing Service" (Jira)   · #billing (Slack)
Customer:  Acme Corp (Jira)        · ACME CORPORATION (Slack)   · acct_acme_prod (code)
```

Pairs above the auto-merge threshold collapse silently. The ambiguous middle band becomes the
**Steward review queue** — which for the first time contains genuine proposals rather than the six
frozen rows in `mock.ts:47-54`.

---

## 4. Architecture

```
contexthub-web/
  src/                          # existing React app        (Vite,  :5173)
  server/                       # NEW — FastAPI             (uvicorn, :8080)
    app/
      main.py                   # app, CORS, X-Persona middleware, routers
      config.py                 # all tunables from §7, env-overridable
      models.py                 # Pydantic mirrors of src/types.ts
      db.py                     # SQLite schema + access
      routers/
        ask.py  sources.py  graph.py  steward.py
        governance.py  measure.py  personas.py
      ingest/
        base.py                 # Connector protocol: fetch() -> list[RawRecord]
        github.py  jira.py  slack.py
        chunker.py              # token-window chunking
        pipeline.py             # normalize -> chunk -> embed -> extract -> link
        resolve.py              # entity resolution + confidence banding
      retrieval/
        lexical.py              # BM25
        vector.py               # brute-force cosine over a numpy matrix
        graph_walk.py           # NetworkX n-hop expansion from seed entities
        rerank.py               # optional cross-encoder
        hybrid.py               # RRF fusion + trust weighting + trace emission
      governance/
        acl.py                  # persona policy -> pre-filter + post-filter
        pii.py                  # detect + mask/redact, returns field count
        injection.py            # prompt-injection scan
        audit.py                # writes a ServeEvent for every serve
      llm/
        client.py               # Anthropic client, streaming, fallback
        prompts.py              # grounded-synthesis system prompt
      synth/
        generate.py             # deterministic, seeded generator
        corpus/                 # generated JSON — committed to the repo
    mcp_server/
      server.py                 # MCP stdio server -> the same governed path
    tests/
    pyproject.toml
  docs/POC-PLAN.md              # this file
```

### 4.1 Deliberately no vector database

~3,500 chunks × 384 dims is a **5 MB float32 numpy array**. Brute-force cosine over it is
sub-5 ms — faster than the network hop to a vector DB would be. Adding Qdrant or pgvector to a POC
costs a service, a schema and a migration path, and buys nothing measurable at this scale.

This will be stated explicitly in the README. It reads as engineering judgment; leaving it
unexplained reads as a shortcut.

### 4.2 The critical invariant

> `POST /api/ask` and the MCP `graph.retrieve` tool **call the same function**.

Governance cannot be bypassed by coming in through the agent door. This is the entire claim being
made on the Governance screen, and it is currently unprovable. Enforced by a test (§9, M6-1).

---

## 5. Data model

### 5.1 SQLite schema

| Table | Key columns |
|---|---|
| `documents` | `id`, `source`, `source_ref`, `title`, `body`, `author_raw`, `created_at`, `updated_at`, `classification`, `domain` |
| `chunks` | `id`, `document_id`, `ordinal`, `text`, `token_count`, `trust`, `embedding_row` |
| `entities` | `id`, `type`, `canonical_name`, `attrs_json`, `steward_state` |
| `aliases` | `entity_id`, `source`, `raw_name`, `confidence`, `resolved_by` |
| `edges` | `src_id`, `dst_id`, `rel`, `weight`, `provenance_doc_id` |
| `pii_spans` | `chunk_id`, `start`, `end`, `kind`, `confidence` |
| `review_queue` | `id`, `type`, `title`, `proposal_json`, `confidence`, `state`, `origin` |
| `serve_events` | `ts`, `actor`, `persona`, `action`, `asset`, `acl`, `pii_masked`, `latency_ms`, `channel` |

`serve_events` is the single source of truth behind Governance, Measure and the LiveFeed.
`channel` distinguishes `http` from `mcp` — which is what makes the MCP demo visible in the UI.

### 5.2 Entity types

Mirrors the seven layers already rendered in `mock.ts:37-45`, so `Graph.tsx` and `Overview.tsx`
need no new legend:

`Document` · `Entity` · `Service` · `Person` · `PII` · `Metric` · `Policy`

### 5.3 Edge relations

`OWNS` · `AUTHORED` · `MENTIONS` · `FIXES` · `IMPLEMENTS` · `DEPENDS_ON` · `MEMBER_OF` · `GOVERNS`

---

## 6. API contracts

All endpoints require an `X-Persona: <persona-id>` header. Missing or unknown → `400`.

| Method | Path | Returns |
|---|---|---|
| `POST` | `/api/ask` | SSE stream: `trace` → `chunks` → `guardrails` → `token`* → `done` |
| `GET` | `/api/sources` | `SourceConnector[]` with real counts and sync state |
| `GET` | `/api/graph?focus=&depth=` | `{nodes, edges}` — ACL-filtered |
| `GET` | `/api/graph/node/{id}` | Node detail for the inspector |
| `GET` | `/api/steward/queue` | `ReviewItem[]` from real ER proposals |
| `POST` | `/api/steward/{id}/approve` \| `/reject` | Mutates the graph, returns the delta |
| `GET` | `/api/governance/audit?limit=` | `ServeEvent[]` |
| `GET` | `/api/governance/stats` | Derived tiles for `Governance.tsx:13-16` |
| `GET` | `/api/measure` | Derived counters for `Measure.tsx:14-18` |
| `GET` | `/api/overview` | Derived globals for `Overview.tsx:9-14` |
| `GET` | `/api/events` | SSE — live serve events for `LiveFeed.tsx` |
| `GET` | `/api/personas` | Persona list + ACL scope summary |

The `POST /api/ask` SSE contract maps 1:1 onto the panels already drawn in `Ask.tsx` — `trace`
fills the Retrieval Trace card, `guardrails` fills the Guardrails card, `chunks` fills Retrieved
Context. **No new UI panels are needed.**

---

## 7. Configuration parameters

All live in `server/app/config.py` as a Pydantic `Settings` object, every value env-overridable
with a `CH_` prefix.

### 7.1 Corpus generation

| Parameter | Default | Note |
|---|---|---|
| `SEED` | `20260831` | Generation must be fully deterministic |
| `ORG_NAME` | `Meridian Pay` | |
| `HEADCOUNT` / `TEAMS` | `40` / `5` | |
| `GITHUB_REPOS` | `12` | |
| `GITHUB_PRS` | `60` | Each links 0–2 Jira keys |
| `GITHUB_DOCS` | `24` | ADRs, RFCs, runbooks under `docs/` |
| `GITHUB_CODE_EXCERPTS` | `40` | |
| `JIRA_PROJECTS` | `PAY, IDN, DATA, SUP` | |
| `JIRA_ISSUES` | `250` | |
| `JIRA_COMMENTS_PER_ISSUE` | `0–6` | Uniform |
| `SLACK_CHANNELS` | `8` | One classified `confidential` |
| `SLACK_THREADS` / `SLACK_MESSAGES` | `40` / `350` | |
| `PLANTED_PII_INSTANCES` | `24` | Asserted by test M1-2 |
| `PLANTED_INJECTIONS` | `2` | |
| `PLANTED_CONTRADICTIONS` | `3` | |
| `PLANTED_ALIAS_COLLISIONS` | `12` | Across Person / Service / Customer |
| `TIME_WINDOW_MONTHS` | `18` | Ending `2026-08-31` |

### 7.2 Chunking & embedding

| Parameter | Default | Note |
|---|---|---|
| `CHUNK_TOKENS` | `320` | |
| `CHUNK_OVERLAP` | `64` | |
| `MIN_CHUNK_TOKENS` | `40` | Below this, discard |
| `EMBED_MODEL` | `sentence-transformers/all-MiniLM-L6-v2` | ~90 MB, cached locally after first pull |
| `EMBED_DIM` | `384` | |
| `EMBED_NORMALIZE` | `true` | Lets cosine reduce to a dot product |
| `EMBED_BATCH` | `64` | |
| *Expected chunk count* | *~3,500* | Sanity bound, not a config |

### 7.3 Retrieval

| Parameter | Default | Note |
|---|---|---|
| `BM25_TOP_K` | `40` | |
| `VECTOR_TOP_K` | `40` | |
| `GRAPH_HOPS` | `2` | Matches the `depth 2` already shown in the UI |
| `GRAPH_SEED_MAX` | `5` | Cap on seed entities parsed from the query |
| `RRF_K` | `60` | Standard reciprocal-rank-fusion constant |
| `GRAPH_PROXIMITY_BOOST` | `0.15` | Per hop closer to a seed entity |
| `RERANK_ENABLED` | `true` | |
| `RERANK_MODEL` | `cross-encoder/ms-marco-MiniLM-L-6-v2` | |
| `FINAL_K` | `8` | Matches "8 chunks" in `Ask.tsx:52` |
| `MIN_SCORE` | `0.25` | Floor — drop below this even if `FINAL_K` unfilled |

### 7.4 Trust scoring

```
trust = 0.40·authority + 0.25·steward + 0.20·freshness + 0.15·corroboration
```

| Component | Values |
|---|---|
| `authority` | `github/docs 1.00` · `github/code 0.85` · `jira 0.60` · `slack 0.35` |
| `steward` | `approved 1.0` · `unreviewed 0.6` · `flagged 0.2` |
| `freshness` | Exponential decay, `TRUST_HALFLIFE_DAYS = 180` |
| `corroboration` | `min(1, supporting_chunks / 3)` |

Weights are config, not constants — the demo may want to tune how hard the contradicting Slack
message is punished.

### 7.5 Entity resolution

| Parameter | Default | Note |
|---|---|---|
| `ER_SIMILARITY` | `rapidfuzz.token_set_ratio` | |
| `ER_AUTO_MERGE` | `≥ 0.90` | Merges silently |
| `ER_REVIEW_BAND` | `0.65 – 0.90` | → Steward queue |
| `ER_REJECT` | `< 0.65` | Discarded |
| `ER_BLOCKING_KEY` | normalized surname / service slug | Avoids O(n²) comparison |

### 7.6 PII

| Parameter | Default |
|---|---|
| `PII_DETECTORS` | `email, phone_e164, phone_us, pan_luhn, iban, ssn, ipv4, person_name` |
| `PII_MASK_STYLE` | `partial` — `a****@meridianpay.io` |
| `PII_REDACT_STYLE` | `full` — `[REDACTED:EMAIL]` |
| `PII_MIN_CONFIDENCE` | `0.75` |

`masked` vs `redacted` is the per-persona difference (§7.7): masked keeps shape and shows a count,
redacted removes the span entirely.

### 7.7 ACL matrix — what makes the three personas differ

| Persona | Sources | Classifications | PII |
|---|---|---|---|
| `data-engineer` | github, jira, slack | public, internal, confidential | masked |
| `digital-engineer` | github, jira, slack | public, internal | masked |
| `citizen` | github, jira | public | redacted |

Expected outcome for `"Who owns the billing retry logic and what's the current retry policy?"`:

| Persona | Chunks | Sources | Governance outcome |
|---|---|---|---|
| Data Engineer | 8 | 3 | 3 PII fields masked · full answer citing RFC-114 |
| Digital Engineer | 6 | 3 | `#payments-security` withheld · answer flags 1 restricted source |
| Citizen Developer | 3 | 1 | **Slack denied entirely** · degraded answer · "2 sources withheld by policy" |

These exact numbers are the assertion in test **M3-1**. If the corpus drifts, fix the test *and*
the demo script together.

### 7.8 Claude synthesis

Parameters verified against the `claude-api` skill on 2026-08-31 — several differ from what a
model-from-memory would produce.

| Parameter | Value | Note |
|---|---|---|
| `LLM_MODEL` | `claude-opus-5` | $5 / $25 per 1M in-out. 1M context |
| `LLM_MAX_TOKENS` | `4096` | Deliberately short output — a grounded answer, not an essay |
| `LLM_STREAMING` | `true` | `client.messages.stream()`, proxied to the browser as SSE |
| `LLM_THINKING` | `{"type": "adaptive"}` | On by default for Opus 5 |
| `LLM_EFFORT` | `medium` | `output_config.effort`. Latency-sensitive UI route — start here and measure before raising |
| `LLM_CACHE` | `cache_control: {"type": "ephemeral"}` on the system block | The grounding instructions are identical across every request |
| `LLM_FALLBACKS` | `"default"` + beta `server-side-fallback-2026-07-01` | Recommended default for Opus 5 |
| `LLM_ENABLED` | `true` | Set `false` to force the offline path |

**Parameter traps to avoid — each returns HTTP 400 on Opus 5:**

- `temperature` / `top_p` / `top_k` — **removed**, not merely ignored
- `budget_tokens` inside `thinking` — **removed**; use `output_config.effort` instead
- Assistant-message prefill — **removed**

Always check `stop_reason == "refusal"` before reading `content`.

**Offline fallback.** With no credential available, `/api/ask` returns a template-assembled
extractive answer over the *same* governed chunks. The demo must survive dead conference wifi.
Auth resolves via `ANTHROPIC_API_KEY`, then `ANTHROPIC_AUTH_TOKEN`, then an `ant auth login`
profile — an unset env var does **not** mean no credential.

### 7.9 Runtime

| Parameter | Default |
|---|---|
| `API_PORT` | `8080` |
| `WEB_PORT` | `5173` (dev) · `8090` (nginx prod, already in `docker-compose.yml`) |
| `CH_DB_PATH` | `./data/contexthub.db` |
| `CH_CORPUS_DIR` | `./app/synth/corpus` |
| `CORS_ORIGINS` | `http://localhost:5173` |
| `SSE_HEARTBEAT_SEC` | `15` |

---

## 8. Milestones

| # | Milestone | Effort | Depends on |
|---|---|---|---|
| M0 | Scaffold | 0.5 d | — |
| M1 | Synthetic corpus | 1.5 d | M0 |
| M2 | Ingestion, graph & entity resolution | 2.0 d | M1 |
| M3 | Retrieval & governance | 2.0 d | M2 |
| M4 | Claude synthesis | 1.0 d | M3 |
| M5 | Front-end wiring | 2.5 d | M3 (M4 for streaming) |
| M6 | MCP server | 1.0 d | M3 |
| M7 | Demo polish | 1.0 d | M5, M6 |
| | **Total** | **11.5 d** | |

---

### M0 — Scaffold · 0.5 d

**Deliverables**
- `server/` package, `pyproject.toml`, pinned dependencies
- FastAPI app with CORS and the `X-Persona` middleware
- `config.py` carrying every parameter in §7
- `GET /api/health`
- `api` service added to `docker-compose.yml` on 8080

**Exit criteria** — `docker compose up` serves both :5173 and :8080; `/api/health` returns 200;
an unknown `X-Persona` returns 400.

---

### M1 — Synthetic corpus · 1.5 d

**Deliverables**
- `synth/generate.py` — deterministic under `SEED`
- `synth/corpus/{github,jira,slack}.json`, committed
- Planted artefacts per §7.1, with a manifest recording where each was placed

**Exit criteria**
- Two runs at the same seed produce byte-identical output
- The manifest's planted counts match §7.1 exactly
- The corpus contains **no** alias→canonical mapping (verified by grep) — M2 must earn it

---

### M2 — Ingestion, graph & entity resolution · 2.0 d

**Deliverables**
- Three connectors implementing the `Connector` protocol
- Chunker, embedder, numpy vector store
- NetworkX graph build with the §5.3 relations
- `resolve.py` with the §7.5 confidence banding
- `POST /api/ingest` (dev-only) and a `make ingest` target

**Exit criteria**
- `arivera` / `Ana Rivera` / `@ana` collapse to one `Person`
- A `FIXES` edge exists: `PR #3182` → `PAY-2871`
- Ambiguous pairs land in `review_queue`, **not** auto-merged
- Full ingest completes in < 60 s on a laptop

---

### M3 — Retrieval & governance · 2.0 d

**Deliverables**
- BM25, vector and graph-walk retrievers
- RRF fusion, proximity boost, trust weighting, optional rerank
- `acl.py`, `pii.py`, `injection.py`, `audit.py`
- One `retrieve_governed(query, persona)` entry point — **the** shared function of §4.2
- Trace object matching the four steps in `Ask.tsx:10-15`

**Exit criteria** — the core test. See M3-1 in §9.

---

### M4 — Claude synthesis · 1.0 d

**Deliverables**
- `llm/client.py` per the §7.8 parameters
- Grounded-synthesis system prompt: cite only supplied chunks as `[n]`, never assert beyond them,
  explicitly state when context was withheld by policy
- SSE streaming through `/api/ask`
- Citation resolution back to real chunk IDs
- Offline extractive fallback

**Exit criteria** — every `[n]` in an answer resolves to a returned chunk ID; the offline path
produces a grounded answer with `LLM_ENABLED=false`.

---

### M5 — Front-end wiring · 2.5 d

**Deliverables**
- `src/api/client.ts` — typed fetch, sends `X-Persona` from the existing `PersonaContext`
- Seven screens wired per §2
- `GraphCanvas.tsx` given real nodes/edges and hit-testing, so the label's promise of
  `CLICK NODE TO INSPECT` stops being false
- Every count on a wired screen derived, never asserted

**Exit criteria** — no wired screen imports from `src/data/mock.ts`; switching persona in the
TopBar visibly changes Ask results; the four out-of-scope screens still render unchanged.

---

### M6 — MCP server · 1.0 d

**Deliverables**
- `mcp_server/server.py`, MCP Python SDK, stdio transport
- Tools: `graph.retrieve(query, k)` · `graph.walk(entity, depth)` · `context.assets(filter)`
- Persona bound at **launch config**, not chosen per call — matching the
  `acting-as user ACL · inherited` guardrail in `Ask.tsx:18`
- A `.mcp.json` snippet in the README

**Exit criteria** — see M6-1 in §9. MCP-originated serves appear in the Governance table with
`channel = mcp`.

---

### M7 — Demo polish · 1.0 d

**Deliverables**
- `make demo` — reset, generate, ingest, serve
- README rewrite: current 12 screens (it still documents 8), the backend, the no-vector-DB
  rationale, and an explicit list of what stays mock
- The §10 demo script, rehearsed

**Exit criteria** — a cold clone reaches step 7 of §10 without manual intervention.

---

## 9. Verification

### Test matrix

| ID | Milestone | Assertion |
|---|---|---|
| M1-1 | M1 | Two runs at `SEED` are byte-identical |
| M1-2 | M1 | Planted counts match §7.1 (24 PII, 2 injections, 3 contradictions, 12 collisions) |
| M1-3 | M1 | No alias→canonical mapping present in the corpus |
| M2-1 | M2 | The three `A. Rivera` aliases resolve to one `Person` node |
| M2-2 | M2 | `PR #3182 --FIXES--> PAY-2871` edge exists |
| M2-3 | M2 | Review-band pairs are queued, not merged |
| **M3-1** | **M3** | **Same query, three personas → chunk counts 8 / 6 / 3; Citizen Developer receives zero Slack chunks; three `ServeEvent` rows with `allowed`/`allowed`/`denied`** |
| M3-2 | M3 | An injection payload produces a `blocked` event and never reaches the LLM |
| M3-3 | M3 | The contradicting Slack chunk ranks below RFC-114 for every persona that can see both |
| M4-1 | M4 | Every `[n]` resolves to a returned chunk ID — no fabricated citations |
| M4-2 | M4 | `LLM_ENABLED=false` still returns a grounded answer |
| **M6-1** | **M6** | **MCP `graph.retrieve` and `POST /api/ask` return identical governed chunk sets for the same persona and query, and both write audit rows** |

M3-1 and M6-1 are the two tests that prove the product thesis. Everything else is hygiene.

---

## 10. Demo script

`docker compose up` → :5173 (web) + :8080 (api)

1. **`/sources`** — three connectors, real record counts, real last-sync times
2. **`/graph`** — click `billing-service`; see real edges to its owner, PRs, tickets, threads
3. **`/steward`** — approve the `Ana Rivera ↔ arivera` merge; watch the graph node count drop by one
4. **`/ask` as Data Engineer** — full grounded answer, 8 chunks, 3 PII fields masked
5. **Switch to Citizen Developer** in the TopBar; ask the *identical* question — visibly degraded
   answer, Slack denied, "2 sources withheld by policy"
6. **`/governance`** — both serves in the audit trail, one `allowed`, one `denied`
7. **Point Claude Code at the MCP server**, ask the same question — a new audit row appears live
   with `channel = mcp`

**Step 5 is the whole product in ten seconds.** Everything else is supporting evidence.

---

## 11. Risks

| Risk | Mitigation |
|---|---|
| Synthetic corpus feels obviously fake under demo scrutiny | Build on entities the UI already names; write real prose in the RFCs, not lorem ipsum. Budget review time in M1 |
| Retrieval quality too weak to make the persona contrast crisp | The corpus is authored alongside the retriever — tune `FINAL_K`, trust weights and the planted contradiction together until M3-1 passes cleanly |
| First-run model download (~90 MB embed + ~90 MB rerank) stalls a cold demo | Bake both into the Docker image at build time; never download during a demo |
| No network at demo time | Offline extractive fallback (§7.8); rehearse the demo with `LLM_ENABLED=false` at least once |
| MCP surface drifts from the HTTP surface | Test M6-1 makes divergence a build failure, not a discovery |
| M5 (front-end) is the largest single block and the most cuttable | If schedule slips, cut M5 screens back to `Ask` + `Governance` — those two carry steps 4–7 of the demo |

---

## 12. Repo hygiene noted along the way

- **`postcss.config.zip`** — a stray 14.6 MB archive at the repo root, not a config file. Delete.
- **`README.md`** documents 8 screens; there are 12. Refresh in M7.
- **`tailwind.config.js`** defines a brighter palette that is entirely unused — the `index.css`
  CSS variables win. Leave it, but do not add to it.
- **The mockup contradicts itself** in several places: `42 connectors` vs 12 rows,
  `34 pending` vs 6, `64 skills · 21 MCP tools` vs 9, and `graphLayers` counts that don't sum to
  the `1.84M Entities` global. Every count on a wired screen becomes derived, so the POC never
  ships a number it cannot defend.

---

## 13. Open questions for review

1. **Confluence as a 4th source** — dropped from the locked set, but RFC-114 currently lives there
   in `Ask.tsx:7`. This plan relocates RFCs into GitHub `docs/`, which is defensible and cheaper.
   Confirm that's acceptable, or add Confluence back (~0.5 d, it is doc-shaped like GitHub docs).
2. **Effort level** — `medium` is the §7.8 starting point for a latency-sensitive UI route.
   Worth a measured sweep in M7 against answer quality.
3. **Persona authentication** — out of scope here; persona is UI-selected. Real entitlement
   inheritance would be the natural first post-POC increment.
