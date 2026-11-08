# Tasks

## 1. Remove the compatibility code

- [x] 1.1 Remove `BeetsPlugin.__init_subclass__` and its class-copying layer (method/property `setattr` from `MetadataSourcePlugin`, abstract-class early return, deprecation warning) from `beets/plugins.py`; drop imports that become unused (`cached_property`, and `deprecate_for_maintainers` if nothing else needs it). <!-- agent: backend-engineer.build, depends_on: [], touches: [beets/plugins.py] -->
- [x] 1.2 Remove `BeetsPlugin._verify_config`, the `register_listener("pluginload", self._verify_config)` call in `BeetsPlugin.__init__`, and the `source_weight` fallback in `MetadataSourcePlugin.data_source_mismatch_penalty` so it reads only `data_source_mismatch_penalty`; drop `NotFoundError` and any other import left unused. <!-- agent: backend-engineer.build, depends_on: [], touches: [beets/plugins.py, beets/metadata_plugins.py] -->
- [x] 1.3 Make `find_metadata_source_plugins()` filter with `isinstance(p, MetadataSourcePlugin)` instead of `hasattr(p, "data_source")`, and remove the `# type: ignore[misc]` on the return. <!-- agent: backend-engineer.build, depends_on: [1.1], touches: [beets/metadata_plugins.py] -->
- [x] 1.4 Remove the `musicbrainz.enabled` compatibility block from `get_plugin_names()` (`mb_enabled` lookup, both `deprecate_for_user` calls, the append and the `disabled_plugins` insertion); drop `deprecate_for_user` if it is no longer used in the module. <!-- agent: backend-engineer.build, depends_on: [], touches: [beets/plugins.py] -->

## 2. Align the test suite with the removed behaviour

- [x] 2.1 Delete `TestDeprecationCopy.test_legacy_metadata_plugin_deprecation` and the `musicbrainz.enabled` deprecation tests (`test_deprecated_enabled`, `test_deprecated_disabled`), which assert the behaviour being removed; keep `test_default` and `test_other_plugin_enabled`. <!-- agent: backend-engineer.build, depends_on: [1.1, 1.4], touches: [test/test_plugins.py] -->
- [x] 2.2 Convert `TestDataSourceDistance`'s `OriginalPlugin` to subclass the test metadata-source helper instead of `BeetsPlugin` and remove the `pytest.deprecated_call()` wrapper and its TODO comment. <!-- agent: backend-engineer.build, depends_on: [1.1], touches: [test/autotag/test_distance.py] -->
- [x] 2.3 Add regression tests: a `BeetsPlugin` subclass that sets `data_source` is absent from the resolved sources and gains no metadata-source attributes; a configured `source_weight` yields the default penalty with no warning. <!-- agent: backend-engineer.build, depends_on: [1.2, 1.3], touches: [test/test_metadata_plugins.py, test/test_plugins.py] -->

## 3. Documentation

- [x] 3.1 Rewrite the "Migration guidance" section of `docs/dev/plugins/autotagger.rst`: plugins must subclass `MetadataSourcePlugin` (API-backed sources `SearchApiMetadataSourcePlugin`), and the promise of legacy support in v3.0.0 is replaced by the statement that it is removed. <!-- agent: backend-engineer.fast, depends_on: [], touches: [docs/dev/plugins/autotagger.rst] -->
- [x] 3.2 Remove the deprecated `source_weight` conf block from `docs/plugins/shared_metadata_source_config.rst` and any remaining reference to it in the plugin configuration reference. <!-- agent: backend-engineer.fast, depends_on: [], touches: [docs/plugins/shared_metadata_source_config.rst] -->
- [x] 3.3 Add changelog entries under `Unreleased`: an entry in "For plugin developers" for the removed metadata-source compatibility layer and `source_weight`, and an entry in "Other changes" for the removed `musicbrainz.enabled` option. <!-- agent: backend-engineer.fast, depends_on: [1.1, 1.2, 1.4], touches: [docs/changelog.rst] -->

## 4. Verification

- [x] 4.1 Run `poe lint`, `poe check-types`, `poe format`, and the affected test modules (`test/test_plugins.py`, `test/test_metadata_plugins.py`, `test/autotag/`); fix fallout and confirm no `v3.0.0` markers for these shims remain. <!-- agent: backend-engineer.fast, depends_on: [2.1, 2.2, 2.3, 3.1, 3.2, 3.3], touches: [] -->

Verified in the lead session: `ruff check` clean on the repo's own code (only pre-existing
`.agents/skills/` findings), `ruff format --check` clean for changed files, `mypy .`
reports no errors, `sphinx-lint --enable all` clean on the changed RST, full suite
2855 passed. Note: `poe` is not installed on this machine, so the underlying
`ruff`/`mypy`/`pytest` commands were run directly.
