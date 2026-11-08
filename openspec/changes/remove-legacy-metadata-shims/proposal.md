# Remove the legacy metadata-source shims slated for v3.0.0

## Why

beets 2.x carries four compatibility layers that the code itself marks for removal
at v3.0.0, and that the documentation already describes as temporary:

- `BeetsPlugin.__init_subclass__` (`beets/plugins.py:119-171`) supports plugins that
  predate the `MetadataSourcePlugin` contract by copying every non-abstract
  method and property of `MetadataSourcePlugin` onto the subclass at class-creation
  time.
- `BeetsPlugin._verify_config` (`beets/plugins.py:195-222`), registered on the
  `pluginload` event for every plugin instance, exists only to warn about the
  deprecated `source_weight` option — and
  `MetadataSourcePlugin.data_source_mismatch_penalty`
  (`beets/metadata_plugins.py:193-198`) still reads `source_weight` in preference to
  the supported key.
- `find_metadata_source_plugins()` (`beets/metadata_plugins.py:47-52`) detects sources
  with `hasattr(p, "data_source")` instead of an `isinstance` check, which is why the
  return line needs a `# type: ignore[misc]`.
- `find_plugins()` (`beets/plugins.py:502-514`) still honours the deprecated
  `musicbrainz.enabled` option, adding or removing `musicbrainz` from the resolved
  plugin list.

These shims are not free: the first performs runtime class surgery, so a plugin's
public surface is decided by `setattr` rather than by its base classes; the others
keep a second, deprecated configuration path alive next to the supported one, with no
test coverage for `source_weight` at all. v3.0.0 is the release that has to honour
those markers, and leaving them in place keeps the metadata-source contract
ambiguous and blocks the `isinstance` cleanup in
`find_metadata_source_plugins()`.

## What Changes

- **Delete the class-copying shim.** `BeetsPlugin` no longer gains
  `__init_subclass__`; a plugin is a metadata source only by subclassing
  `MetadataSourcePlugin`. No deprecation warning is emitted for the legacy shape.
- **Delete the configuration shims.** `_verify_config` and its `pluginload`
  registration go, and `data_source_mismatch_penalty` reads only
  `data_source_mismatch_penalty`. A configured `source_weight` is ignored silently.
- **Tighten source discovery.** `find_metadata_source_plugins()` filters with
  `isinstance(p, MetadataSourcePlugin)`, and the `type: ignore` is removed.
- **Drop `musicbrainz.enabled`.** The resolved plugin list comes solely from
  `plugins`/`disabled_plugins`; the deprecated option neither enables nor disables
  MusicBrainz.
- **Update the tests that assert the removed behaviour**, replace the one test that
  deliberately relies on the shim, and add regression coverage for the new rules.
- **Update the developer documentation and changelog**: the migration guidance in
  `docs/dev/plugins/autotagger.rst` stops promising legacy support, the deprecated
  `source_weight` entry disappears from the plugin configuration reference, and both
  breaks are recorded under Unreleased.

## Impact

- Affected specs: `metadata-source-plugins` and `plugin-loading` (both added by this
  change; the repository has no existing specs).
- Affected code: `beets/plugins.py`, `beets/metadata_plugins.py`,
  `test/test_plugins.py`, `test/autotag/test_distance.py`,
  `test/test_metadata_plugins.py`, `docs/dev/plugins/autotagger.rst`,
  `docs/plugins/shared_metadata_source_config.rst`, `docs/changelog.rst`.
- **Breaking for third-party plugins** that inherit `BeetsPlugin` and only set
  `data_source`; they must subclass `MetadataSourcePlugin` (or
  `SearchApiMetadataSourcePlugin`) instead. Every bundled source already does, so no
  in-tree plugin changes behaviour.
- **Breaking for users** who set `source_weight` or `musicbrainz.enabled`; both
  options become inert and must be migrated to `data_source_mismatch_penalty` and the
  `plugins` list.
- Out of scope: bumping the package version to 3.0.0 (release tooling owns
  `pyproject.toml` and `extra/release.py`), and any other `DeprecationWarning` surface
  not listed above.
