# Design

## Context

See `proposal.md` for motivation and the measurements.

Three hot spots produce the profile: two `Item.destination()` calls per item from
the alternatives plugin, a per-call config read inside `sanitize_path()`, and one
album fetch per item from `FormattedItemMapping`. The relevant current mechanics:

- `beets/util/`: `Item.destination()` (`beets/library/models.py:1249`)
  evaluates a template and then calls `util.legalize_path()`, which runs up to
  three `_legalize_stage()` rounds; each round calls `sanitize_path()` (which
  calls `get_empty_component_replace()` — a config read plus `re.compile`) and
  `truncate_path()` (which uses the already-`@cache`d
  `get_max_filename_length()`).
- `FormattedItemMapping._get()` (`beets/library/models.py:251`) evaluates
  `self.album` — `item._cached_album` → `Library.get_album()` → two SQL
  statements — before it looks at `self.model_keys`.
- `Library.get_album()` has no cache; each `Item` holds one album object, so an
  album with n tracks is fetched n times per pass.
- Two sinks in `@cache`-style caching already exist in this code, so the
   precedent for process-lifetime caches is established (`get_max_filename_length`).

## Goals / Non-Goals

**Goals:**

- Remove the per-call config read and regex compile from `sanitize_path()`.
- Stop fetching an album when a template does not reference an album field.
- Make `alt update` compute each destination once and resolve each album once.
- Keep `Item.destination()`, template evaluation, and event behavior identical.

**Non-Goals:**

- No library-wide album cache. Albums are mutable (`Album.store()`,
  `item_change_actions` re-reads art), and a stale cache would be a correctness
  change disguised as an optimization. Album laziness plus plugin-side reuse is
  enough for this change.
- No batching of `item.store()` calls in the plugin. Measured at 0.05 s per 500
  stores on the dev machine, this is not the bottleneck, and holding one
  transaction across file copies and transcodes would hold the global database
  lock for the entire run.
- No SQLite `journal_mode`/`synchronous` changes.

## Decisions

### Cache the empty-component replacement resolver

Wrap the resolver in a module-level memo (`@cache`), plus an explicit
`clear_path_caches()` (or equivalent) that the test suite calls between tests.

- Alternative considered: key the cache on the configured replacement string.
  Rejected: the config lookup, not the `re.compile`, dominates the 16.8 µs call;
  keying on the value still pays for the lookup.
- Alternative considered: thread the resolver through `Library.replacements`
  like the compiled `replace` rules. Rejected: `sanitize_path()` is public API
  used by plugins without a `Library`, and widening `Replacements` changes a
  public type for an internal optimization.
- Staleness is the risk: `beets.config["empty_component_replace"]` is set at
  runtime by tests (`test/test_util.py:132`, `test/test_library.py:521-539`) and
  could in principle be changed by a plugin. Mitigated by clearing the cache in
  the test fixture setup, which runs before each test sets its value.

### Make the album lookup in `FormattedItemMapping` lazy

Reorder `_get()` so `model_keys` is consulted before `self.album`. Album
precedence for shared fields (`for_path=True`) is decided with the static set
`Album.item_keys` instead of `self.album_keys`, which requires the album
object.

- `Album.item_keys` is `_field_names - {"artpath", "id"}` (`models.py:364`),
  which is exactly the condition the current `album_keys` builder applies to
  keys that are also item fields (`key in Album.item_keys or key not in
  self.item._fields`, `models.py:243`). Keys outside `model_keys` still reach
  the album through `album_keys`.
- When `included_keys` is an explicit list, keep today's behaviour: the
  current builder treats every listed key as album-eligible, so the album is
  still fetched. Only the `ALL_KEYS` case takes the static fast path.
- Album *flex* fields still need the album object; they are only reachable when
  the key is not in `model_keys`, so the fetch happens then, not per item.
- Guard `item.album_id` pointing at a missing album: `self.album` can be `None`
  even when the id is set.
- `FormattedMapping.__iter__` (`all_keys`) legitimately needs every key when a
  caller asks for all fields (`beet export -l`, `print_data`), so it keeps
  fetching the album.

Note the limit: a path format that uses `$album`/`$albumartist` still needs the
album, because album-level values take precedence. The plugin decision below is
what removes that query from an `alt update` run.

### Single destination and album map in `alternatives`

In the plugin, `_items_actions()` yields the computed destination with the
actions, and `External.update()` reuses it. Albums are read once:

```
albums  = list(lib.albums())        # one query (+ flex batch)
items   = list(lib.items())         # one query (+ flex batch)
by_album = group items by album_id  # in memory
matched  = {album.id for album in albums if query.match(album)}
for item in items:
    if item.id in matched_ids or query.match(item): ...
```

`item_change_actions()` and `_sync_art()` take the album object from the map
instead of calling `item.get_album()`; `update_art()` uses the same grouped
items instead of `album.items()` per album. The loop also seeds
`item._cached_album` from the map before computing a destination, so
`FormattedItemMapping` reuses the run's album instead of fetching one per
track. This is a private attribute, but the plugin already drives beets
through private state and the alternative (a library-wide album cache) is a
larger semantic change.

This lives in the `beets-alternatives` submodule, so the change includes a
submodule commit and a pin bump.

## Risks / Trade-offs

- **Path-format regressions are existential** → the cache and laziness change
  lookups, never the computed string. Verify with `test/test_util.py`,
  `test/test_library.py` (path format cases) and a before/after comparison of
  `destination()` output over a fixture library.
- **Stale cached config in tests or long-lived processes** → explicit cache
  clear in the test fixture; document that the resolver is config-snapshot
  state, like `max_filename_length`.
- **Album precedence change for `for_path=True`** → membership is tested with
  `Album.item_keys`, which reproduces the current builder's condition for keys
  that are item fields; the explicit-`included_keys` branch keeps the existing
  code path. Add a test asserting an album field wins over the item field for
  path formats, and one asserting an item-only template performs no album
  query.
- **Seeding `item._cached_album`** → a seeded album is refreshed by
  `Album.load()` only when the database revision changed, so a run that stores
  nothing issues no album queries and a run that stores does. Add a test that a
  seeded album is reused rather than re-fetched.
- **Submodule pin drift** → the tasks include bumping the pin and a CI-visible
  commit message; the plugin change is independently testable in its own repo.

## Migration Plan

No data migration. `docs/changelog.rst` gets an entry under the current
unreleased section. Rollback is a revert of the three code changes and the
submodule pin.
