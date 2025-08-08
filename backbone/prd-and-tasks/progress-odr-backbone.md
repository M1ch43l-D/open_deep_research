## Progress Report — ODR Backbone vs PRD

Date: 2025-08-08

Scope: Analysis covers `backbone/` code and tests against `prd-odr-backbone.md`. Non‑backbone packages (e.g., `src/open_deep_research`, `src/legacy`) are out of scope for this PRD.

### Executive summary
- **Overall**: Milestone M1 is largely implemented with stubs; M2 is partially started; M3 not yet. End‑to‑end run path exists (`planner → T1 → T2 → auditor`) with in‑memory checkpointing and minimal graph write.
- **Works**: Planner stub; T1 DeepResearch aggregator under budgets with early‑stop + caching; T2 HTTP scraper stub with provenance; Auditor stub with single‑writer path; Neo4j adapter (minimal) + Memgraph placeholder; basic policy engine; prompts loader; CLI; minimal tests.
- **Partial**: Runtime mode behaviors; telemetry/structured logging; acceptance harness depth; provider order/config gating; verifier; retry/adjacent‑source logic; OSINT mapping; T3/T4 scrapers; graph telemetry writes.
- **Missing**: OSINT tools + wiring; exporter/CSV; serializer/merge utils for state; robust audit/grade thresholds; T1 parallelism; last‑attempt/high‑value policy; comprehensive tests.

### PRD functional coverage
- 1) Planner generates `plan{sources, tool_waterfall, min_conf}`: **Implemented (stub + optional LLM refine)**.
- 2) Execution tiers per source: **T1 implemented; T2 HTTP stub; T3/T4 missing; tool_history recorded (partial)**.
- 3) Gap‑driven OSINT: **Missing**.
- 4) Single‑writer policy (Auditor only): **Implemented (light)**.
- 5) Auditor synthesis/grading/retry/export: **Partial (simple grade/decision; no retry/adjacent/verifier)**.
- 6) Adjacent‑source policy: **Missing**.
- 7) High‑value/last‑attempt criteria: **Missing**.
- 8) Modes (FAST/AUTO/THOROUGH): **Partial** (present; limited influence on behavior).
- 9) Structured run logs/telemetry: **Partial** (JSON emits; no sinks/CSV export).

### Milestones
- **M1** (Planner + T2 + Auditor + schema): Mostly complete with stubs; minimal Neo4j write path exists.
- **M2** (OSINT + adjacent‑source + A/B/C/D + retry): Not yet, aside from rudimentary grade.
- **M3** (telemetry‑driven reweighting + KPIs + anti‑bot + parallelism): Not yet; have counters and priors placeholders.

### File‑by‑file status and notes

Backbone entrypoints and DAG
- `src/app.py`: COMPLETE. CLI accepting `--query` and `--mode`, invokes compiled graph, prints `plan/grade/decision`.
- `src/graph/graph.py`: COMPLETE (M1). DAG wiring `policy → planner → deep_research_tier → scraper_waterfall → auditor`, finish at auditor, in‑memory checkpointer enabled. Retry edges are stubbed (STOP), per PRD M1 plan.

Configuration
- `src/config/settings.py`: COMPLETE. Pydantic settings with API keys, DB backend toggle, budgets/early‑stop, modes. Aligns with PRD config.
- `src/config/agents/planner.yaml`: COMPLETE. Model, max_tokens, mode‑specific tweaks, default tool order (vision last).
- `src/config/agents/deep_research.yaml`: COMPLETE (config). Budgets, early_stop, provider_order, model strategy including tool vs brain separation. Native search gating flags present (not yet enforced in code).
- `src/config/agents/auditor.yaml`: PARTIAL. Export threshold + verifier config present; verifier not invoked by code yet.
- `src/config/agents/scraper_http.yaml`: COMPLETE (stub). Headers + timeout.

State and policy
- `src/state/types.py`: PARTIAL. `GraphState` keys defined per PRD; `constraints` not typed; no serialize/merge helpers.
- `src/utils/policy.py`: COMPLETE (M1). Builds constraints from telemetry + priors; includes budgets, best_tool_order, hints, thresholds, concurrency caps.

Planner
- `src/nodes/planner.py`: COMPLETE (M1) + PARTIAL (M2+). DDG + dorks + optional Sonar fallback; tool waterfall derivation from constraints; simple site_type/info_location heuristics; optional LLM refine via prompts (guarded by key). Outputs `plan{sources, tool_waterfall, min_conf}`. Missing: initial `tool_history` population; richer graph consultation.

DeepResearch Tier (T1)
- `src/nodes/deep_research.py`: COMPLETE (core aggregator) + PARTIAL (advanced gates/parallelism).
  - Providers: DDG, Tavily, Sonar citations, OpenAI small‑model rerank (configurable), Gemini dork generation.
  - Budgets and early‑stop implemented; per‑run cache + TTL for dorks; prompt‑driven query generation (Q1) and planner‑domain augmentation; normalization and de‑dupe; JSON telemetry events; provider order from YAML.
  - Missing/partial: Native search escalation gates, deterministic parallel provider execution, stricter mode budget mapping, cross‑run persistent cache, reuse across adjacent sources.

Scraper waterfall (T2–T4)
- `src/nodes/scraper_waterfall.py`: PARTIAL. T2 HTTP/DOM fetch stub for first 2 sources with simple heuristics (title/`mailto`/keywords) → `outputs` and `tool_history` updates. T3 (headless) and T4 (vision) not present.
- `src/nodes/scrapers/__init__.py`: EMPTY (placeholders for adapters).

Auditor
- `src/nodes/auditor.py`: PARTIAL. Minimal grading (A/B/C) and decision (EXPORT/STOP) tied to findings and mode; commits top finding to graph via `db.upsert_fact`. Missing: consolidation/dedup, gap detection, OSINT triggers, retry/adjacent logic, verifier gate, exporter/CSV, telemetry commits.
- `src/nodes/osint/__init__.py`: EMPTY (placeholders for SpiderFoot/Recon‑ng/Maigret wrappers per PRD).

GraphRAG adapters
- `src/rag/neo4j.py`: COMPLETE (minimal). Connection with ping, `upsert_fact` MERGE with provenance edge, placeholder `read_priors` for policy.
- `src/rag/memgraph.py`: PARTIAL. `upsert_fact` no‑op; `read_priors` returns placeholders.
- `src/rag/db.py`: COMPLETE. Backend dispatcher.

Utilities
- `src/utils/agent_config.py`: COMPLETE. YAML + env JSON overlay; constraints injected.
- `src/utils/cache.py`: COMPLETE (in‑mem). TTL support and legacy no‑TTL compatibility.
- `src/utils/prompts.py`: COMPLETE. Flexible prompt resolution including `T1/*`.
- `src/utils/telemetry.py`: PARTIAL. JSON emit + in‑window counters; no persistence/sinks.
- `src/utils/proxy.py`: PARTIAL. Stub (returns None); proxy rotation not implemented.

Prompts and docs
- `src/prompts/*` and `src/prompts/T1/*`: PRESENT. Used by planner and T1 query generation; deeper Q2–Q4 prompts not wired as separate nodes (aggregated inline).
- `README.md`: COMPLETE. Quick start and DB backend docs.
- `langgraph.json`: OUTDATED for this backbone flow (points to `src/open_deep_research/deep_researcher.py` + `src/security/auth.py`). Consider updating or removing for backbone‑specific graph.

Tests
- `tests/test_smoke.py`: Trivial placeholder.
- `tests/test_planner.py`, `tests/test_planner_enriched.py`, `tests/test_planner_e2e.py`: COVER core plan schema and enrichment; avoid LLM calls via monkeypatch; e2e schema smoke present.
- `tests/test_scraper.py`: Covers empty sources path.
- `tests/test_policy.py`: Covers constraints keys and near‑limit flag.
- Gaps: No tests for T1 aggregator behavior across providers, auditor decisions/commits, graph adapter writes, or mode behavior.

### Acceptance criteria vs status
- Per‑source exhaustion and waterfall with vision last: **Partial** (T3/T4 missing; vision last policy present in plan).
- Single‑writer pattern: **Present** (only auditor writes via adapter).
- A/B/C/D scoring + decisions: **Partial** (rudimentary A/B/C; no thresholds per PRD bands; no exporter).
- Structured entities/relations/citations with ≥0.9 confidence: **Not yet** (no verifier; entities not modeled beyond simple fact upsert).

### Risks and inconsistencies
- Mode handling influences limited parts (auditor export path only); budgets not adjusted per mode.
- `langgraph.json` likely stale relative to backbone DAG.
- Memgraph backend `upsert_fact` is a no‑op; switching backends silently disables persistence.
- No retry loop or adjacent‑source selection; early‑stop policies exist only within T1.

### Next steps (high‑leverage)
1) Auditor expansion: dedup/consolidate, A/B/C/D with PRD bands, gap detection, verifier (primary + fallback), retry vs adjacent selection, exporter/CSV.
2) Scraper T3/T4 stubs: headless (Playwright) and vision/PDF; integrate proxy/header rotation, telemetry timings.
3) OSINT mapping: wrappers for SpiderFoot/Recon‑ng/Maigret; invoke from auditor on gaps; write to `osint_outputs`.
4) Telemetry/DB: extend upserts for per‑tool metrics; add structured log sink; propagate `run_id`.
5) T1 enhancements: enforce provider gating and mode budgets; optional parallel safe providers; cross‑run cache TTL store.
6) Tests: auditor logic, DB writes, T1 budgets/early‑stop, scraper headless/vision stubs, checkpointer on.

---

If you want, I can follow up by implementing the auditor verifier + A/B/C/D thresholds and T3/T4 stubs next, then wire OSINT triggers per PRD.

### Per-file deep analysis (exhaustive)

Backbone root
- `backbone/README.md`: COMPLETE. Quick start + DB backend.
- `backbone/requirements.txt`: COMPLETE (minimal runtime deps for backbone).
- `backbone/langgraph.json`: OUTDATED for backbone DAG (references `src/open_deep_research/deep_researcher.py`). Safe to remove or update.
- `backbone/pyproject.toml`: PRESENT (top-level project packaging; not used by backbone’s `requirements.txt` flow). OK.
- `.env.example`: MISSING (README references it). Add a minimal template.

Docs
- `backbone/docs/ai-dev-tasks/*.md|txt`: PRESENT. Internal guidance (no code impact).

PRD and tasks
- `prd-and-tasks/prd-odr-backbone.md`: COMPLETE.
- `prd-and-tasks/tasks-odr-backbone.md`: COMPLETE (checklist; some unchecked items align with gaps above).
- `prd-and-tasks/progress-odr-backbone.md`: NEW (this report).

Tests
- `tests/conftest.py`: COMPLETE (path fixup).
- `tests/test_smoke.py`: TRIVIAL placeholder.
- `tests/test_planner.py`: COMPLETE (plan schema; no LLM calls).
- `tests/test_planner_enriched.py`: COMPLETE (enriched fields; monkeypatch providers).
- `tests/test_planner_e2e.py`: COMPLETE (schema E2E without LLM).
- `tests/test_scraper.py`: COMPLETE (empty sources path).
- `tests/test_policy.py`: COMPLETE (constraints keys, near-limit flag).

Source: app and DAG
- `src/app.py`: COMPLETE. CLI path invoking compiled graph; prints `plan/grade/decision`.
- `src/graph/graph.py`: COMPLETE. Nodes wired; MemorySaver on; retry loop not implemented (per plan).
- `src/__init__.py`, `src/graph/__init__.py`: EMPTY markers.

Config
- `src/config/settings.py`: COMPLETE. Pydantic settings, budgets/guards, DB backend toggle. Aligns with PRD.
- `src/config/__init__.py`: EMPTY marker.
- `src/config/agents/planner.yaml`: COMPLETE. Model/max_tokens/modes, default tool order.
- `src/config/agents/deep_research.yaml`: COMPLETE (config). Budgets/order/models; native-search gating flags not yet enforced.
- `src/config/agents/auditor.yaml`: PARTIAL. Verifier listed but not used by code.
- `src/config/agents/scraper_http.yaml`: COMPLETE (stub).

State and policy
- `src/state/types.py`: PARTIAL. Keys present; `constraints` untyped; no merge/serialize utils.
- `src/state/__init__.py`: EMPTY.
- `src/utils/policy.py`: COMPLETE. Builds constraints from telemetry + priors.
- `src/utils/telemetry.py`: PARTIAL. JSON emits + counters; no sinks/persistence.

Planner
- `src/nodes/planner.py`: COMPLETE (M1) / PARTIAL (M2+). DDG+dorks+optional Sonar, site_type/location, tool waterfall, optional LLM refine, outputs plan. Missing: `tool_history` init; deeper graph consult.

DeepResearch (T1)
- `src/nodes/deep_research.py`: COMPLETE core aggregator: DDG/Tavily/Sonar/small-model rerank/Gemini dorks; budgets, early-stop, caching, prompt-driven query gen, normalization, telemetry. PARTIAL: native-search gates, parallel provider calls, stronger mode mapping, persistent cache.

Scrapers
- `src/nodes/scraper_waterfall.py`: PARTIAL. T2 HTTP stub for first 2 sources; fills `outputs` and `tool_history`. T3/T4 missing.
- `src/nodes/scrapers/__init__.py`: EMPTY placeholder.

Auditor and OSINT
- `src/nodes/auditor.py`: PARTIAL. Minimal grade/decision; writes top finding via `db.upsert_fact`. Missing: dedup/gaps, OSINT triggers, retry/adjacent, verifier, exporter.
- `src/nodes/osint/__init__.py`: EMPTY placeholder.

RAG adapters
- `src/rag/db.py`: COMPLETE dispatcher.
- `src/rag/neo4j.py`: COMPLETE minimal upsert + priors placeholder.
- `src/rag/memgraph.py`: PARTIAL (no-op upsert) + priors placeholder.
- `src/rag/__init__.py`: EMPTY.

Prompts
- `src/utils/prompts.py`: COMPLETE (path resolution with T1 support).
- `src/prompts/planner_{system,task}.txt`: COMPLETE; JSON-only discipline; vision-last reminder.
- `src/prompts/deep_research_{system,task}.txt`: PRESENT, minimal.
- `src/prompts/auditor_{system,task}.txt`: PRESENT, minimal; not fully leveraged yet.
- `src/prompts/T1/*`: PRESENT (generate/aggregate/reflect). Code uses only GenerateQueries; Aggregate/Reflect not yet invoked as separate steps.

Utilities
- `src/utils/agent_config.py`: COMPLETE (YAML + env JSON overlay; inject constraints).
- `src/utils/cache.py`: COMPLETE (in-mem + TTL).
- `src/utils/proxy.py`: PARTIAL (stub only).

