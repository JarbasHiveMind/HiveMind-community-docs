# TODO — HiveMind-community-docs

## Open issues

- [ ] #4 Dependency Dashboard (Renovate bot)

## Gaps

- [ ] No automated tests (a `mkdocs build --strict` check in CI would catch broken nav/links).
- [ ] Publish workflow `build.yml` is hand-rolled `mkdocs gh-deploy`, not a `OpenVoiceOS/gh-automations` reusable workflow.
- [ ] `build.yml` pins outdated actions (`actions/checkout@v2`, `actions/setup-python@v2`).
- [ ] No `pyproject.toml`/`setup.py` (expected — docs-only repo).
- [ ] Standard gh-automations CI (build-tests, coverage, license-check, release_workflow, publish_stable) absent; not all apply to a docs site, but none are present.
- [ ] Dead/empty links in content (e.g. `[http bridge]()` in `docs/TODO.md`).

## Code TODOs

- `docs/07_micsat.md:47` — Dialog Transformers plugins (TODO - support in the future)
- `docs/TODO.md:41` — a storage node can be a http api (see [http bridge]() TODO)
- `docs/14_localhive.md:22` — skills should be able to request to listen for specific messages; cross-skill communication currently impossible
