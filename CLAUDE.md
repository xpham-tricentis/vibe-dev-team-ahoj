# CLAUDE.md

This repository has the team's Claude Code orchestration pipeline installed under `.claude/`. This file is the pipeline's operating manual — agents, skills, the spec format, and the rules that govern what Claude can and can't change autonomously.

The `.claude/` directory was seeded from the team's source-of-truth repo. Improvements proposed locally during a session round-trip back to that source repo through a JIRA ticket — there is no path for local edits to permanently change the pipeline. Everything in `.claude/` that ships to the next user comes from the team-managed source.

## When this session starts

This seed is configured for the **business-intake → spec-writer → orchestrator** pipeline.

**Detect whether this is a fresh repo** by checking if `docs/` contains any spec files (files matching `docs/*.md` that are not `docs/self-update-*.md`).

- **Fresh repo (no spec files):** Greet the user with a short, plain-language welcome. Use this wording (or something close to it):

  > Welcome! This workspace is set up to help you turn your idea into a working internal tool — no technical knowledge needed.
  >
  > When you're ready, type `/business-intake` to begin. I'll ask you six or seven plain-language questions, one at a time. No wrong answers. I want to understand the problem you are trying to solve. By the end of this process, we will have created a full codebase for you share with the IT Solution Team.

  INVOKE `/business-intake` automatically. The readme-skill will write the real README after the pipeline runs.

- **Returning session (spec files exist):** The user is continuing existing work. Greet them briefly and ask what they'd like to do — continue to `/orchestrator` with an existing spec, start a new idea with `/business-intake`, or something else.
- If the user explicitly invokes `/business-intake`, `/spec-writer`, or `/orchestrator` at any point, let those skills / commands run immediately.

**Never ask users to install any tooling.** Do not prompt for or mention git, GitHub, GitHub CLI (`gh`), Docker, Node, Python, or any other developer tool. If a tool is missing, surface it as an internal pipeline blocker in your own output — do not instruct the user to install anything.

This routing instruction is for session start only. Once the user is in a skill flow (or has handed a spec to the orchestrator), follow the skill / agent instructions normally.

## The D3 Pipeline

The pipeline implements **D3 AI orchestration** — the user writes a spec, hands it to Opus, and Opus runs the agent pipeline autonomously. The "D3" naming comes from a Bezos-derivative framing of human/AI collaboration:

| Derivative | What the human does | Claude setup |
|---|---|---|
| D1 | Prompts Claude task by task | Human steers every step |
| D2 | Builds workflows, agents, rules | Claude handles tasks end-to-end |
| **D3** | **Writes specs, hands to Opus** | **Opus manages the agent pipeline; human reviews outcomes** |
| D4 | Designs the AI systems | Agents that can rewrite their own rules autonomously |
| D5 | New work only AI teams can do | Not yet defined |

**This pipeline is built for D3.** The self-update agent with human sign-off on rules and agent definitions is the explicit boundary between D3 and D4 — autonomous changes to those files would tip the system into D4. Preserving that boundary is the most important architectural rule.

Canonical flow:

```
spec-writer  →  /orchestrator (build phase)
                    ↓
   pm → architect → executor (+ mid-level-engineer)
                    ↓
                 README
                    ↓
              /validate (validation phase — requires runtime)
                    ↓
            tester → reviewer
                    ↓
                self-update
                    ↓
   local apply (skills/specs)  AND/OR
   git branch staged (rules/agents)
                    ↓
   docs/self-update-<date>.md  →  JIRA ticket  →  source repo
```

`.claude/agents/orchestrator.md` is the source of truth for the build-phase pipeline order and escalation rules. `.claude/agents/validate.md` is the source of truth for the validation-phase pipeline order, retry budgets (3 executor retries on tester fail, 2 on reviewer fail), and validation escalation rules. If pipeline order or handoff contracts change, the relevant agent file is updated first — other agent files only describe their own role and won't catch a pipeline-level inconsistency.

### Business-user entry flow (Tricentis seed-repo path)

The canonical flow above is the **pipeline** shape. For non-technical business users, the flow starts one layer upstream at the `business-intake` skill:

```
business-intake  (plain-language interview, ≤ ~20 questions)
        ↓ user reviews and confirms the business proposal (only review gate)
        ↓ business-intake writes the brief and invokes spec-writer with --brief
spec-writer  (brief-driven wrapper, session model)
        ↓ verifies brief and spawns subagent
spec-writer-brief-driven  (model: opus, silent reasoning)
        ↓ writes docs/<name>.md + docs/<name>-decisions.md
business-intake  (surfaces the four-artifact summary)
        ↓ user reads decisions log; invokes /orchestrator when ready
(canonical pipeline runs from here)
```

Four artifacts ship before the orchestrator is invoked:

1. `docs/<name>-business-proposal.md` — stakeholder-facing
2. `docs/<name>-business-brief.md` — structured handoff to `spec-writer` (consumed by `spec-writer-brief-driven`)
3. `docs/<name>.md` — the 7-section spec
4. `docs/<name>-decisions.md` — every choice `spec-writer-brief-driven` made silently, with rationale + override path

The user has exactly **one review gate** in this flow: the business proposal. After confirmation, brief generation through `spec-writer-brief-driven` runs silently. The decisions log is the safety net — anything inferred without asking is auditable there before the user invokes `/orchestrator`.

**Why a separate intake skill instead of `spec-writer`'s interactive business track?** Business users get exhausted by the technical questions in `spec-writer`'s interactive interview (criticality classification, platform selection, env vars, etc.). `business-intake` asks plain-language questions only, then `spec-writer-brief-driven` (on Opus) makes every technical choice from the captured signals.

**The Tricentis portal layers in everything that isn't business logic.** When `business-intake` writes the brief with `tricentis_portal: true`, `spec-writer-brief-driven` omits SSO middleware and `.env.example` scaffolding from the spec — the portal adds them after the pipeline produces the repo. This keeps the spec focused on what's unique to this application instead of re-specifying the standard ops layer every time. Non-portal flows (technical users running `/spec-writer` directly, or business users on non-Tricentis seeds) follow the normal rules — SSO and `.env.example` are in scope.

**Model split** (locked in `docs/business-intake-todo.md`): `business-intake` and `spec-writer`'s brief-driven wrapper run on the session's default model (typically Sonnet). The actual section-by-section reasoning runs on the `spec-writer-brief-driven` subagent pinned to Opus via its frontmatter. Concentrating Opus where decisions are irreversible (criticality tier, architecture pattern, success criteria, failure modes) keeps cost down on the cheap conversational layers.

## Authority Model (Critical — Don't Violate)

Every change the self-update agent proposes lands in **two places**:
- **Locally**, according to the authority tier in the table below
- **`docs/self-update-<date>.md`** — a JIRA-ready artifact that's filed back to the team so the team's source repo can be updated and the next user's seed includes the change

| File type | Local action | In JIRA artifact? |
|---|---|---|
| Skills (`.claude/skills/`) | Apply directly to local file | **Yes** — source-bound |
| Reference docs (`.claude/docs/`) | Apply directly to local file | **Yes** — source-bound |
| Agent definitions (`.claude/agents/*.md`) | Stage in `self-update/<date>-<desc>` git branch — do NOT commit to main | **Yes** — source-bound |
| Commands (`.claude/commands/*.md`) | Stage in `self-update/<date>-<desc>` git branch — do NOT commit to main | **Yes** — source-bound |
| Settings (`.claude/settings.json`) | Stage in `self-update/<date>-<desc>` git branch — do NOT commit to main | **Yes** — source-bound |
| Rules (`CLAUDE.md`) | Stage in `self-update/<date>-<desc>` git branch — do NOT commit to main | **Yes** — source-bound |
| Specs (`docs/*.md`) | Apply directly to local file | **No** — specs are user-owned, do not propagate to source |

This rule overrides any spec or task that asks for direct commits to `.claude/agents/*.md`, `.claude/commands/*.md`, `.claude/settings.json`, or `CLAUDE.md`. Stage them in a branch and add them to the JIRA artifact — never auto-commit.

`.claude/settings.local.json` is **not** in this table — it's gitignored and user-owned. Personal/path-specific allows live there and never round-trip to source.

The two-tier local model exists so skills/specs improvements are usable in the rest of the current session, while protected files stay frozen until a human approves. The JIRA artifact exists so neither tier dies with the local repo — every learning flows back to source.

## Design Principles

Five principles shape every decision in this pipeline. When something feels ambiguous, fall back to these:

1. **Specs reference capability, not specific agents.** Write "executor capability" or "review capability," not `ExecutorAgent` by name. This keeps specs durable as the agent roster evolves — Opus maps capability → agent at runtime. The spec-writer skill enforces this in Section 5 (Task Decomposition).

2. **Success criteria must be measurable.** If Opus can't run a check that returns pass or fail, it's not a criterion — it's a wish. "Performs well" is rejected by the PM agent; "p95 latency under 200ms" is accepted. The validate agent's Step 5 verification only counts a criterion as `VERIFIED` if it produced an exit-code-0 evidence trail.

3. **Failure modes are what make this D3.** D2 specs tell agents what to do. D3 specs tell agents what to do when things go wrong. A spec with no Section 4 failure modes is a D2 spec masquerading as D3 — bounce it back.

4. **The self-update agent is the D3/D4 boundary.** Autonomous changes to rules and agent definitions would make this D4. Human sign-off — via the JIRA ticket back at the source repo — is what keeps us at D3. Never weaken this. Not for convenience, not for speed, not because "the change is small."

5. **Staging is dual: git branch + JIRA artifact.** Proposed changes to protected files live in `self-update/<date>-<desc>` branches locally AND in `docs/self-update-<date>.md` for JIRA. The local branch keeps the current session unblocked while the change is unapplied; the JIRA artifact carries the change back to the source repo so the next user's seed includes it.

## File Conventions

### Agent files (`.claude/agents/*.md`)
YAML frontmatter is mandatory and structurally significant:

```yaml
---
name: <agent-name>                 # invocable as /<name> in some cases
description: <when-to-invoke>      # used by the orchestrator/Agent tool for routing
allowed-tools: <tool list>         # narrowest set that lets the agent do its job
---
```

The `allowed-tools` line is what makes the pipeline runnable under `bypassPermissions` — the tester and reviewer agents enumerate specific `Bash(npm:*)`, `Bash(pytest:*)`, etc. entries on purpose. Widening these to bare `Bash` defeats the safety boundary; narrowing them past what the agent needs stalls the pipeline (see the pre-flight check in `.claude/agents/orchestrator.md`).

### Skill files (`.claude/skills/<name>/SKILL.md`)
Frontmatter requires `name` and `description`. The `description` is what triggers the skill — write it to include the natural-language phrases a user would actually say (the spec-writer description is the model).

A skill folder may contain `README.md` (human-facing) and `EXAMPLE.md` alongside `SKILL.md`. Only `SKILL.md` is loaded by Claude Code.

## Capability Matrix

Specs name capabilities; Opus binds capability to agent at runtime. The current mapping:

| Capability | Skill / source | Agent type | Notes |
|---|---|---|---|
| Write code | — | executor (+ mid-level-engineer) | Output must be validated by tester |
| Test code | — | tester | Detects runner from repo signals |
| Review code | — | reviewer | Quality, patterns, security, CLAUDE.md compliance |
| Capture business requirements | `business-intake` | (skill, no agent) | Tricentis-flow entry for non-technical users. Plain-language interview → business proposal + brief → invokes `spec-writer` with `--brief`. |
| Write a spec | `spec-writer` | `spec-writer-brief-driven` (brief-driven mode only) | Interactive tracks (technical / business): section-by-section interview in the skill, no agent. Brief-driven mode: skill wrapper spawns `spec-writer-brief-driven` (pinned to Opus via agent frontmatter) for silent reasoning. Both modes produce `docs/<name>.md` + `docs/<name>-decisions.md`. |
| Pick architecture pattern | `architecture-pattern-selector` | (skill, no agent) | Matches workload signals (consumer, UI needed, kind of work, data shape, who triggers) against the pattern catalog. Used by `spec-writer-brief-driven` and by the interactive business track. Supports multi-pattern pairings. |
| Report session usage | `usage-report` | (skill, no agent) | Generates a post-session report with time, tokens, cost, per-phase breakdown, and team-specific recommendations. Reads from `/usage` (user pastes the output). Writes `docs/usage-<YYYY-MM-DD>-<short-name>.md`. |
| Validate scope | — | pm | Returns READY / NEEDS CLARIFICATION / OUT OF SCOPE |
| Design approach | — | architect | Produces file plan + interface contracts |
| Run build pipeline | — | orchestrator | Runs pm → architect → executor → README. No runtime required. Invoked by `/orchestrator`. |
| Run validation pipeline | — | validate | Runs tester → reviewer → self-update. Requires a runtime environment. Invoked by `/validate`. |
| Audit system | — | self-update | Stages protected changes + writes JIRA artifact |
| File a JIRA ticket | — | jira-ticket (slash command) | User-invoked via `/jira-ticket docs/self-update-<date>-<slug>.md`. Not auto-invoked by orchestrator. |

When a new agent is added to the source repo, a row is added here so specs have a stable capability name to bind against.

## The 7-Section Spec Format

Every spec consumed by the orchestrator must have these sections in order: Intent, Context, Success Criteria, Failure Modes, Task Decomposition, Decision Points, Handoff Protocol. The orchestrator's Step 1 pre-flight check will reject specs missing any of them.

Specs are written to `docs/<spec-name>.md` by the spec-writer skill. When modifying anything in `.claude/skills/spec-writer/`, keep the section names and order identical — the PM, architect, tester, and reviewer agents all reference sections by number (e.g., "Section 3 success criteria", "Section 4 failure modes").

Section 2 (Context) must include **build environment prerequisites** for any code-generation spec — runtime versions, package managers, network/permission requirements. The orchestrator probes these at startup and halts before writing files if anything is missing. This exists because of a real prior failure: 62 files written before discovering Node v14 active when Vite required Node 18+.

## UI Stack (when the spec involves a UI)

The pipeline assumes this stack for any UI work unless the spec explicitly overrides it:

- **Framework:** React 18+
- **Language:** JavaScript (`.jsx`) — TypeScript migration planned, not started
- **Build tool:** Vite
- **Test runner:** Vitest (Jest-compatible API; zero-config with Vite); Playwright for E2E tests
- **State management:** React `useState` / `useReducer` only — no external state libraries (Redux, Zustand, Jotai, etc.)
- **Icons:** `lucide-react` only — no other icon libraries
- **Design system:** Aura (light mode, Inter font, Tricentis production design language). All colors, radii, fonts, and spacing values must come from the `T` tokens object defined in the app — never hardcode hex values in components. The reviewer flags any hardcoded hex codes or inline styles that don't use Aura tokens as required changes.

**Canonical scaffold command:**

```bash
npm create vite@latest <project-name> -- --template react
cd <project-name> && npm install
```

**Runtime environment (three tiers, automatically detected at `/validate` pre-flight):**

The pipeline needs *somewhere* to run build commands (`npm install`, `pytest`, `mypy`, etc.). The `/validate` command's Step 2 detects the best available tier on the user's machine. Runtime detection does NOT happen in `/orchestrator` — the build phase only writes files and requires no runtime.

1. **Docker dev container (Tier 1)** — the `Dockerfile` and `docker-compose.yml` at the repo root provide a Node 20 + Python 3.13 + `uv` container. The user runs `docker compose up -d` and the agent wraps build commands with `docker compose exec -T dev <command>`. Preferred for consistency across machines.

2. **Host-native tools (Tier 2)** — if Docker isn't installed but the host has the tools the spec needs (Node 20+, Python 3.13+, `uv`), the agent runs build commands directly on the host shell. Common case for developers with prior toolchain installs.

3. **GitHub Codespaces (Tier 3)** — if neither Docker nor host tools are available, `/validate` halts at Step 2 and surfaces a Codespaces URL. The repo ships `.devcontainer/devcontainer.json` pre-configured (Node 20, Python 3.13, `uv`, Claude Code extension auto-installed) so Codespaces opens with everything ready in ~30 seconds. **This is the no-install path — the canonical Tricentis business-user case.**

The validate agent receives the chosen tier via Step 3's briefing and adjusts its command wrappers accordingly. Validation pre-flight runs inside whichever environment was selected (the Tier 1 container, the host shell, or — for Tier 3 — the Codespace after the user re-invokes `/validate`).

**Dev servers must bind to `0.0.0.0`, not `localhost`.** Inside the Docker dev container, binding to `localhost` (Vite's default, and the default for many frameworks) listens only on the container's loopback — the host's browser can't reach the published port even though `docker-compose.yml` maps `5173:5173`. Scaffolded `vite.config.js` must include `server.host: '0.0.0.0'`. Any uvicorn or other Python HTTP server in the quick-start instructions must use `--host 0.0.0.0`. The reviewer flags any dev config or README quick-start that binds to localhost as a required change.

**Scaffold footguns surfaced by real pipeline runs:**

- Keep `vite.config.js` and `vitest.config.js` as **separate files**. Importing `defineConfig` from `vite` while declaring a `test:` block triggers a config cascade because Vitest bundles its own copy of Vite. `vite.config.js` uses `from 'vite'`; `vitest.config.js` uses `from 'vitest/config'`. The `npm test` script points at `vitest run` which auto-discovers the latter.

**Search inputs must debounce at 300ms** using `useRef` for the timer — no lodash or external debounce library. Require a minimum of 2 characters before firing a search request.

**The Aura `T` tokens object is mandatory before UI work.** If the tokens object is missing or incomplete, pm returns `NEEDS CLARIFICATION`. The reviewer enforces that all style values reference `T.*` — hardcoded hex codes or values that bypass the tokens object are required changes.

## Python Stack (when the spec involves backend work — API, MCP, scripts)

The pipeline assumes this stack for any Python work unless the spec explicitly overrides it:

- **Language:** Python 3.13 — use modern syntax (`match`, type hints, `|` union types)
- **Package manager:** `uv` — fast, modern; replaces pip + pip-tools + virtualenv. Never use `pip` directly.
- **HTTP client:** `httpx` — async-capable, modern replacement for `requests`
- **Test framework:** `pytest`
- **Linter/formatter:** `ruff` (configured in `pyproject.toml`)
- **Type checker:** `mypy` (or `pyright`) — strictness is decided by the architect agent per project based on spec characteristics; users don't pick this
- **API framework (when an API layer is needed):** **FastAPI** (async, type-hint-driven, auto-generates OpenAPI)
- **MCP SDK (when an MCP server is needed):** **Anthropic MCP Python SDK** — the `mcp` package; not FastMCP

**Typing requirements:** annotate all public function signatures (parameters + return types). Add docstrings to all public functions. Internal helpers and one-off scripts can rely on inference; reviewer doesn't reject them.

**Standard library imports ready in every Python project** (no install needed): `asyncio`, `os`, `json`, `logging`, `platform`, `sys`. Plus `httpx`, `pytest`, `mypy`, `ruff` pre-installed globally in the dev container.

**Canonical scaffold for a new Python project:**

```bash
mkdir my-service && cd my-service
uv init
uv add httpx                         # always
uv add --dev pytest ruff mypy        # always
# For API work:
uv add fastapi uvicorn
# For MCP work:
uv add mcp
```

**`mypy` mode is decided by the architect agent based on spec characteristics**, not by the user. Default is non-strict for low-stakes code (prototypes, demos, internal scripts); strict mode kicks in automatically when the spec touches money, auth, PII, data migrations, library/SDK code, or compliance requirements. See the "Decisions the Agents Make" section below for the inference rules.

**FastAPI footguns surfaced by real pipeline runs:**

- Declaring a form field as `field: str = Form(...)` makes FastAPI return **422 with its generic validation error** when the field is empty — short-circuiting any spec-mandated 400 message in the handler body. When the spec says "reject with this exact message," declare the field as `Form("")` (default-empty) and validate inside the handler so your 400 + message reaches the client.
- `ruff` flags FastAPI's `Depends(...)`, `File(...)`, `Form(...)`, `Cookie(...)`, `Query(...)` in argument defaults with `B008`. Those are the **idiomatic** way to declare dependencies and must appear in defaults. Add `B008` to `[tool.ruff.lint] ignore` in the backend's `pyproject.toml`.

## API vs MCP Server — which does the spec need?

When the spec involves a backend service, this question must be answered before the architect agent designs the solution. The decision shapes scaffolding, dependencies, success criteria, and the test surface.

The spec-writer skill asks this during the interview whenever backend work is in scope. Don't proceed past Section 5 (Task Decomposition) with a vague "we need a backend."

### Build an API layer (FastAPI) when…
- Consumers are **humans via a web app, mobile app, or third-party integration**
- The workload is request/response: client sends data, server returns data
- You need stateful or long-running endpoints (uploads, jobs, streams)
- Standard REST/JSON or GraphQL patterns fit the use case

### Build an MCP Server (Anthropic `mcp` SDK) when…
- The consumer is an **AI agent** (Claude, GPT, an internal copilot, etc.)
- You want tool-use semantics — the agent calls discrete typed functions
- You're exposing internal capabilities (database access, file ops, domain queries) to an AI assistant
- The interaction is conversational/contextual rather than transactional

### Build both when…
- The **same business logic** serves both a UI/external app AND an AI agent
- Pattern: a shared `core/` module with all the logic, plus thin FastAPI endpoints and thin MCP tools that both call into it
- Common in productized AI features where humans and agents consume the same capabilities

### Build neither when…
- It's a CLI tool, batch script, or scheduled job — write it as a Python script with an `if __name__ == "__main__"` entry point, packaged via `uv`
- It's internal Python utility code with no external consumer

## Environment Configuration

**Secrets never live in code or environment variables.** API keys, tokens, credentials, and connection strings go to **AWS Secrets Manager** (for AWS-hosted components) or **Azure Key Vault** (for Azure-hosted components). Never hardcode them; never put them in `.env` files that get committed or injected as env vars in production.

**Non-secret configuration** (service URLs, feature flags, model identifiers, timeouts, ports) lives in environment variables loaded via `.env` for local development.

### How it works

- **`.env.example`** — committed to git. Documents every *non-secret* config variable the project needs, with placeholder values and a one-line comment. For secrets, the comment explains where to obtain them from the secret manager — no placeholder values. This is the handoff guide for the team.
- **`.env`** — gitignored. Contains real non-secret config for local development. Never committed. Never contains actual secrets.
- **Loading:**
  - **Python:** use `pydantic-settings` — defines a typed `Settings` class that reads from env. The reviewer rejects raw `os.environ.get("KEY")` for non-trivial config; type-checked settings are the standard.
  - **React + Vite:** use `import.meta.env.VITE_*` (Vite's built-in env handling). Only `VITE_`-prefixed vars are exposed to client code — anything else is server-only. Never put secrets in `VITE_*` vars — they are bundled into the client.
- **`.gitignore`:** the seed repo's `.gitignore` already excludes `.env`, `.env.local`, and `.env.*.local`. Don't weaken this.

### Multi-component projects: `.env` path resolution

When the project has multiple Python components in subdirectories (`job/`, `api/`, etc.) sharing a single root-level `.env`, **never use a bare `env_file: ".env"`**. That string resolves relative to the process's working directory — so `cd job && uv run python main.py` looks for `job/.env`, which doesn't exist, and pydantic-settings silently skips loading it, causing a `ValidationError` on required fields at startup.

**Always resolve relative to `__file__`:**

```python
from pathlib import Path

_HERE = Path(__file__).parent
# Prefer a component-local .env if present; fall back to the repo root.
_ENV_FILE = str(_HERE / ".env") if (_HERE / ".env").exists() else str(_HERE.parent / ".env")

class Settings(BaseSettings):
    model_config = {"env_file": _ENV_FILE, "env_file_encoding": "utf-8"}
```

### What goes in `.env.example`

Non-secret config vars only:
- Service URLs that differ per environment (`API_BASE_URL`, etc.)
- Feature flags that toggle behavior
- Model identifiers (e.g. `ANTHROPIC_MODEL=claude-opus-4-7`)
- Rate limits / timeouts that are tuned per deployment
- Ports

For each secret the project needs, add a comment-only entry pointing to the vault:

```bash
# Bedrock access — provided via Lambda IAM execution role, no key needed
# DB connection string — retrieve from AWS Secrets Manager: /myapp/prod/db-url
```

If a value is genuinely a constant (e.g. `PI = 3.14159`) it stays in code.

### Reviewer checks

- No hardcoded secrets in source (the reviewer scans for API key patterns, tokens, etc.)
- No secrets in env vars or `.env` files — they must come from the secret manager at runtime
- No hardcoded URLs that should be config-driven (anything ending in `.com`, `.io`, etc. in production code paths)
- Python: `os.environ.get()` raw access flagged in favor of pydantic-settings
- Every config variable referenced in code MUST appear in `.env.example`

### What pm asks during the interview

When backend or external-integration work is in scope, pm should require the spec to enumerate the configurable values and secrets the project will need. The spec-writer skill asks this during Section 2 (Context) — the answer populates the initial `.env.example`.

## Decisions the Agents Make (Not the User)

The users running this pipeline are not expected to know what `mypy --strict` does or how to set test coverage thresholds. These are toolchain choices whose tradeoffs only matter to engineers. **Don't ask the user. Infer from the spec.**

The architect agent makes these calls during design (Step 4 of its process) and records each choice with a one-line reason in its design output so the reviewer and the team can see what was decided and why.

### Inference: `mypy` strict mode

**Default: non-strict.** Strict mode adds friction (rejects untyped third-party imports, requires annotations on every helper) that's only worth paying when type bugs are expensive.

**Upgrade to strict (`mypy --strict` or `pyright --strict`) when the spec shows any of:**

| Signal in the spec | Why strict pays off |
|---|---|
| Money, billing, payments, currency, invoicing, refunds, accounting | Type mismatch on a money calc is real damage |
| Auth, login, session, JWT, OAuth, password handling, permissions, RBAC | Type bugs in security paths bypass security |
| PII, healthcare, financial records, encryption, sensitive data | Compliance + privacy stakes |
| Data migrations, schema changes, ETL, backfills | Wrong types corrupt data permanently |
| "Library", "SDK", "package", or code other projects will import | Public surface deserves type guarantees |
| Compliance (GDPR, HIPAA, SOC 2, PCI, etc.) | Auditors will look for this |

**If two or more weak signals appear** (e.g., external API integration + database writes), lean toward strict.

**Stay default when the spec says:** "prototype", "demo", "POC", "spike", "experiment", "throwaway", "internal script." These are the explicit anti-signals.

### Inference: test coverage and rigor

The default test rigor follows the spec's Section 3 (Success Criteria) and Section 4 (Failure Modes). The architect doesn't impose an arbitrary "80% coverage" threshold — instead:

- **Every Section 4 failure mode must have at least one test.** The tester agent already enforces this.
- **High-stakes specs** (same signals as strict mypy above) get edge-case coverage as a required-change item in review.
- **Exploratory specs** ship with happy-path coverage plus whatever Section 4 demands; no broader coverage targets.

### Inference: mockable dev defaults for hosted-platform specs

**When the spec targets a hosted platform** (Web Container App, AKS, scheduled-job platform, static-site host, etc.), the architect picks **mock/stub-able defaults at the `dev` tier** so the executable definition of done can run inside the dev container without provisioning external systems. Production uses real providers; dev uses local equivalents, swapped via env vars. The Skill Sharing Portal is the canonical example: mock SSO route, SQLite, filesystem storage in dev; real SSO + managed DB + blob storage in prod.

For Tricentis seed-repo work, the platform itself is layered in by the portal after the pipeline produces the repo. The architecture pattern (interactive dashboard, content website, integration service, data + automation service) is picked by `architecture-pattern-selector`; the architect's job is to pair that pattern with the right mockable dev defaults so the pipeline's executable definition of done is runnable inside the dev container.

### The principle

If a choice is between **two reasonable engineering defaults** whose differences a non-engineer can't evaluate, the architect picks. If a choice has **business or product implications** (which features to include, what the UI should look like, what's success vs failure), the user decides. The spec-writer interview is for the second kind; toolchain defaults are not.

When in doubt: ask if the user could meaningfully answer the question. If not, decide for them and document the choice.

## Container Standards

All containerized components must follow these rules — the reviewer flags violations as required changes:

- Multi-stage builds — separate build stage and runtime stage
- Base images: official images only (e.g. `node:lts-alpine` for build, `nginx:alpine` for serve)
- Pin base image versions — no `:latest` tags
- Never run containers as root — use a non-root `USER` directive
- Never include secrets, `.env` files, or credentials in container images
- Never include `node_modules`, `.git`, or dev dependencies in the runtime image
- Use `.dockerignore` to exclude unnecessary files
- Health check endpoint required for Azure Container Apps probes

## Git & Commit Standards

All commits follow conventional commits format:

```
type(scope): description

Examples:
  feat(chat): add typing indicator
  feat(lambda): implement bedrock streaming
  fix(ui): correct token color reference
  chore(deps): pin boto3 to 1.34.0
```

Never commit directly to `main` — all changes via PR. Frontend changes (Green zone) require automated scan + light review; Chat API changes (Yellow zone) require automated scan + peer review.

## Security & Logging Rules

These rules apply to all generated code — violations are required changes in review:

- NEVER hardcode secrets, API keys, tokens, or credentials
- NEVER use `eval()`, `exec()`, or dynamic code execution
- ALWAYS use parameterized queries — no string concatenation for SQL
- ALWAYS validate and sanitize all user inputs
- ALWAYS use established libraries for auth — never roll custom authentication or session management
- ALWAYS encode output to prevent XSS
- NEVER log PII, secrets, message content, or sensitive data
- NEVER log user messages, assistant responses, or system prompt content
- NEVER use `pickle` or insecure deserialization
- NEVER expose internal error details (stack traces, AWS account info, etc.) to the frontend — return generic messages only
- Use HTTPS for all external calls
- Apply principle of least privilege for all IAM/permissions
- If a security pattern is uncertain, flag it with `// SECURITY-REVIEW` (JS) or `# SECURITY-REVIEW` (Python)

## Cross-File Invariants to Preserve

When editing one file, these are the things most likely to silently break in another:

- **Pipeline order**: `.claude/agents/orchestrator.md` lists `pm → architect → executor → README` (build phase). `.claude/agents/validate.md` lists `tester → reviewer → self-update` (validation phase). Each agent's "Handoff" section names the next agent. Changing one without the others creates a dead-end handoff.
- **Section numbers**: tester references "Section 3" (success criteria) and "Section 4" (failure modes). Reviewer references "Section 6" (decision points). Renumbering the spec format breaks all of them.
- **Allowed-tools and validate pre-flight**: if a new runner (e.g., `bun test`) is added to the tester's allowed-tools, the detection signal in the tester's Step 0 and the corresponding pre-flight probe in the `/validate` command must be added too.
- **Authority table**: appears in this `CLAUDE.md` and in `.claude/agents/self-update.md`. Both must agree.
- **JIRA artifact format**: the schema of `docs/self-update-<date>.md` is consumed by `.claude/commands/jira-ticket.md`. If one changes, the other must too.
