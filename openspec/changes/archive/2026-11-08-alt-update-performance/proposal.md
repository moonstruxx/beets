# Proposal

## Why

`beet alt update` is slow even when nothing needs syncing: a no-op run over a
3,600-item collection takes 10.2 s (2.8 ms/item, measured on the dev machine),
so a real library takes minutes for a sync that writes nothing. Almost all of
that time is path computation — each destination is computed twice per item and
every path legalization re-reads config and recompiles a regex — plus one
database round trip per track to fetch the album the template may not even use.

## What Changes

- Cache the resolver returned by `beets.util.get_empty_component_replace()` so
  `sanitize_path()` stops reading `empty_component_replace` from the config and
  recompiling its regex on every call. Measured: `legalize_path()` drops from
  69 µs to 34 µs per call.
- Resolve an item's album in `FormattedItemMapping` only when a template key
  actually needs an album field, instead of on every `_get()`.
- In the `beets-alternatives` plugin (submodule), compute each item's
  destination once per update and reuse it for the action pass, and look up
  each album once instead of once per item.
- No behavior change: same files land at the same paths, the same rows are
  written, and the same events fire.

## Capabilities

### New Capabilities

None: this is a pure performance change with no observable behavior difference.
The change sets `skip_specs: true`.

### Modified Capabilities

None.

## Impact

- `beets/util/__init__.py` — `get_empty_component_replace`, `sanitize_path`.
- `beets/library/models.py` — `FormattedItemMapping`.
- `beets-alternatives/beetsplug/alternatives.py` (submodule) —
  `_items_actions`, `_matched_item_action`, `External.update`.
- Every path-format consumer benefits: `beet move`, `beet import`,
  `beet list -f`, and any plugin that calls `Item.destination()` or
  `evaluate_template()` per object.
