# CLAUDE.md

Guidance for Claude Code (claude.ai/code) when working in this repository.

## What This Is

**SearXNG** is a self-hostable, privacy-respecting metasearch engine: a Flask
web app that fans a query out to dozens of upstream search engines/APIs
("engines"), normalizes and merges the results, and renders them without
tracking or profiling users. This repo (`RJHuey73/searxng`) is a fork of
`searxng/searxng`.

## Layout

| Path | Purpose |
|------|---------|
| `searx/` | The application package (Python). |
| `searx/webapp.py` | Flask app / HTTP entry point (`searxng-run` console script calls `searx.webapp:run`). |
| `searx/search/` | Search orchestration: `SearchWithPlugins`, per-engine `processors/`, request/response models. |
| `searx/engines/` | One module per upstream search engine/API (~200+), e.g. `bing.py`, `arxiv.py`, `google.py`. |
| `searx/enginelib/` | Shared `Engine`/`EngineAbout` base types used by `searx/engines/__init__.py:load_engines`. |
| `searx/plugins/` | Built-in plugins (pre-search / post-search / on-result hooks); see docstring in `searx/plugins/__init__.py`. |
| `searx/result_types/` | Typed result models (`Answer`, etc.) returned by engines/plugins. |
| `searx/botdetection/` | Bot/abuse detection for the public-facing instance. |
| `searx/settings.yml`, `searx/settings_defaults.py`, `searx/settings_loader.py`, `searx/_settings.py` | Default configuration and the settings-loading pipeline (env var `SEARXNG_SETTINGS_PATH` overrides). |
| `searx/data/`, `searx/favicons/` | Bundled static data (engine traits, currencies, locales, useragents) and favicon resolvers. |
| `searx/static/`, `searx/templates/`, `client/` | Frontend assets/themes and the `simple` theme's TS/CSS sources. |
| `searx/translations/`, `.weblate` | i18n; translations are managed via Weblate, not direct PRs. |
| `searxng_extra/` | Auxiliary scripts: `docs_prebuild`, `update` (fetchers that refresh `searx/data/*`). |
| `tests/unit/` | nose2 unit tests, mirrors `searx/` structure (`engines/`, `network/`, `processors/`, `settings/`). |
| `tests/robot/` | Robot Framework browser-driven integration tests. |
| `utils/` | Shell library backing `./manage` (`lib.sh`, `lib_sxng_test.sh`, `lib_sxng_themes.sh`, etc.) and dev/ops scripts. |
| `docs/` | Sphinx documentation source (admin, dev, user guides) built by `manage docs.html`. |
| `container/` | Docker/container entrypoint and build assets. |

## Commands

All developer commands go through the `./manage` script (backed by `utils/lib*.sh`) or the `Makefile` wrapper around it. `make install` sets up a Python virtualenv under `local/py3` (per `pyrightconfig.json`).

```bash
make install              # create venv + install SearXNG for development (./manage pyenv.install)
make run                  # run the dev instance (installs first)
make uninstall             # remove the dev virtualenv

make test                 # full CI-equivalent suite (yamllint, black, pyright, pylint, unit, robot, rst, shell, shfmt)
make ci.test               # test + pybabel
./manage test.unit         # unit tests only (nose2 over tests/unit)
./manage test.coverage     # unit tests with coverage report (searx package)
./manage test.robot        # Robot Framework browser tests
./manage test.pylint       # pylint over searx/engines, searx, searxng_extra, tests
./manage test.pyright      # basedpyright static type check (full)
./manage test.pyright_modified   # basedpyright, only locally-modified .py files
./manage test.black        # black --check --diff
./manage test.yamllint     # yamllint over YAML files
./manage test.rst          # doctest/lint of .rst files incl. README.rst

make format                # format.python (black) + format.shell (shfmt)
./manage docs.html         # build Sphinx docs
```

Run a single unit test file directly once the venv is active (`./manage pyenv.activate` or `source local/py3/bin/activate`):

```bash
python -m nose2 -s tests/unit tests.unit.test_search
```

## Conventions

- **Settings pipeline**: `searx/settings.yml` (defaults) is loaded via
  `searx/settings_loader.py`/`searx/_settings.py`; `SEARXNG_SETTINGS_PATH` (env
  var) points at an instance's override file. `searx/settings_defaults.py`
  supplies programmatic fallbacks. Don't hardcode config — read through
  `searx.settings`.
- **Engines are declarative modules.** Each file in `searx/engines/` exposes
  module-level attributes (`engine_type`, `paging`, `categories`, `request()`,
  `response()`, etc.) consumed by `load_engines()` in
  `searx/engines/__init__.py`; engine behavior/capabilities are also declared
  per-engine in `searx/settings.yml`'s `engines:` list. New engines should
  follow the shape of an existing similar engine, not be built from scratch.
- **Plugins hook three points**: `pre_search`, `post_search`, `on_result` (see
  the module docstring in `searx/plugins/__init__.py` for worked examples).
  Plugins return/modify typed `searx.result_types` objects (e.g. `Answer`),
  not raw dicts.
- **Search flow**: `searx.search.SearchWithPlugins` drives `searx/search/processors/`
  (one processor class per engine type/protocol) which issue requests via
  `searx/network/` and feed results into `searx.results.ResultContainer`.
- **Coding style**: PEP 8 + PEP 20, Clean Code (see CONTRIBUTING.rst) —
  simplicity, one thing per function, descriptive names, no dead/commented-out
  code, no obvious comments. Formatting is enforced by `black` (`test.black`)
  and linted by `pylint`/`basedpyright`/`yamllint`/`shellcheck` (`make test`).
- **Commit messages**: descriptive, present tense, imperative mood ("Add
  feature", not "Added feature"/"fix bug"), first line ≤72 chars — see
  CONTRIBUTING.rst and the linked Commits guide.
- **Translations** are managed externally via Weblate
  (`translate.codeberg.org/projects/searxng`) — don't hand-edit
  `searx/translations/` in a PR.
- **AI-assisted contributions are policy-gated** — see `AI_POLICY.rst`: any AI
  use must be disclosed, the human contributor must fully understand and own
  the change, AI must not be the main author, and issue/PR prose must be
  human-written (translation of self-written text is the only carve-out).

## Test seam

- `tests/unit/` mirrors `searx/`'s structure (`engines/`, `network/`,
  `processors/`, `settings/`) plus top-level `test_*.py` files (search, query,
  webapp, plugins, preferences, results, external bangs, locales, etc.). Run
  with `./manage test.unit` (nose2) or `make test` for the full gate.
- `tests/robot/` holds Robot Framework end-to-end/browser tests
  (`./manage test.robot`).
- `.coveragerc` configures `./manage test.coverage`.
- CI parity: `make ci.test` = `make test` + `test.pybabel` (checks translation
  extraction is up to date).

## Gotchas

- `./manage` sources multiple `utils/lib_sxng_*.sh` files — when adding a new
  dev command, add it to the right `lib_sxng_*.sh` and wire it into the
  `Makefile`'s `MANAGE +=` list so `make <target>` forwards to it.
- `test.pyright` (full basedpyright run) intentionally ignores its own exit
  code in `utils/lib_sxng_test.sh`; use `test.pyright_modified` for a
  fail-fast check on files you've actually touched.
- Engine additions/changes typically need a matching entry in
  `searx/settings.yml`'s `engines:` list, not just the new module in
  `searx/engines/`.
- Data files under `searx/data/` (engine traits, currencies, useragents,
  locales) are generated/refreshed by `searxng_extra/update/` scripts and
  `./manage data.*` targets — don't hand-edit generated JSON there.
