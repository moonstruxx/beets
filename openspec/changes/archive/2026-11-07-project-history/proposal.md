# Project History: beets

## Why

This repository is an existing, mature open-source project (beets, version 2.14.1,
15,643 commits). Before any agent-driven change work begins, the history and the
decisions already embodied in the codebase need a durable record in OpenSpec, so
future changes can be argued against something more than the current file tree.

Recorded at HEAD `e978874d1` (2026-11).

## What this project is

beets is the media library management system for obsessive music geeks. It is a
Python CLI application *and* a library:

- **`beet import`** scans files, identifies releases via MusicBrainz, Discogs or
  Beatport, and writes corrected metadata and a canonical on-disk layout.
- **A SQLite-backed library database** records items (tracks) and albums and makes
  them queryable with a plugin-extensible query language.
- **~74 bundled plugins** (`beetsplug/`) extend it: artwork fetching, lyrics,
  transcoding, ReplayGain, duplicates/missing-track checks, a Flask web UI
  (`aura`), an MPD-speaking player (`bpd`), Plex/Kodi/Sonos/Emby update hooks,
  and more.
- **A plugin API** is the project's stated extension point; the README advertises
  that writing a plugin is "shockingly simple if you know a little Python".

Entry point: `beet = beets.ui:main` (see `pyproject.toml [project.scripts]`).

## Key decisions already made (inferred from code and docs)

### Packaging and platform

- **uv** is the packaging and environment tool: `[project]` metadata in
  `pyproject.toml`, `uv.lock` committed, `hatchling` as the build backend.
  (The project used Poetry until the 2.13/2.14 era; `poetry.lock` and
  `[tool.poetry]` are gone.)
- Supports Python 3.11 through 3.15 (`requires-python = ">=3.11,<3.16"`).
- Heavy features are *optional extras* rather than hard dependencies: `web`,
  `fetchart`, `lyrics`, `chroma`, `replaygain`, `bpd`, `discogs`, `lastgenre`,
  `tidal`, `beatport`, `autobpm`, etc. Core dependencies stay deliberately small
  (`confuse`, `mediafile`, `numpy`, `jellyfish`, `lap`, `requests`,
  `requests-ratelimiter`, `unidecode`, `pyyaml`, `platformdirs`, `packaging`).
- Platform-specific behaviour uses environment markers (`colorama` on Windows,
  `gstreamer-bundle` for Windows bpd/replaygain, `PyGObject` off Windows,
  `dbus-python` for `metasync` outside Windows), plus `[tool.uv]
  constraint-dependencies` pinning broken gstreamer plugin releases.

### Architecture

- **Layered package layout** under `beets/`:
  - `dbcore/` — the database abstraction: typed `Model` classes, a
    schema/field system (`types.py`), the query language (`query.py`,
    `queryparse.py`), sorting (`sort.py`) and path handling (`pathutils.py`).
  - `library/` — concrete `Item`/`Album` models, field definitions (`fields.py`),
    `Library` facade, plugin-facing queries, and **schema migrations**
    (`migrations.py`). Migrations are a first-class concern and run automatically
    against long-lived user databases.
  - `importer/` — a staged pipeline (`stages.py`) driven by `session.py` and
    `tasks.py`, with deferred actions in `actions.py` and shared state in
    `state.py`. Import is pluggable at each stage via events.
  - `autotag/` — matching against MusicBrainz (`match.py`) and distance
    heuristics (`distance.py`) with `hooks.py` for plugin overrides.
  - `events.py` — a typed registry of every plugin event name and its payload
    (`EventType` union, `ALL_EVENTS`, one `TypedDict` per payload). This is the
    machine-readable contract for plugin hooks.
  - `ui/` — the CLI: command dispatch (`ui/commands/`) plus option/query parsing.
  - `util/` — cross-cutting helpers, including `functemplate.py` (the path
    format template language), `artresizer.py`, `pipeline.py`, `bluelet.py`,
    `deprecation.py`, `config.py`, `hidden.py`, `layout.py`.
  - `plugins.py` + `metadata_plugins.py` — plugin loading, event dispatch and
    the metadata-source plugin contract.
- **SQLite is the store**, accessed through the `dbcore` model layer rather than
  ad-hoc SQL, so plugin authors get typed fields and the same query semantics.
- **Events, not subclassing**, are the primary plugin integration mechanism
  (`import_begin`, `import_task_*`, `album_imported`, `item_copied`,
  `before_choose_candidate`, `write`, `database_change`, `mb_*_extract`, etc.),
  documented in `docs/dev/plugins/events.rst` and typed in `beets/events.py`.
- **Configuration is `confuse`-based** with defaults in
  `beets/config_default.yaml` and user config in a platformdirs-resolved file
  (`BEETSDIR` overrides).
- **Path formatting** is a user-visible mini-language
  (`$albumartist/$album%aunique{}/$track $title`) implemented in
  `functemplate.py` with a stability contract, because users' on-disk libraries
  depend on it.

### Process and quality

- Tests are pytest-based under `test/`, mirroring the source tree
  (`test/dbcore`, `test/library`, `test/autotag`, `test/plugins`, `test/ui`,
  `test/util`, `test/extra`), with fixtures in `test/conftest.py`, a fixture
  library in `test/rsrc/`, HTTP mocking via `responses`/`requests-mock`, and
  parallel execution via `pytest-xdist` (`-n auto`, `--dist=loadscope`) plus
  `pytest-randomly`.
- Lint/format: `ruff` (line length 80, target py310, rule families including
  isort, pyupgrade, flake8-type-checking, pytest-style). Typing: `mypy` with
  `beets/py.typed` — the package ships types, and several modules are under
  `strict`.
- Docs: Sphinx (`docs/`), split into guides, plugin pages, reference, CLI docs
  and developer docs; `docstrfmt` and `sphinx-lint` gate formatting.
- Changelog discipline: every user-visible change gets an entry in
  `docs/changelog.rst` with `:bug:`/`:user:` roles and PR references; a workflow
  comments on PRs that change Python without touching the changelog.
- Commit subjects are prefixed by area with PR numbers
  (`convert: skip missing album art instead of crashing (#6749)`), merged via
  branch + merge commits.
- Task automation via `poe` tasks in `pyproject.toml` (`poe install`, `poe test`,
  `poe lint`, `poe format`, `poe check-types`, `poe docs`, `poe lint-docs`).
- `AI_POLICY.md` sets expectations for AI-assisted contributions.

## Known tech debt and constraints visible in the codebase

- **Plugin count vs. maintenance surface**: ~74 plugins in one flat package, many
  requiring external binaries or services (`ffmpeg`, `fpcalc`, `mp3val`,
  ImageMagick, gstreamer, beets.io-hosted APIs). Tests mock heavily; some plugin
  paths are only exercised by the weekly `integration_test` workflow.
- **MusicBrainz rate limiting** shapes the autotag design; import correctness and
  performance are both constrained by it (`requests-ratelimiter`).
- **Long-lived user databases**: anything touching `library/models.py` or
  `library/fields.py` risks needing a `migrations.py` entry. Breaking migrations
  is the highest-blast-radius class of change here.
- **Path format compatibility**: changes in `functemplate.py` / `pathformats.py`
  can silently relocate users' files; treated as a stability boundary.
- **Optional-dependency sprawl** in `pyproject.toml` (single-key extras, many
  commented-out entries, OS-conditional gstreamer pins) makes dependency
  auditing noisy.
- **Deprecations** are managed through `beets/util/deprecation.py`; removing a
  deprecation has an explicit version-gated contract.
- RST documentation and an RST changelog must satisfy `sphinx-lint`/`docstrfmt`,
  which is stricter than Markdown.
- Windows/macOS/Linux behaviour differences (path separators, filesystem
  encoding, external tool discovery) are real and tested unevenly.
- Test suite is heavily mocked and slow; `setup.cfg` disables coverage by default
  (`-p no:cov`) and pushes caches to `/tmp`.

## Current state

- Version 2.14.1, active `Unreleased` section in `docs/changelog.rst`.
- Packaging is uv + hatchling; CI installs with uv and `poe install`.
- CI matrix: Ubuntu and Windows × Python 3.11–3.15, plus a base-install job that
  installs without dev dependencies and runs `beet version`.
- Governance is maintainer-led (README credits Adrian Sampson; `CONTRIBUTING.rst`
  is the operative guide).

## What changes

Nothing in the codebase. This change only records project history as an archived
OpenSpec change so future proposals have a baseline to cite.

## Out of scope

- Any code, dependency, or documentation modification.
- Reconstructing a per-release decision log; only decisions inferable from the
  current tree and its docs are recorded.
