# Tasks

## 1. Core: stop re-reading config per path legalization

- [x] 1.1 Memoize `beets.util.get_empty_component_replace()` with `@cache` and add a `clear_path_caches()` helper that clears it; verify `pytest test/test_util.py` passes and a direct call count check shows one config resolution for two `sanitize_path()` calls.
  <!-- agent: fixer | depends_on: - | touches: beets/util/__init__.py -->
- [x] 1.2 Clear the path caches before each test through an autouse fixture in `test/conftest.py`; verify the tests that set `config["empty_component_replace"]` (`test/test_util.py:132`, `test/test_library.py:521,529,539`) pass under `pytest -p randomly --randomly-seed=1`.
  <!-- agent: fixer | depends_on: 1.1 | touches: test/conftest.py, test/test_util.py -->
- [x] 1.3 Confirm caching changes no output: `pytest test/test_util.py test/test_library.py -k "path or sanitize or legalize or destination"` passes.
  <!-- agent: fixer | depends_on: 1.2 | touches: - -->

## 2. Core: fetch an item's album only when a template needs it

- [x] 2.1 In `FormattedItemMapping._get()`, consult `self.model_keys` before `self.album`, and decide `for_path` album precedence with the static `Album.item_keys` set; keep today's behaviour when `included_keys` is an explicit list, and guard a set `album_id` whose album is missing. Verify `pytest test/test_library.py -k format` passes.
  <!-- agent: fixer | depends_on: - | touches: beets/library/models.py -->
- [x] 2.2 Add a regression test that evaluating an item-only path format (`$artist/$title`) performs no `Library.get_album()` call (wrap or count the method); verify the test fails against the pre-change code.
  <!-- agent: fixer | depends_on: 2.1 | touches: test/test_library.py -->
- [x] 2.3 Add a regression test that an album field still wins over the item field for `for_path=True` templates and that an album flex field remains reachable; verify `pytest test/test_library.py` passes.
  <!-- agent: fixer | depends_on: 2.1 | touches: test/test_library.py -->

## 3. Plugin: one destination and one album lookup per item

- [x] 3.1 In `beets-alternatives`, read `lib.albums()` and `lib.items()` once in `External._items_actions()` and group items by `album_id` in memory instead of calling `album.items()` per album; verify the plugin's tests pass.
  <!-- agent: fixer | depends_on: - | touches: beets-alternatives/beetsplug/alternatives.py, beets-alternatives/test/ -->
- [x] 3.2 Yield the computed destination from `_items_actions()` and reuse it in `External.update()`, removing the second `destination()` call per item; verify the plugin's tests pass.
  <!-- agent: fixer | depends_on: 3.1 | touches: beets-alternatives/beetsplug/alternatives.py, beets-alternatives/test/ -->
- [x] 3.3 Pass the album from the grouped map into `item_change_actions()` and `_sync_art()`, seed `item._cached_album` from the map before computing a destination, and use the grouped items in `update_art()`; verify no `item.get_album()` work remains in the update loop and the plugin's tests pass.
  <!-- agent: fixer | depends_on: 3.2 | touches: beets-alternatives/beetsplug/alternatives.py, beets-alternatives/test/ -->
- [x] 3.4 Run the plugin's full suite (`uv run pytest`) and type check (`uv run pyright`) inside `beets-alternatives/`; report any environment failure instead of skipping the checks.
  <!-- agent: fixer | depends_on: 3.3 | touches: - -->

## 4. Integration in this repo

- [x] 4.1 Commit the plugin change in the `beets-alternatives` submodule and bump the pin in this repo; verify `git submodule status` shows the new commit.
  <!-- agent: lead | depends_on: 3.4 | touches: beets-alternatives -->
- [x] 4.2 Add a `docs/changelog.rst` entry under Other changes describing the faster path computation; verify `poe lint-docs` passes.
  <!-- agent: fixer | depends_on: 1.3, 2.3 | touches: docs/changelog.rst -->
- [x] 4.3 Re-run the throwaway benchmark (`/tmp/opencode/alt-bench.py`, `alt-bench2.py`) against the 3,600-item synthetic library: no-op `update()` must drop from 10.2 s to under 5 s and `destination()` must be called once per item; record the numbers in the PR description.
  <!-- agent: lead | depends_on: 4.1, 4.2 | touches: - -->
- [x] 4.4 Run the full gate: `poe test`, `poe lint`, `poe check-format`, `poe check-types`.
  <!-- agent: lead | depends_on: 4.3 | touches: - -->
