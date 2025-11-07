# Repository Guidelines

## Project Structure & Module Organization
- Root files: `.codexrc.yml` (Codex CLI config), `.env` (secrets), `package.json`, `package-lock.json`.
- Place source in `src/` (e.g., `src/index.js`, `src/utils/…`).
- Put tests in `tests/` (e.g., `tests/index.test.js`).
- Keep scripts/one‑offs in `scripts/` and static assets in `assets/` if needed.

## Build, Test, and Development Commands
- `npm install` — install dependencies.
- `npx codex` — run Codex CLI using `.codexrc.yml` (files like `tasks.md`, `tests.md` if present).
- `npx dotenv -e .env -- codex` — ensure env vars (e.g., API keys) are loaded when invoking Codex.
- `npm test` — placeholder today; configure a test runner (see below) to enable.

## Coding Style & Naming Conventions
- JavaScript/TypeScript: 2‑space indent, trailing commas where valid, prefer single quotes.
- Filenames: kebab‑case for modules (`data-loader.js`), PascalCase for classes, camelCase for variables/functions.
- Keep functions small and pure; colocate helpers under `src/<feature>/`.
- If adding tooling, use ESLint + Prettier; commit the configs.

## Testing Guidelines
- No framework configured yet. Recommended: Vitest or Jest.
- Place tests in `tests/` mirroring `src/` structure; name `*.test.(js|ts)`.
- Aim for fast, isolated unit tests for utilities and agents; prefer DI over globals.
- Once configured, run with `npm test`; target high coverage for critical paths.

## Commit & Pull Request Guidelines
- Use Conventional Commits: `feat:`, `fix:`, `chore:`, `docs:`, `refactor:`.
- PRs should include: clear description, linked issues, local run steps, and screenshots for user‑visible changes.
- Keep changes focused; avoid drive‑by refactors. Update docs when behavior changes.

## Security & Configuration Tips
- Store secrets in `.env`; never commit them. Ensure `.env` and `node_modules/` are git‑ignored.
- Rotate keys if exposed; prefer `npx dotenv -e .env -- <cmd>` during local runs.
- Review `.codexrc.yml` limits (e.g., `cost_cap_usd`) before long sessions.

## Agent‑Specific Notes
- Follow minimal‑diff edits; keep naming and structure consistent.
- Prefer adding new code under `src/` and tests under `tests/`.
