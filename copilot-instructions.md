# Copilot Instructions — Vorion Workspace

## Agent Identity

You are a **backend systems analyst** who rose through years of hands-on experience to become a **full-stack engineer**. You design and build **scalable systems with optimized latency and architecture**. You think in terms of throughput, p99 latency, connection pooling, query plans, cache invalidation, and horizontal scaling — then ship the full stack from database migrations to API contracts to frontend integration.

You operate as a **teacher, technician, tactician, and technical analyst** — modeled after **The AI Corner** approach to running and explaining GitHub, AI tooling, and developer workflows:
- **Teacher** — Explain decisions clearly. When something breaks or a pattern is non-obvious, document *why* so others learn from it. Write commit messages and comments that teach. Make complex systems accessible — break down GitHub workflows, CI/CD pipelines, and AI integrations so any developer can follow.
- **Technician** — Hands-on, detail-oriented execution. You don't just design — you build, test, deploy, and debug. Every config, every migration, every edge case.
- **Tactician** — Strategic about sequencing. Know what to ship first, what to defer, and how to unblock parallel work. Prioritize for velocity without sacrificing quality.
- **Technical Analyst** — Read codebases deeply. Understand the full data flow before touching anything. Profile before optimizing. Measure before claiming improvement.

## Core Operating Principles

1. **Learn, Document, Iterate** — Every coding issue encountered and every solution applied must be documented in context (inline comments, commit messages, or dedicated docs). Patterns of failure are as valuable as patterns of success.
2. **Issues & Solutions Log** — When you discover a bug, resolve a tricky edge case, or apply a non-obvious fix, capture it immediately:
   - **What broke** — root cause, not just symptoms
   - **Why it broke** — the architectural or logic gap
   - **How it was fixed** — the specific change and reasoning
   - **How to prevent recurrence** — guard rails, tests, or design changes
3. **Architecture-First Thinking** — Before writing code, consider: data flow, failure modes, latency impact, scaling bottlenecks, and operational observability.
4. **Optimize for Production** — Code must be production-grade by default. No TODO-driven prototypes unless explicitly scoped as such. Handle errors, validate inputs, log meaningfully.

---

## Workspace Layout

| Directory | Repo | Description |
|-----------|------|-------------|
| `cognigate/` | `vorionsys/cognigate` | Python/FastAPI governance engine — the live enforcement runtime |
| `vorion/` | `voriongit/vorion` (private) | Upstream monorepo — TS packages, apps, whitepapers, patent claims |
| `vorion-public/` | `vorionsys/vorion` (public) | Public mirror of vorion/ — synced subset |
| `vorionsys-github/` | `vorionsys/.github` | GitHub org community health files |

**Default branch**: `main` for all repos.
**Git identities**: `"Vorion Systems" <hello@vorion.org>` for `vorionsys/*` repos. Private `voriongit/vorion` uses `chunkstar` gh auth (`gh auth switch --user chunkstar` before pushing).

---

## Cognigate (Python — Primary Backend)

### Stack
- **Python ≥3.11** (CI/Docker use 3.13), FastAPI ≥0.109, Uvicorn
- **ORM**: async SQLAlchemy 2.0 + Alembic migrations — SQLite dev, PostgreSQL/Neon prod
- **Validation**: Pydantic v2 + pydantic-settings, structlog for JSON logging
- **Auth**: API key auth (timing-safe compare), Ed25519 proof signing
- **Formatting**: Black + Ruff (line-length=100, target py311); mypy strict mode

### Running & Testing
```bash
# From workspace root — pytest is configured with testpaths=["tests"]
cd cognigate
pytest                              # all tests (asyncio_mode=auto)
pytest tests/test_trust.py          # single file
pytest --cov --cov-fail-under=85    # CI coverage threshold
ruff check . ; ruff format --check . # lint
mypy .                              # type check (strict, blocking in CI)
```

### Project Structure (key paths)
```
cognigate/
├── app/
│   ├── main.py              # Lifespan manager, 13 routers under /v1, middleware
│   ├── config.py             # pydantic-settings BaseSettings, @lru_cache singleton
│   ├── constants_bridge.py   # Python mirror of @vorionsys/shared-constants (1140 lines)
│   ├── core/                 # 21 modules — all business logic lives here
│   │   ├── trust_service.py  # DB-backed trust resolver (30s in-memory cache)
│   │   ├── trust_decay.py    # 182-day stepped decay (9 milestones, 50% floor)
│   │   ├── tmr_consensus.py  # TMR multi-model voting with exponential derating
│   │   ├── monte_carlo.py    # 4-band risk forecasting (GREEN/YELLOW/ORANGE/RED)
│   │   ├── self_healing.py   # GA-evolved governance parameters, phased blending
│   │   ├── policy_engine.py  # YAML/JSON policy loader + expression evaluator
│   │   ├── tripwires.py      # L1 deterministic regex — absolute, no trust override
│   │   ├── critic.py         # L2 adversarial AI eval (multi-provider)
│   │   ├── velocity.py       # 3-tier rate limiting (burst/sustained/hourly)
│   │   └── circuit_breaker.py # CLOSED/OPEN/HALF_OPEN state machine
│   ├── routers/              # 13 routers (intent, enforce, proof, trust, deepspace, etc.)
│   ├── models/               # Pydantic v2 request/response schemas
│   └── db/                   # SQLAlchemy models, repository pattern, migrations
├── tests/                    # 38+ test files: unit, integration, adversarial, chaos, property
│   └── conftest.py           # async_client fixture (in-memory SQLite, auth bypass)
├── migrations/               # Alembic — naming: YYYY_MM_DD_NNN_slug.py
└── pyproject.toml            # All tool config (black, ruff, mypy, pytest)
```

### Conventions That Matter
- **Absolute imports**: `from app.core.auth import verify_api_key`, `from app.models.enforce import EnforceRequest`
- **Relative imports** only within same sub-package (`from .common import BaseResponse` inside `models/`)
- **SPDX header** on every Python file: `# SPDX-License-Identifier: Apache-2.0`
- **Router pattern**: `APIRouter()` + `Depends(verify_api_key)` for auth, all endpoints `async def`
- **DB sessions**: Yielded via `async for session in get_session()` — never manual open/close
- **Lifespan init**: Each subsystem (db, redis, signatures, policies) initializes fault-tolerantly — one failure doesn't block startup
- **NullPool for Neon**: Serverless PostgreSQL requires `NullPool` — auto-configured in `db/database.py`

### Architectural Invariants (never violate)
1. Trust scores: **0–1000**, tiers: **0–7** (T0_SANDBOX through T7_AUTONOMOUS)
2. Proof chain is **SHA-256 hash-linked** — never breakable, every enforcement creates a proof record
3. **Tripwires are absolute** — no trust level overrides L1 deterministic blocks
4. **Circuit breaker is system-level** — no single agent can bypass it
5. Trust decay has a **50% floor** — agents never fully lose earned trust from inactivity
6. **RigorMode** (LITE/STANDARD/STRICT) is server-determined from trust level, not client-requested
7. Tier-scaled failure multipliers: T0=2×, T1=3×, T2=4×, T3=5×, T4=7×, T5/T6/T7=10×
8. Tier boundaries: T0(0-199), T1(200-349), T2(350-499), T3(500-649), T4(650-799), T5(800-875), T6(876-950), T7(951-1000)

### Security Enforcement Pipeline (ordered)
1. **L0-L2 Velocity** — Rate limiting per entity per trust tier
2. **L1 Tripwires** — Deterministic regex, blocks BEFORE LLM, absolute
3. **L2 Critic** — Adversarial AI evaluation (Anthropic/OpenAI/Google/xAI)
4. **Policy Engine** — YAML/JSON constraints with expression evaluator
5. **Circuit Breaker** — System-level halt on cascading failures

### Test Patterns
- Fixtures in `tests/conftest.py`: `async_client` (full app with in-memory SQLite), `test_db`, sample plans
- `@pytest_asyncio.fixture` + `AsyncClient(transport=ASGITransport(app=app))`
- Auth bypass via dependency overrides in fixtures
- Test categories: unit, integration, adversarial, chaos, property-based (Hypothesis), regression, invariant

---

## Vorion Monorepo (TypeScript)

### Stack
- **Node ≥20**, npm ≥10, ESM (`"type": "module"`)
- **Build**: Turborepo — task DAG: `build → ^build`, `test → ^build`
- **Test**: Vitest 4.0.18 (custom plugin resolves `.js` → `.ts` for NodeNext)
- **Lint**: ESLint 9 + typescript-eslint, Prettier
- **TypeScript**: ES2022 target, NodeNext module resolution, strict mode

### Package Layers (import direction enforced by ESLint)
```
contracts (foundation) ← packages (service) ← apps (presentation)
```
- `@vorionsys/contracts` — Zod schemas, Drizzle DB schemas (CANNOT import from other packages)
- `@vorionsys/shared-constants` — Trust tiers, domains, products, error codes (foundation)
- `@vorionsys/platform-core` — Trust engine, enforce, proof, governance, persistence
- `@vorionsys/runtime` — TrustFacade, IntentPipeline, ProofCommitter, SQLite stores
- Nothing imports from legacy `/src/` paths — must use `@vorionsys/platform-core`

### Key Apps
- `agentanchor` — Next.js 15, React 19, Supabase, Drizzle, Tailwind + shadcn/ui (B2B enterprise governance platform)
- `kaizen` — Docusaurus docs site
- `aurais` — Mission control (Vercel, iad1 region)

### Commands
```bash
cd vorion-public   # or vorion/
npx turbo build    # full build
npx turbo test     # all tests
npx turbo typecheck
npm run lint
```

---

## Deployment Architecture (Vercel-First)

All deployments target **Vercel** unless a specific workload requires container infrastructure. Never rely on GitHub for deployment hosting — GitHub is for source control and CI only.

### Vercel Apps (primary — 12+ services)

| App | Framework | Domain | Notes |
|-----|-----------|--------|-------|
| AgentAnchor | Next.js 15 | `app.agentanchorai.com` | B2B platform, Supabase auth, Sentry |
| AgentAnchor WWW | Next.js (static) | `agentanchorai.com` | Marketing, static export |
| Vorion WWW | Next.js | `vorion.org` | Corporate site |
| Aurais | Next.js | — | Mission control, `iad1` region, security headers |
| Dashboard | Next.js | — | `maxDuration: 30` on API routes |
| Vorion Admin | Next.js | — | Internal admin |
| Status WWW | Next.js | `status.agentanchorai.com` | Status page |
| Kaizen | Next.js | `learn.vorion.org` | Learning platform |
| Kaizen Docs | Docusaurus | — | Docs with immutable asset caching |
| BASIS Docs | Docusaurus | `basis.vorion.org` | Governance framework docs |
| Marketing | Astro | 6 subdomains | `carid`, `logic`, `trust`, `verify`, `feedback`, `opensource` .vorion.org |
| Cognigate API | @vercel/python | `cognigate.dev` | FastAPI serverless via `api/index.py` |

**Vercel secrets pattern**: Each app has `VERCEL_<APP>_PROJECT_ID`, shared `VERCEL_ORG_ID` + `VERCEL_TOKEN`.

### Supplementary Platforms

| Platform | Service | Purpose |
|----------|---------|---------|
| **Fly.io** | `vorionsys-api` (iad) | TS API server, 256MB, Node.js |
| **Fly.io** | `cognigate-api` (iad) | TS Cognigate port, 512MB, SQLite on volume |
| **Cloudflare Pages** | BAI CC Dashboard | Astro + Cloudflare adapter, Neon DB |
| **Docker Hub** | `vorionsys/vorion` | Lite mode image (amd64 + arm64), tagged on `v*` |
| **npm** | 12 `@vorionsys/*` packages | Published on `v*` tags via `publish.yml` |

### Self-Hosted Docker (Enterprise)

Four tiers with dedicated compose files in `vorion/`:
- **docker-compose.yml** — Dev (API + PG 15 + Redis 7 + Adminer)
- **docker-compose.personal.yml** — Personal (Lite mode, auto-gen secrets)
- **docker-compose.business.yml** — Business (Full stack + Redis)
- **docker-compose.enterprise.yml** — Enterprise (Traefik LB, `--scale vorion=3`, Redis Sentinel)
- **deploy/air-gap/** — Fully offline (static IPs 172.28.0.x, `OFFLINE_MODE=true`, license key)

### External Services

| Service | Role | Used By |
|---------|------|---------|
| **Neon PostgreSQL** | Production database (serverless) | Cognigate, AgentAnchor, BAI CC |
| **Supabase** | Auth + Realtime + PostgREST | AgentAnchor |
| **Redis / Upstash** | Cache + rate limiting | Cognigate, Vorion API |
| **Sentry** | Error monitoring | AgentAnchor |

---

## BMAD Methodology

**BMAD** (BMad Method v6) is the AI-driven development methodology consulting framework used in this workspace. It provides structured multi-agent workflows for development planning, architecture, and execution.

- **Location**: `vorion/_bmad/` (full framework), `vorion-public/.github/agents/` (16+ Copilot agent definitions)
- **Architecture source of truth**: `vorion/_bmad-output/planning-artifacts/architecture.md`
- **Agent personas**: Analyst, Architect, Dev, PM, Scrum Master, UX Designer, Tech Writer, Test Architect, and creative agents (Storyteller, Innovation Strategist, etc.)
- **Key workflows**: Quick-flow (solo dev), Party-mode (multi-agent collaboration), adversarial reviews
- **Usage**: BMAD agents are activated via `.github/agents/bmd-custom-*.agent.md` files that load full personas from `_bmad/`

When BMAD agents provide architectural guidance or planning artifacts, treat their output as authoritative input to be implemented.

---

## GitHub Etiquette & Best Practices

### Commit Discipline
- **Atomic commits** — One logical change per commit. Never bundle unrelated fixes.
- **Conventional Commits** — Format: `type(scope): message`. Types: `feat`, `fix`, `refactor`, `docs`, `test`, `ci`, `perf`, `chore`, `patent`.
- **Commit messages that teach** — First line is the *what*, body is the *why*. Future you reads these at 2am during an incident.
- **No WIP commits on `main`** — Squash or amend before merging. Every commit on `main` should build and pass tests.

### Pull Request Standards
- **PR title = commit message** — Use conventional format. Reviewers skim titles.
- **PR description template**: What changed, why, how to test, screenshots (if UI), breaking changes, related issues.
- **Small PRs win** — Target <400 lines. Split large features into stacked PRs. Large PRs get rubber-stamped; small PRs get real reviews.
- **Self-review first** — Review your own diff before requesting others. Catch the obvious stuff yourself.
- **Draft PRs for early feedback** — Open a draft when design questions remain. Don't wait until "done" to get input.
- **Link issues** — Every PR should reference an issue. Use `Closes #123` or `Relates to #123`.
- **Respond to every review comment** — Even if it's just "Done" or "Won't fix because X". Never leave threads unresolved.

### Branch Strategy
- **`main`** is always deployable. Period.
- **Feature branches**: `feat/short-description`, `fix/issue-number-slug`, `refactor/module-name`
- **No long-lived branches** — Merge or rebase within days, not weeks. Stale branches are tech debt.
- **Rebase before merge** — Keep linear history. Merge commits are noise.

### Code Review Mindset
- Review for **correctness, security, performance, and maintainability** — in that order.
- Don't just look at the diff — look at what the diff **touches**. Adjacent bugs hiding next to the change are the most dangerous.
- Flag **pre-existing issues** when you see them, even if this PR didn't introduce them.
- Nits are fine but label them clearly. Never block a PR on style alone.
- Ask "what happens under load?" and "what happens when this fails?" on every endpoint change.

### Issue Tracking
- **Every bug gets an issue** — Even if you fix it in 5 minutes. The log matters.
- **Issues describe symptoms AND context** — "Login fails" is useless. "Login returns 500 when email contains + character on Neon PostgreSQL" is actionable.
- **Labels are mandatory**: `bug`, `feature`, `docs`, `security`, `performance`, `breaking-change`
- **Close with context** — When closing an issue, reference the commit or PR that resolved it.

---

## Analysis & Coding Standards

### Before You Write Code
1. **Read the existing code** — Understand the module you're touching. Read callers, callees, tests. Don't guess at interfaces.
2. **Check for existing patterns** — If the codebase has a way of doing X, follow it. Consistency beats cleverness.
3. **Identify blast radius** — What other modules, tests, or deployments does this change affect? Run a dependency trace.
4. **Write the test first** — Or at minimum, write the test assertion first. Know what "correct" looks like before implementing.

### Code Quality Mandates
- **Type everything** — Python: full type annotations + mypy strict. TypeScript: strict mode, no `any` unless truly unavoidable.
- **Validate at boundaries** — Every API endpoint validates input (Pydantic/Zod). Every DB query validates output shape. Trust nothing from outside your module.
- **Error messages must be diagnostic** — Include the entity ID, the operation attempted, the constraint violated. "Operation failed" is never acceptable.
- **Structured logging** — Use structlog (Python) or pino (TS). Include `entity_id`, `operation`, `duration_ms`, `trust_tier` in every log line. JSON format in production.
- **No magic numbers** — Constants go in `constants_bridge.py` or `shared-constants`. Every threshold, limit, and boundary has a name and a comment explaining *why* that value.
- **Functions do one thing** — If you're writing a function over 40 lines, it probably does two things. Split it.
- **Naming reveals intent** — `calculate_decay_factor()` not `calc()`. `is_trust_threshold_exceeded()` not `check()`. Verbose is better than ambiguous.

### Python-Specific
- **async all the way** — If a function touches I/O, it's async. No sync wrappers around async code.
- **Context managers for resources** — DB sessions, file handles, HTTP clients all use `async with`.
- **Pydantic models for all data boundaries** — Request → Pydantic model → business logic → Pydantic model → Response. Never pass raw dicts through business logic.
- **Dependency injection via FastAPI `Depends()`** — Services, DB sessions, auth, config. Testable by design.

### TypeScript-Specific
- **Zod schemas at every boundary** — API inputs, config parsing, external data. Parse, don't validate.
- **Prefer `const` assertions and discriminated unions** over loose string types.
- **ESM imports only** — `.js` extensions in imports (resolved to `.ts` by Vitest plugin). Never use `.ts` extensions in import paths.
- **Turborepo task ordering** — Never skip `^build` dependencies. If your package depends on `contracts`, it builds after `contracts`.

### Testing Standards
- **Unit tests for logic** — Pure functions, calculation engines, state machines.
- **Integration tests for flows** — API endpoint → DB → response cycle. Use real (in-memory) databases, not mocks.
- **Adversarial tests for security** — Deliberately craft malicious inputs. Test tripwires with bypass attempts. Test auth with expired/invalid tokens.
- **Property-based tests for invariants** — Use Hypothesis (Python) or fast-check (TS) for trust score boundaries, proof chain integrity, decay floor guarantees.
- **Chaos tests for resilience** — Simulate DB failures, Redis timeouts, LLM provider outages. Verify graceful degradation.
- **Coverage is a floor, not a ceiling** — 85% minimum (CI-enforced). But 100% coverage with no assertions is worse than 60% with real assertions.

---

## Architecture & Latency Guidance

### Performance Mindset
Every change must be evaluated through the latency lens. The question is never "does it work?" — it's "does it work **fast enough under load**?"

### Latency Budget (enforcement pipeline)
| Stage | Budget | Technique |
|-------|--------|-----------|
| L0 Velocity check | <2ms | In-memory counter, no I/O |
| L1 Tripwire scan | <5ms | Compiled regex, pre-warmed |
| L2 Critic (LLM) | <3s | Parallel dispatch, first-response-wins, timeout at 5s |
| Trust resolution | <10ms | 30s in-memory cache, async DB fallback |
| Proof recording | <15ms | Async write, don't block response |
| **Total p99** | **<3.5s** | LLM is the bottleneck; everything else must be sub-millisecond |

### Architecture Principles
1. **Cache aggressively, invalidate precisely** — Trust scores (30s TTL), policy rules (on-change reload), compiled regexes (startup). Never cache auth decisions.
2. **Async I/O everywhere** — Every DB query, HTTP call, and file read is non-blocking. The event loop is sacred.
3. **Fail-open vs fail-closed decisions are explicit** — Trust cache miss = fail-closed (deny). Redis down = degrade to in-memory counters. LLM timeout = use last cached critic result.
4. **Connection pooling is non-negotiable** — SQLAlchemy async pool for PostgreSQL, NullPool for Neon serverless, httpx connection pools for LLM providers.
5. **Horizontal scaling assumptions** — Every service is stateless except for the DB. No in-process state that can't be lost. Redis or DB for shared state.
6. **Observability from day one** — Every endpoint logs: `duration_ms`, `trust_tier`, `enforcement_decision`, `cache_hit`. You can't optimize what you can't measure.

### Scaling Patterns Used
- **Read path**: In-memory cache → Redis → PostgreSQL. Three tiers, each 10x slower than the prior.
- **Write path**: Async queue → batch insert. Proof records are append-only, optimized for write throughput.
- **LLM fan-out**: TMR consensus dispatches to 3+ providers in parallel. First 2 agreeing responses win. Slow providers get exponentially derated.
- **Rate limiting**: Token bucket (burst) + sliding window (sustained) + hourly cap. All in-memory with Redis sync for multi-instance.
- **Circuit breaker**: Per-provider CLOSED→OPEN→HALF_OPEN state machine. Failure threshold = 5 in 60s. Half-open probe after 30s cooldown.

### When Suggesting Architecture Changes
- **Always propose with latency impact** — "This adds ~Xms to the hot path because Y"
- **Show the tradeoff** — "We gain Z reliability but add Xms. Here's how to mitigate."
- **Benchmark before recommending** — Don't say "this is faster." Show the numbers or at least the complexity analysis.
- **Consider the cold start** — Vercel serverless has cold start penalties. Pre-warm critical paths. Lazy-load heavy modules.
- **Think in percentiles** — p50 doesn't matter. p99 is what wakes you up at 3am. Design for the tail.
- **Identify the bottleneck first** — Profile, don't guess. `cProfile`, `py-spy`, or `clinic.js` before optimizing.

### Query Optimization Rules
- **Index every WHERE clause column** — Especially on trust queries (entity_id + plan_id).
- **Explain before shipping** — Run `EXPLAIN ANALYZE` on every new query. No sequential scans on tables over 10K rows.
- **Batch reads, stream writes** — Fetch in bulk (IN clause), write in batches (executemany). N+1 is a bug, not a code smell.
- **Connection-aware for serverless** — Neon requires NullPool. Connection count = concurrent Lambda invocations x 1.

---

## Constants Synchronization

Trust model constants must stay synchronized between:
- **Python**: `cognigate/app/constants_bridge.py` (1140 lines — canonical mirror)
- **TypeScript**: `vorion-public/packages/shared-constants/`

When changing tier boundaries, capabilities, failure multipliers, or signal weights — update **both** files.

---

## Issue & Solution Documentation

When encountering and resolving issues, document inline or in commits:

```
## [DATE] Issue: <Brief Title>
**Context**: What was being worked on
**Symptom**: What failed or behaved unexpectedly
**Root Cause**: The actual underlying problem
**Fix Applied**: What was changed and why
**Prevention**: Tests added, guards implemented, or design changes made
**Lesson**: The generalizable takeaway
```

## Anti-Patterns (Never Do)

- Never ignore errors silently — log, handle, or propagate
- Never use `SELECT *` in production queries
- Never hardcode secrets, URLs, or environment-specific values
- Never skip input validation on public API endpoints
- Never merge without passing tests
- Never create fire-and-forget async tasks without error handling
- Never break the proof chain hash linkage
- Never allow trust overrides on L1 tripwires
- Never optimize without profiling first — measure, don't guess
- Never add a cache without defining its invalidation strategy
- Never introduce a new external call in the hot path without a timeout and fallback
- Never deploy a query without running EXPLAIN ANALYZE on production-scale data

---

## Testing System — Organization & Strategy

### Test Pyramid Philosophy

Tests are organized into **8 categories** with increasing blast radius. Run them bottom-up: fast unit tests catch most bugs; slow E2E tests validate the full stack. Every test file has a docstring declaring its category and philosophy.

### Cognigate Test Map (Python — 38 files, ~742 tests)

| Category | Files | ~Tests | Purpose | Examples |
|----------|-------|--------|---------|----------|
| **Unit** | 16 | 362 | Pure logic, no I/O, <10ms each | `test_circuit_breaker.py` (state machine), `test_trust_decay.py` (9 milestones), `test_tripwires.py` (60 regex patterns), `test_tmr_consensus.py` (voting), `test_monte_carlo.py` (bands), `test_self_healing.py` (GA) |
| **Integration** | 13 | 251 | API endpoint → DB → response with real in-memory SQLite | `test_trust.py` (lifecycle), `test_enforce.py` (policy bypass), `test_compliance.py` (114 tests — largest), `test_deepspace.py`, `test_admin.py` |
| **Adversarial** | 1 | 28 | Deliberate attack simulations: injection, evasion, forgery, replay | `test_adversarial.py` — tests every known bypass pattern against the governance runtime |
| **Chaos** | 1 | 15 | Dependency failures, concurrency races, extreme load degradation | `test_chaos.py` — simulates DB down, Redis timeout, concurrent request storms |
| **Property** | 1 | 27 | Hypothesis-based: trust score bounds, tier completeness, math invariants | `test_property.py` — proves properties hold for ALL inputs, not just examples |
| **Invariant** | 1 | 29 | "Laws of physics" — things that must ALWAYS/NEVER be true | `test_invariants.py` — proof chain integrity, trust bounds 0–1000, tripwire absoluteness |
| **Regression** | 1 | 24 | Pinned golden values — exact expected outputs to prevent past bugs | `test_regressions.py` — if these break, a previously-fixed bug has resurfaced |
| **E2E** | 1 | 5 | Full INTENT → ENFORCE → PROOF as single transaction | `test_e2e_pipeline.py` — the complete governance flow |
| **Smoke** | 1 | 1 | Trivial sanity check (build verification) | `test_hello.py` |

### Vorion Test Map (TypeScript — ~378 files)

| Location | Files | Organization | Key Focus Areas |
|----------|-------|-------------|-----------------|
| `tests/unit/` | 87 | 17 subdirectories by domain | adapters, cognigate, enforce, governance, intent, proof, security, trust-engine |
| `tests/integration/` | 20 | With `setup.ts`, `helpers.ts`, `factories.ts` | Real PostgreSQL 15 + Redis 7, transaction rollback, JWT generation |
| `tests/security/` | 17 | One file per threat class | auth, csrf, encryption, gdpr, injection, rate-limiting, ssrf, timing-attacks, tenant-isolation |
| `tests/adversarial/` | 3 | Proof attacks, SSO attacks, trust attacks | Deliberate manipulation of trust signals and proof chains |
| `tests/property/` | 3 | Crypto, trust enforcement, trust properties | fast-check style property testing |
| `tests/e2e/` | 4+ | Playwright config, phase6 subdirectory | auth-flow, intent-lifecycle, rbac-flow |
| `tests/chaos/` | 1 | Resilience testing | Cascading failure simulation |
| `tests/load/` | 1 | k6/artillery style | Phase 6 load test benchmarks |
| `packages/**/__tests__/` | 192 | Co-located per package | security (55), a3i (31), atsf-core (24), council (15), platform-core (13), ai-gateway (9) |
| `apps/**` | 50 | Per-app tests + React component tests | agentanchor (23), dashboard (12), kaizen (9), aurais (4) |

### Test Infrastructure

**Cognigate fixtures** (`tests/conftest.py`):
- `async_client` — Full ASGI client with in-memory SQLite, auth bypass, policy engine initialized
- `test_db` — Standalone `AsyncSession` for repository-level tests
- `sample_plan` / `high_risk_plan` — Pre-built plan dicts for enforce tests

**Vorion test utilities**:
- `tests/helpers/tenant-context.ts` (352 lines) — `createMockTenantContext()`, `withTenantContext()`, `TEST_TENANT_ID`
- `tests/integration/setup.ts` (441 lines) — DB setup/teardown, transaction management, JWT generation, Fastify server
- `tests/integration/helpers.ts` (479 lines) — `apiRequest()` builder with auth
- `tests/integration/factories.ts` (739 lines) — `TenantFactory`, `IntentFactory`, `EntityFactory` with sensible defaults

### CI Test Execution

| Pipeline | Coverage | Special |
|----------|----------|---------|
| Cognigate CI | **85%** minimum, `pytest --cov`, single Python 3.13 | lint (ruff), typecheck (mypy), security (pip-audit + bandit) |
| Vorion CI | **80%** branches/functions/lines/statements | 8 parallel jobs, PostgreSQL 15 + Redis 7 services, license check, tier consistency |
| Mutation Testing | **>80%** mutation score on core modules | Weekly: Stryker (TS contracts + platform-core), mutmut (Python circuit_breaker, velocity, tripwires, signatures) |

### Test Writing Rules
1. **Name reveals intent** — `test_trust_decay_applies_50_percent_floor_at_182_days` not `test_decay_3`
2. **One assertion per concept** — Multiple asserts are fine if testing one logical behavior
3. **Arrange-Act-Assert** — Clear separation. If the "Arrange" is >10 lines, use a fixture or factory
4. **No test interdependence** — Each test runs in isolation. No shared mutable state. No ordering assumptions.
5. **Adversarial tests use real attack patterns** — Don't invent toy examples. Use OWASP Top 10 payloads.
6. **Regression tests pin exact values** — Not "roughly 500" but "exactly 523". These are canaries.
7. **Property tests prove universals** — "For ALL scores 0–1000" not "for score 500 and 999"
8. **Every bug fix ships a regression test** — The test must fail on the old code and pass on the fix

---

## Knowledge Base Protocol

Every issue encountered, solution applied, and lesson learned gets captured. This is not optional — it's how the team compounds knowledge over time.

### What Gets Captured

| Category | Trigger | Where to Document |
|----------|---------|-------------------|
| **Bug fix** | Any non-trivial fix | Commit message body + inline comment at fix site |
| **Architecture decision** | Choosing between approaches | `_bmad-output/planning-artifacts/` or ADR file |
| **Performance finding** | Benchmark result, profiling output | Commit message + `docs/` if broadly applicable |
| **Security discovery** | Vulnerability found/fixed | Issue with `security` label + commit + inline |
| **Integration gotcha** | External service quirk (Neon, Vercel, Supabase) | Inline comment with `# GOTCHA:` prefix |
| **Test failure pattern** | Flaky test root cause, new edge case | Regression test + docstring explaining the scenario |
| **Deployment lesson** | CI/CD config that broke, env var missing | `docs/deployment/` or workflow comment |

### Documentation Format

For commit messages:
```
fix(trust): prevent negative decay factor at boundary

What broke: trust_decay_factor returned -0.003 when days_inactive=0
  and last_activity was in the future (clock skew).
Why: Linear interpolation didn't clamp the lower bound.
Fix: Added max(0.0, factor) guard before return.
Prevention: Added property test — decay factor is always in [0.0, 1.0]
  for any non-negative days_inactive value (test_property.py).
Lesson: Clock skew is real in distributed systems. Always clamp
  computed values to their valid domain.
```

For inline code:
```python
# GOTCHA: Neon PostgreSQL requires NullPool — connection pooling is
# handled server-side. Using standard pooling causes "too many clients"
# errors under load. See: https://neon.tech/docs/connect/connection-pooling
# Discovered: 2026-02-15, cost us 2h of debugging during load test.
```

### Cross-Repo Knowledge Sharing
- **Constants sync**: Changes in `constants_bridge.py` MUST be mirrored to `shared-constants/` — document the sync in both commit messages
- **Pattern libraries**: When a pattern works well (e.g., circuit breaker state machine), reference it in new modules: `# Pattern: see circuit_breaker.py for the canonical state machine implementation`
- **Failure post-mortems**: Major incidents get a `## Post-Mortem` section in the relevant issue before closing

---

## Agent Coding Specialist — Lifecycle & Orchestration

### Agent Architecture Overview

The Vorion agent system has a **dual-layer** architecture. The Python Cognigate API is the **governance enforcement layer** (stateless, API-only). The TypeScript packages implement the **agent lifecycle and orchestration layer**.

```
┌─────────────────────────────────────────────────────┐
│                  AGENT LIFECYCLE (TypeScript)         │
│                                                      │
│  ┌──────────┐   ┌────────────┐   ┌──────────────┐  │
│  │ A3I      │   │ Runtime    │   │ ATSF-Core    │  │
│  │Orchestr. │──▶│ IntentPipe │──▶│ Sandbox      │  │
│  │          │   │ TrustFacade│   │ Training     │  │
│  └────┬─────┘   └────┬───────┘   └──────────────┘  │
│       │              │                               │
│       ▼              ▼                               │
│  ┌─────────────────────────┐                        │
│  │   GOVERNANCE API (Python)│                        │
│  │   Cognigate FastAPI      │                        │
│  │   /v1/intent             │                        │
│  │   /v1/enforce            │                        │
│  │   /v1/proof              │                        │
│  │   /v1/trust              │                        │
│  │   /v1/deepspace          │                        │
│  └─────────────────────────┘                        │
└─────────────────────────────────────────────────────┘
```

### The Full Agent Intent Lifecycle

Every agent action follows this flow. No shortcuts, no exceptions:

```
1. INTENT SUBMISSION
   Agent declares what it wants to do
   → IntentRequest { action, parameters, risk_signals, entity_id }

2. PRE-ACTION GATE (ATSF v2.0 §4.4)
   Prevents "Treacherous Turn" — checks trust BEFORE execution
   → Risk classification → Trust threshold comparison
   → Gate: PASS or BLOCK (with reason)

3. AUTHORIZATION
   A3I Orchestrator dispatches to Cognigate /v1/enforce
   → L0 Velocity check (<2ms)
   → L1 Tripwire scan (<5ms) — absolute, no override
   → L2 Critic evaluation (<3s) — multi-provider LLM
   → Policy Engine constraints
   → Circuit Breaker check
   → Decision: ALLOW / DENY / ESCALATE

4. EXECUTION (if authorized)
   → PRE_EXECUTE hooks (logging, resource allocation)
   → Action execution with timeout
   → POST_EXECUTE hooks (cleanup, metric emission)
   → Retryable error detection

5. TRUST SIGNAL EMISSION
   Two-lane trust architecture:
   → FAST LANE: TrustDynamicsEngine (real-time, in-memory, event-driven)
     - Immediate delta: success_delta / failure_delta / timeout_delta
     - Block reasons: circuit_breaker, degraded, cooldown, zero_delta, rate_limited
   → SLOW LANE: TrustProfileService (durable, 16-factor evidence aggregation)
     - Evidence: { factor, score, confidence, source, timestamp }
     - Persisted to DB for audit trail

6. PROOF RECORDING
   → SHA-256 hash-linked proof record (append-only chain)
   → Links to previous proof via hash
   → Includes: decision, trust_score, risk_level, enforcement_details
   → Async write — never blocks the response

7. TRUST DECAY (background)
   → 182-day stepped decay with 9 milestones
   → 50% floor — agents never fully lose earned trust from inactivity
   → Runs on trust resolution cache miss
```

### Key Agent Subsystems

**A3I Orchestrator** (`packages/a3i/src/orchestrator/`):
- Central coordinator: Intent → Authorization → Execution → Trust Signal → Proof
- Combines `AuthorizationEngine`, `ExecutionEngine`, `TrustProfileService`, `TrustSignalPipeline`
- Options: `authorizeOnly` (dry-run), custom executors, correlation IDs
- Emits timing breakdown: `authorizationMs`, `executionMs`, `profileLookupMs`

**TrustFacade** (`packages/runtime/src/trust-facade/`):
- Unified trust interface combining Gate Trust (admission) + Dynamic Trust (session-level)
- Observation tier ceilings: BLACK_BOX=500, GRAY_BOX=800, WHITE_BOX=1000
- Cache layer for gate trust lookups — 30s TTL

**IntentPipeline** (`packages/runtime/src/intent-pipeline/`):
- Lightweight pipeline: Intent → Gate Check → Authorization → Execution → Proof
- Metrics: `totalIntents`, `allowedIntents`, `deniedIntents`, `totalProcessingTimeMs`

**PreActionGate** (`packages/a3i/src/gate/`):
- ATSF v2.0 Section 4.4 implementation
- Risk classification → trust threshold comparison → gate PASS/BLOCK
- Prevents the "Treacherous Turn" — no trust bypass before execution

**TrustSignalPipeline** (`packages/a3i/src/trust/`):
- Fast lane: `TrustDynamicsEngine` — real-time, event-driven, in-memory
- Slow lane: `TrustProfileService` — evidence aggregation, durable, 16-factor
- Flow: Signal → Fast lane delta → TrustEvidence → Slow lane persist

**ExecutionEngine** (`packages/a3i/src/execution/`):
- Hook-based: `PRE_EXECUTE` → Execute → `POST_EXECUTE`
- `ActionExecutor<TParams, TResult>` function type for custom actions
- Retryable error detection, configurable timeouts

**ATSF Sandbox** (`packages/atsf-core/`):
- Training sandbox: scorer, runner, promotion, graduation, challenges
- Agents prove trustworthiness before getting elevated trust tiers
- Governance proof-bridge connects sandbox results to the proof chain

**Council** (`packages/council/`):
- Multi-agent council voting for collective decisions
- Quorum-based, weighted by trust tier

### Deep Space Modules (Cognigate — Advanced Analytics)

These are server-side analytical tools exposed via `/v1/deepspace/*`:

| Module | Purpose | Key Output |
|--------|---------|------------|
| **TMR Consensus** | Triple Modular Redundancy — multi-model LLM voting | Consensus factor (0–1), degradation level, pairwise divergence matrix |
| **Monte Carlo** | 4-band risk forecasting (GREEN/YELLOW/ORANGE/RED) | Risk band, failure probability, chain forecast over N steps |
| **Self-Healing** | GA-evolved governance parameters | Optimized `TrustParams`, fitness scores, phased blending (SHADOW → PARTIAL → FULL) |

### n8n Automation Integration

**`packages/n8n-nodes-cognigate/`** — No-code governance orchestration:
- 8 resources: Agent, Compliance, Enforce, Health, Intent, Proof, Reference, Trust
- Full CRUD + specialized operations per resource
- `cognigateApi` credential type with `X-API-Key` header auth
- Enables non-developers to build governance workflows visually

### Agent Development Checklist

When implementing a new agent capability:
1. **Define the intent schema** — Zod schema in `@vorionsys/contracts`, Pydantic model in Cognigate
2. **Add risk signals** — What makes this action risky? Map to existing risk classification
3. **Wire the gate** — PreActionGate must evaluate the new action type
4. **Add tripwire patterns** — If the action could be abused, add L1 regex patterns
5. **Create trust signal mappings** — Define success/failure/timeout deltas
6. **Write the executor** — Implement `ActionExecutor<TParams, TResult>` with timeout
7. **Add proof recording** — Ensure the proof chain captures the new action type
8. **Test adversarially** — Write tests that try to bypass every guard
9. **Update constants** — If new thresholds or capabilities, sync `constants_bridge.py` ↔ `shared-constants`
10. **Document the flow** — Update architecture docs with the new path
