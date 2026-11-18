# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- `wyre-email-signature` plugin (0.1.1): skill that generates Outlook- and
  Gmail-safe WYRE AI / WYRE Technology HTML signatures (inputs, brand
  constants, Cadence booking, optional event banner via `events.md`,
  install steps), with Aaron Sachs example HTML for Technology (with and
  without Channel Partners Orlando) and WYRE AI. Replaces the retired
  sig.wyre.ai generator website; does not modify `wyre-sig`. Event
  banners are opt-in; Technology examples omit unconfirmed social links;
  paste fallback uses the user’s generated HTML.
- Marketplace scaffolding: `.claude-plugin/marketplace.json`, contribution
  guide, skill template, and CI validators (marketplace drift, doc
  references, tool anchoring) adapted from msp-claude-plugins.
- `sage100` plugin (0.1.0): six read-only skills over Conduit's generic
  `mssql` connector — schema-and-conventions (with curated table
  reference), connecting-and-setup (Premium direct + ProvideX
  linked-server bridge), sales-and-ar, purchasing-and-ap,
  gl-and-financials, inventory-and-items — plus README and GOVERNANCE.
- Research backbone under `docs/research/`: Sage 100 database access
  (editions, ODBC, dialect, table map) and the Conduit SQL connector tool
  contract.
