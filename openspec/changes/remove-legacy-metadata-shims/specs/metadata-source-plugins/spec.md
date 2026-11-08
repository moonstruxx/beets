# Spec Delta

## Purpose

Defines the contract a plugin fulfils to act as a metadata source for the auto-tagger,
and how the cross-source penalty of each source is configured and applied.

## ADDED Requirements

### Requirement: Metadata sources are declared by subclassing the metadata-source plugin class

A plugin SHALL be treated as a metadata source only when its class subclasses the
metadata-source plugin base class. Declaring a bare `data_source` attribute on a
general plugin SHALL NOT be sufficient, and the system SHALL NOT add, copy, or
otherwise fabricate metadata-source methods or properties on such a class at
definition time.

#### Scenario: Legacy plugin shape is not a metadata source

- **WHEN** a loaded plugin declares a `data_source` value without subclassing the
  metadata-source plugin base class
- **THEN** it SHALL NOT appear in the set of registered metadata sources
- **AND** it SHALL NOT gain any metadata-source method or property
- **AND** loading it SHALL NOT emit a deprecation warning

#### Scenario: Subclassing plugin is a metadata source

- **WHEN** a loaded plugin subclasses the metadata-source plugin base class
- **THEN** it SHALL appear in the set of registered metadata sources

#### Scenario: Source discovery is type-based

- **WHEN** the set of registered metadata sources is computed
- **THEN** each loaded plugin SHALL be classified by its type, not by the presence
  of a `data_source` attribute

### Requirement: Only the supported penalty option configures a source penalty

A metadata source's cross-source penalty SHALL be read exclusively from its
`data_source_mismatch_penalty` configuration option. The deprecated `source_weight`
option SHALL be ignored, SHALL NOT override the supported option, and SHALL NOT
produce a user-facing or maintainer-facing deprecation message.

#### Scenario: Deprecated option is ignored

- **WHEN** a metadata source is configured with `source_weight` and without
  `data_source_mismatch_penalty`
- **THEN** its penalty SHALL be the metadata-source default
- **AND** no deprecation message SHALL be emitted

#### Scenario: Supported option still applies

- **WHEN** a metadata source is configured with `data_source_mismatch_penalty`
- **THEN** that value SHALL be used as its cross-source penalty
