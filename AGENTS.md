# AGENTS.md — LiteLLM (fork)

This file is the **operational playbook** for agentic coding in this repository.
It is grounded in the repo’s actual configs/scripts (Makefile, pyproject, ruff, CI workflows, UI package.json).

## Repo map (high-signal)
- `litellm/` Python SDK + provider adapters
- `litellm/proxy/` Proxy server (AI Gateway)
- `tests/` pytest suites (unit + integration buckets)
- `ui/litellm-dashboard/` Next.js dashboard (TypeScript, Vitest, Playwright)
- `enterprise/` enterprise extras

## Build / lint / test commands

### Python (root)
**Install**
- `make install-dev` (dev deps via Poetry)
- `make install-proxy-dev` (proxy dev deps)
- CI parity installs:
  - `make install-dev-ci`
  - `make install-proxy-dev-ci`

**Format + lint (CI parity)**
- `make format` (Black apply)
- `make format-check` (Black check)
- `make lint` (format-check + Ruff + MyPy + circular import + import-safety)
- Faster local loop: `make lint-dev` (only changed formatting + MyPy + safety checks)

**Tests**
- Full: `make test` (pytest `tests/`)
- Unit (common default): `make test-unit` (pytest `tests/test_litellm -x -vv -n 4`)

**Run a single test** (pytest)
```bash
# file
poetry run pytest tests/test_litellm/path/to/test_file.py -vv

# single test
poetry run pytest tests/test_litellm/path/to/test_file.py::test_name -vv

# pattern
poetry run pytest tests/test_litellm -k "pattern" -vv
```

**CI matrix buckets (Makefile targets)**
- `make test-unit-llms`
- `make test-unit-proxy-guardrails`
- `make test-unit-proxy-core`
- `make test-unit-proxy-misc`
- `make test-unit-integrations`
- `make test-unit-core-utils`
- `make test-unit-other`
- `make test-unit-root`
- `make test-proxy-unit-a` / `make test-proxy-unit-b`

### UI dashboard (Next.js)
Location: `ui/litellm-dashboard/`

```bash
cd ui/litellm-dashboard
npm install
npm run dev
npm run build
npm run lint
npm run format
npm run test
npm run e2e
```

**Run a single UI test** (Vitest)
```bash
cd ui/litellm-dashboard

# file
npm run test -- src/components/foo/foo.test.tsx

# by name/pattern
npm run test -- -t "should render"
```

## Code style (enforced by config)

### Python
- **Formatter**: Black (run via `make format`, checked in `make lint`).
- **Line length**: 120 (`ruff.toml`).
- **Lint**: Ruff (`make lint-ruff`). `ruff.toml` excludes `tests/*` and some generated/types paths.
- **Import order**: isort with `profile = "black"` (`pyproject.toml`). Prefer:
  1) stdlib 2) third-party 3) `litellm.*` local imports.
- **Typing**: MyPy (`make lint-mypy`) is intentionally permissive (see `litellm/mypy.ini`):
  - `ignore_missing_imports = True`
  - pydantic mypy plugin enabled (`pyproject.toml`).
- **No “unprotected imports”**: `make check-import-safety` runs `from litellm import *`.
  - If you add optional dependencies, guard imports and keep module importable.
- **Exceptions**: follow existing patterns in `litellm/exceptions.py` (carry `llm_provider`, `model`, `response` context).
- **Avoid**: bare `except:`, silent `except Exception: pass`, and logging secrets.

### UI (TypeScript/React)
- Formatting: Prettier (`npm run format` / `format:check`).
- Testing: Vitest + React Testing Library.
  - Use `screen.*` queries.
  - Prefer semantic queries: `getByRole` → `getByLabelText` → `getByPlaceholderText` → `getByText` → `getByTestId`.
  - Tests are conventionally named `it("should ...")` (see existing `*.test.tsx`).
- Tremor: avoid introducing new Tremor components where possible; test harness mocks some Tremor components
  (`ui/litellm-dashboard/tests/setupTests.ts`).

## Testing conventions
- Unit tests for Python changes go in `tests/test_litellm/` and mirror `litellm/` structure (see `CONTRIBUTING.md`).
- Prefer mocked tests for unit coverage (avoid real provider API calls in unit tests).
- Pytest config is in `pyproject.toml` (`asyncio_mode=auto`, retries enabled, markers like `no_parallel`).

## Cursor / Copilot instructions
- No `.cursor/rules/`, `.cursorrules`, or `.github/copilot-instructions.md` were found in this repo at scan time.

## Git workflow for this fork (重要 / fork 合并逻辑)
This repo is a **fork**. Local primary branch is `main`.

### One-time upstream setup
```bash
git remote add upstream https://github.com/BerriAI/litellm.git
git fetch upstream
```

### Merge logic (以指定上游分支为基准)
Given a target upstream base branch (default `upstream/main`):
1) **Use upstream as base**: `BASE=upstream/<branch>`
2) **Check whether local-only commits already exist in BASE**
```bash
git fetch upstream

# commits that are on local main but not in BASE
git log --oneline --cherry $BASE...main

# alternative view
git cherry $BASE main
```
3) If there are local-only commits, integrate them onto the base:
- Preferred (linear): `git checkout main && git rebase $BASE`
- If you must preserve commit selection: create a feature branch from `$BASE` and `git cherry-pick <sha...>`.

### Safety
- Do not rewrite shared history.
- Avoid force-pushing to `main` unless you own the remote branch and understand the impact.
- After rebasing/merging, run `make lint` + `make test-unit` before opening a PR.

### Fork-specific constraints (THIS FORK ONLY)
When merging upstream stable branches, the following local commits MUST be preserved:

1. **CI Workflow Optimization** (commit: `1e1e7d8bba`)
   - Removes `docker-hub-deploy` dependency from helm and release jobs
   - Required for: Fork仓库独立使用GitHub Actions构建Docker镜像
   - File: `.github/workflows/ghcr_deploy.yml`

2. **Provider api_base Support** (commit: `ad729c6f00`)
   - Exposes `api_base` credential field for Anthropic and Gemini providers
   - Required for: 支持自定义API端点配置
   - File: `litellm/proxy/public_endpoints/provider_create_fields.json`

3. **Vertex AI Role Fix** (commit: `d4973d87f3`)
   - Adds explicit role to tool call response ContentType
   - Required for: Vertex AI provider正确性
   - File: `litellm/llms/vertex_ai/gemini/transformation.py`

4. **OpenAI Pricing Updates** (commit: `bd4d189348`)
   - Adds gpt-5.3-codex-spark pricing and registry updates
   - Required for: 新模型定价支持
   - Files: `model_prices_and_context_window.json`, `litellm/llms/openai/responses/transformation.py`

5. **AGENTS.md Documentation** (commit: `472a87ae71`)
   - Fork-specific operational playbook
   - Required for: Agent工作流一致性
   - File: `AGENTS.md`

**Merge Checklist for Future Upstream Syncs:**
- [ ] Verify all 5 fork-specific commits are preserved or re-applied
- [ ] Check for duplicate keys in pricing JSON files (critical data integrity issue)
- [ ] Validate CI workflow dependencies remain decoupled from Docker Hub
- [ ] Confirm api_base fields present for Anthropic and Gemini
- [ ] Run `make lint` + `make test-unit` before force-pushing to main
