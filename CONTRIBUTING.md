# Contributing to BigLake

Thanks for your interest in contributing. BigLake is a multi-repo platform — this guide covers the conventions that apply across all repos.

## What contributions are welcome

- **Bug reports** — open an issue in the relevant repo with steps to reproduce
- **Bug fixes** — open a PR against the relevant repo
- **New data sources** — ETL pipelines for new Australian government datasets (see ETL conventions below)
- **Documentation improvements** — corrections, clarifications, missing docs
- **UI/UX improvements** — against the `ui` repo
- **API improvements** — against the `api` repo

If you're considering a larger change (new architecture, new service, major refactor), open an issue first to discuss.

## Before you start

1. Read [ARCHITECTURE.md](ARCHITECTURE.md) to understand how the repos fit together.
2. Read the [DEVELOPMENT.md](DEVELOPMENT.md) guide to run the relevant service locally.
3. Check the relevant repo's `README.md` for repo-specific conventions.

## Code conventions

**Python (etl, api, knowledge, intelligence)**
- Python 3.8+
- Use `yaml.safe_load()` for config files — never `yaml.load()`
- Follow OWASP Top 10 — no hardcoded secrets, sanitize inputs, no SQL injection
- `snake_case` for variables and dataset names

**Vue (ui)**
- Vue 3 Composition API with `<script setup>`
- No new npm dependencies without discussion
- All HTTP calls go through `src/api/client.js` — never call backends directly from components

**SQL (etl)**
- All business transformation logic belongs in DuckDB SQL in the silver layer
- AI (Gemini) is for structural parsing only — finding headers, data boundaries

## Documentation requirements

Every code change must include documentation updates. This is enforced:

- **API changes** — update `api/BFF_API_REQUIREMENTS.md` and the OpenAPI spec docstrings. Run `python scripts/update_openapi_baseline.py` and commit `openapi.json`.
- **ETL pipeline changes** — update or create `robots.md` and `humans.md` under `etl/documentation/data_source_ingestions/{source}/{pipeline}/`.
- **New GCP resources or secrets** — update the relevant repo's `setup.md`.
- **Local dev setup changes** — if any service's setup changes in a way that affects local development, update `DEVELOPMENT.md` in this repo.
- **Architectural decisions** — write an ADR in the relevant repo's `documentation/adr/` directory. See existing ADRs for the format.

## Pull request process

1. Fork the relevant repo and create a feature branch.
2. Make your changes with appropriate tests and documentation.
3. Ensure the existing tests pass.
4. Open a PR with a clear description of what changed and why.
5. A maintainer will review — expect feedback within a few days.

Commits are squashed on merge. Write a clear PR title — it becomes the commit message.

## Reporting security issues

See [SECURITY.md](SECURITY.md) for the disclosure policy and reporting channels. Do not report security issues in public issues, PRs, or discussions.

## Copyright

By contributing, you agree that your contributions will be subject to the same copyright terms as the project. See [COPYRIGHT](COPYRIGHT).
