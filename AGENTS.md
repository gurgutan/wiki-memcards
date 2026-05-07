# Repository Guidelines

## Project Structure & Module Organization

This repository currently contains the product and planning foundation for Wiki Cards, a web app that turns Wikipedia pages into study cards. Root documents hold product-level guidance: `README.md`, `PRODUCT_SENSE.md`, `DESIGN.md`, and `OBSERVABILITY.md`. Draft templates and examples live under `docs/drafts/`, including `FEATURE_TEMPLATE.md`, `product-sense-interview.md`, and `.gitlab-ci.example.yml`.

No application source tree is present yet. When implementation starts, keep frontend code in `apps/web/`, reusable packages in `packages/`, tests in `tests/` or feature-local `__tests__/`, and static assets in the relevant app's `public/` or `src/assets/` directory.

## Build, Test, and Development Commands

There are no committed build scripts yet. Use these checks for the current documentation-only state:

- `git status --short` checks the working tree before and after edits.
- `find . -maxdepth 3 -type f | sort` verifies the repository layout.
- `npx markdownlint-cli2 "**/*.md"` may be used for Markdown linting once Node tooling is available.

When an app is added, document its real commands here, for example `npm run dev`, `npm run build`, `npm test`, or `uv run pytest`.

## Coding Style & Naming Conventions

Keep Markdown concise, with sentence-case prose and descriptive headings. Use kebab-case for documentation filenames such as `feature-template.md`; preserve existing uppercase root knowledge-base files where already established.

For future TypeScript frontend code, prefer React with TypeScript, two-space indentation, PascalCase components, camelCase functions, and explicit domain names such as `deckStorage` or `wikiClient`. For Python services, use `uv`, `ruff`, `pyright`, `pytest`, `pydantic`, `FastAPI`, and `SQLModel` where applicable.

## Testing Guidelines

Current changes should be reviewed by reading rendered Markdown or linting it. Future code changes must include automated tests before completion. Name Python tests `tests/test_<feature>.py`; name frontend tests close to the feature or under `tests/`. Add browser validation with Playwright or Chrome DevTools MCP for visible UI flows.



## Security & Configuration Tips

Do not commit secrets, tokens, local databases, or generated build artifacts. Prefer `.env` for local settings and document required variables next to the app or service that consumes them.
