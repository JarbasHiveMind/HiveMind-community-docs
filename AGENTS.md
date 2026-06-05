# AGENTS.md — HiveMind-community-docs

Community documentation for HiveMind, published as a MkDocs static site to GitHub Pages. There is no Python package, library, or runtime code here — content is Markdown under `docs/`.

## Setup

```bash
pip install mkdocs
```

The site uses the built-in `readthedocs` theme (no extra theme package required).

## Test

No automated tests. To validate the build locally:

```bash
mkdocs build --strict   # fails on broken nav / warnings
mkdocs serve            # live preview at http://127.0.0.1:8000
```

## Lint/Typecheck

None configured. Keep `mkdocs.yml` `nav` entries in sync with files in `docs/`.

## Layout

- `mkdocs.yml` — site config and `nav` tree (Home, About, Servers, Satellites, Integrations, Protocol, Security, Developers).
- `docs/` — all page content as numbered Markdown (`00_index.md` … `19_crypto.md`), plus `gpt_*.md` explainer pages and `img_*.png` / diagram assets referenced inline.
- `docs/javascripts/analytics.js` — injected via `extra_javascript`.
- `.github/workflows/build.yml` — builds and publishes via `mkdocs gh-deploy` on push.
- `renovate.json` — Renovate dependency dashboard config.

Not a plugin/skill — no entry points, no `pyproject.toml`/`setup.py`.

## Conventions

- Branches: work on `dev`, stable is `master`. NEVER use `main`.
- Never edit any `version.py`; gh-automations bumps semver from conventional-commit prefixes (`feat:`/`fix:`/`feat!:`). (No version file here, but the rule stands org-wide.)
- New repos are private by default; never make a source repo public without asking.
- Commit identity: JarbasAi <jarbasai@mailfence.com>.
- Reference `OpenVoiceOS/gh-automations` reusable workflows at `@dev`.
- No Neon / `neon-*` references.
- No meta-commentary in docs/commits/PRs (no history, no dates, describe current state only).
- CI is provided by `OpenVoiceOS/gh-automations`.

## Gotchas

- The publish workflow (`build.yml`) is a hand-rolled `mkdocs gh-deploy`, not a gh-automations reusable workflow, and pins old actions (`actions/checkout@v2`, `actions/setup-python@v2`).
- New pages must be added to the `nav` in `mkdocs.yml` or they will not appear in the site.
- Several pages contain inline `TODO` markers and dead/empty links (e.g. `[http bridge]()`); these are content gaps, not build failures.
- `gpt_*.md` pages are AI-authored explainers — verify accuracy against the protocol pages before relying on them.
