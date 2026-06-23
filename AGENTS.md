# AGENTS.md — CatchTheVibe Vibe Coding Standards
# This file is read by AI coding assistants (Cursor, Claude Code,
# Copilot, Codex, Windsurf) at the start of every session.
# It ensures all generated code follows Tricentis standards.

## Project Overview
# CatchTheVibe is Tricentis's internal vibe coding governance portal.
# It is a mono-repo with two workspaces:
#   1. Frontend — React 18+ SPA (JavaScript/JSX) using the Aura design system
#   2. Chat API — Python 3.13 Lambda backend proxying to Amazon Bedrock
# Zone classification: Frontend is Green, Chat API is Yellow.
# All access is behind Tricentis SSO. No PII stored or processed.

## Approved Tech Stack

### Frontend (repo root: src/)
- Framework: React 18+ (functional components, hooks only)
- Language: JavaScript (.jsx) — TypeScript migration planned, not started
- Styling: Inline styles only using Aura design tokens (T object in App.jsx)
- Design system: Aura (light mode, Inter font, Tricentis production design language)
- Icons: lucide-react — no other icon libraries
- State: React useState/useReducer only — no external state management
- Build: Vite (npm run dev / npm run build)
- Container: nginx serving Vite build output, deployed to Azure Container Apps
- Dockerfile: Multi-stage build (Node for build, nginx for serve) in repo root
- Testing: Vitest for unit tests, Playwright for E2E (when added)
- Architecture: Single-file — all components live in src/App.jsx

### Chat API (chat-api/ subdirectory)
- Language: Python 3.13 (latest stable LTS)
- Package manager: uv — NOT pip. All dependency management uses uv exclusively.
- Dependency spec: pyproject.toml with pinned versions
- Lockfile: uv.lock — committed to source control, deterministic builds
- Runtime: AWS Lambda (Python 3.13 managed runtime)
- AI model access: Amazon Bedrock via boto3 (invoke_model_with_response_stream)
- Authentication: Lambda IAM execution role — no API keys
- Network: AWS PrivateLink (VPC Interface Endpoint) to Bedrock — no public internet
- Linter/formatter: ruff (configured in pyproject.toml)
- Testing: pytest with moto for AWS service mocking

### Infrastructure
- Cloud: AWS (Lambda, Bedrock) + Azure (Container Apps, APIM, App Gateway)
- IaC: OpenTofu only — no Bicep, no manual console changes
- Containers: Azure Container Apps (serverless) with Azure Container Registry (ACR)
- Access: Private-only via Zscaler Private Access (ZPA) — no public endpoints
- Secrets: AWS Secrets Manager / Azure Key Vault — no secrets in code or env vars
- CI/CD: GitHub Actions

### Container Standards
- Dockerfile must use multi-stage builds — build stage and runtime stage separate
- Base images: use official images only (node:lts-alpine for build, nginx:alpine for serve)
- Never run containers as root — use non-root USER directive
- Never include secrets, .env files, or credentials in container images
- Never include node_modules, .git, or development dependencies in runtime images
- Use .dockerignore to exclude unnecessary files
- Pin base image versions — no :latest tags
- Health check endpoint required for Container Apps probes

## Security Requirements — MANDATORY
- NEVER hardcode secrets, API keys, tokens, or credentials
- NEVER use eval(), exec(), or dynamic code execution
- ALWAYS use parameterized queries — no string concatenation for SQL
- ALWAYS validate and sanitize all user inputs
- ALWAYS use established libraries for auth — never roll custom
  authentication or session management
- ALWAYS encode output to prevent XSS
- NEVER log PII, secrets, message content, or sensitive data
- NEVER log user messages, assistant responses, or system prompt content
- NEVER use pickle or insecure deserialization
- NEVER expose internal error details (stack traces, Bedrock errors,
  AWS account info) to the frontend — generic messages only
- Use HTTPS for all external calls
- Apply principle of least privilege for all IAM/permissions
- If unsure about a security pattern, flag it with a
  // SECURITY-REVIEW (JS) or # SECURITY-REVIEW (Python) comment

## Coding Standards

### JavaScript (Frontend)
- Use const over let; never use var
- Prefer arrow functions for components and handlers
- Use destructuring for props and state
- Keep components under ~100 lines — extract when they grow
- Use early returns to reduce nesting
- Meaningful variable names — avoid abbreviations except e, i, idx
- All colors, radii, fonts, and spacing values must come from the
  T (tokens) object — never hardcode hex values in components

### Python (Chat API)
- Python 3.13 — use modern syntax (match, type hints, | union types)
- Type hints on all function signatures
- Docstrings on all public functions
- Async where appropriate (Lambda handler, Bedrock streaming)
- Small single-purpose functions
- Use early returns to reduce nesting
- ruff for formatting and linting — follow pyproject.toml config

## Dependencies

### Frontend (npm)
- Only use packages available in the Tricentis private registry
- Pin all dependency versions — no floating ranges (^, ~)
- Check for known CVEs before adding any new package
- Prefer well-maintained packages with active communities
- If you are unsure a package exists, say so — do not guess
  or hallucinate package names
- Currently approved: react, react-dom, lucide-react, vite

### Chat API (uv)
- All dependency management through uv — NEVER use pip
- Dependencies defined in chat-api/pyproject.toml
- Lockfile: chat-api/uv.lock — MUST be committed to source control
- In CI: uv sync --frozen (fails if lockfile is stale)
- Pin all versions — no floating ranges
- Currently approved: boto3
- Dev dependencies: pytest, pytest-asyncio, moto, ruff

## Architecture Boundaries
- This is a mono-repo — frontend (repo root) and backend (chat-api/) coexist
- Frontend and backend concerns are strictly separated
- The frontend sends { system, messages, max_tokens } to APIM — nothing else
- The Lambda translates to Bedrock format — the frontend never knows about Bedrock
- The Lambda authenticates to Bedrock via IAM role — no credentials cross boundaries
- Bedrock traffic stays on AWS PrivateLink — never traverses public internet
- APIM handles SSO validation and rate limiting — the Lambda does not
- Do not introduce new frameworks or major dependencies without explicit approval
- No direct public internet access from the Lambda except through PrivateLink

## Testing Requirements
- All new functions must include unit tests
- All API endpoints must include integration tests
- Test both success and failure/edge cases
- No tests that depend on external services without mocking
- Frontend: Vitest (when added)
- Chat API: pytest with moto for Bedrock mocking

## Git & Workflow
- One logical change per commit
- Commit messages: type(scope): description
  Frontend: feat(chat): add typing indicator
  Chat API: feat(lambda): implement bedrock streaming
- Never commit directly to main — all changes via PR
- Never commit .env files, secrets, or credentials
- Never commit chat-api/.venv/ or node_modules/
- Frontend changes (Green zone): automated scan + light review
- Chat API changes (Yellow zone): automated scan + peer review required

## What NOT To Do
- Do not make sweeping changes across multiple files unless explicitly asked
- Do not refactor unrelated code while implementing a feature
- Do not remove or weaken existing security controls
- Do not change infrastructure configuration without flagging it
- Do not assume — ask for clarification when requirements are ambiguous
- Do not use pip — use uv for all Python package management
- Do not run uv commands from the repo root — always cd chat-api first
- Do not create new React component files — everything stays in App.jsx
- Do not use CSS files, Tailwind, or styled-components — inline styles only
- Do not use localStorage or sessionStorage — not supported in deployment
- Do not log message content, user input, or assistant responses anywhere
