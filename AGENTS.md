# AGENTS.md — HiveMind-community-docs

Community documentation for HiveMind, published as a MkDocs static site to GitHub Pages. There is no Python package, library, or runtime code here — content is Markdown under `docs/`.

## Setup

```bash
pip install -r requirements.txt
```

The site is built with **Material for MkDocs** (`mkdocs-material`) plus
`pymdown-extensions`. Pinned in `requirements.txt`. Beginner/advanced layering relies on
the configured markdown extensions — `admonition`, `pymdownx.details` (collapsible
`??? note`), and `pymdownx.tabbed` (content tabs) — so keep those enabled.

## Test

No automated tests. To validate the build locally:

```bash
mkdocs build --strict   # fails on broken nav / warnings
mkdocs serve            # live preview at http://127.0.0.1:8000
```

## Lint/Typecheck

None configured. Keep `mkdocs.yml` `nav` entries in sync with files in `docs/`.

## Layout

- `mkdocs.yml` — site config and `nav` tree. Sections: Home, About, Quick Start, FAQ,
  Core Concepts, Satellites, Hub & Server, Integrations, Developer Guide, Reference. Each
  section has an `index.md` landing page (enabled via the `navigation.indexes` feature).
- `docs/` — page content grouped by section folder (`concepts/`, `satellites/`, `server/`,
  `integrations/`, `developers/`, `reference/`).
- `docs/assets/` — `logo.png` / `favicon.png` used by the theme.
- `docs/javascripts/analytics.js` — injected via `extra_javascript`.
- `requirements.txt` — pinned build deps (mkdocs, mkdocs-material, pymdown-extensions).
- `.github/workflows/build.yml` — installs `requirements.txt`, runs `mkdocs build --strict`,
  then publishes via `mkdocs gh-deploy` on push.
- `renovate.json` — Renovate dependency dashboard config.

Not a plugin/skill — no entry points, no `pyproject.toml`/`setup.py`.

## Editorial conventions

- **Dual audience.** Every page opens with a plain-language orientation a non-developer
  understands, then layers depth. Put advanced material in collapsible `??? note "Advanced:
  …"` blocks so beginners can skip it.
- **Cross-reference to source.** Each concept/reference/developer page ends with a
  `## Source` section linking the backing source files via
  `https://github.com/<owner>/<repo>/blob/HEAD/<path>` (use `HEAD`, never a branch name).
- Define jargon on first use or link it to `reference/glossary.md`.

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

- The publish workflow (`build.yml`) is a hand-rolled `mkdocs gh-deploy`, not a gh-automations reusable workflow.
- New pages must be added to the `nav` in `mkdocs.yml` or they will not appear in the site. `mkdocs build --strict` fails if a nav entry points at a missing file (and vice-versa with strict).
- Source-footer links use `/blob/HEAD/` so they don't break when a repo's default branch differs (`dev` vs `master`). Note case-sensitive repo slugs (`HiveMind-core`, but `hivemind-plugin-manager`) and that the a2a plugin lives under the `TigreGotico` org, not `JarbasHiveMind`.
- The JSON DB plugin's module dir is `hivemind_json_database` (not `…_db_plugin`).
