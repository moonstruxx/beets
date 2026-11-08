# Spec Delta

## Purpose

Defines how the set of active plugins is resolved at startup, so a run loads exactly
the plugins the user listed and the deprecated MusicBrainz switch cannot change that
set behind their back.

## ADDED Requirements

### Requirement: The active plugin set comes from the plugin list only

The set of plugins loaded for a run SHALL be exactly the plugins named in the
`plugins` option, minus those named in `disabled_plugins`. The deprecated
`musicbrainz.enabled` option SHALL NOT add or remove the `musicbrainz` plugin and
SHALL NOT emit a deprecation message.

#### Scenario: Deprecated option does not enable MusicBrainz

- **WHEN** the `plugins` option omits `musicbrainz`
- **AND** `musicbrainz.enabled` is set to true
- **THEN** `musicbrainz` SHALL NOT be among the loaded plugins

#### Scenario: Deprecated option does not disable MusicBrainz

- **WHEN** the `plugins` option includes `musicbrainz`
- **AND** `musicbrainz.enabled` is set to false
- **THEN** `musicbrainz` SHALL be among the loaded plugins

#### Scenario: MusicBrainz loads from the default plugin list

- **WHEN** beets runs with its default configuration
- **THEN** `musicbrainz` SHALL be among the loaded plugins
