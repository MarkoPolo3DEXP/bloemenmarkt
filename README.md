# bloemenmarkt

Tuleap Product Line Engineering (PLE) build: Engineering Artifact and
Configuration Management on self-hosted Tuleap Enterprise Edition, using
Tuleap's native trackers to implement an ALM-style system aligned with
INCOSE PLE recommendations and ISO/IEC 2655x (particularly ISO/IEC 26580).
Multiple products draw their configurations from one shared Feature Catalog.

This repo holds the canonical tracker definitions (importable Tuleap XML)
and the design/handover documentation for that build. It is not application
source code — see [`docs/handover.md`](docs/handover.md) for the full
design rationale, platform constraints, and test dataset.

## Repository structure

```
tuleap/trackers/
├── feature_catalog/   Product, Feature, Feature Constraint
├── configuration/      Variant, Configuration Baseline
└── engineering/         Requirement, Design, Test Case, Document
docs/
└── handover.md          Full design handover: architecture, constraints, test data, open items
```

The three tracker folders mirror the system's three conceptual layers:

- **feature_catalog/** — the shared, rarely-changing domain vocabulary
  (products, the feature tree, cross-tree Requires/Excludes constraints)
  that everything else is tagged against.
- **configuration/** — ties a Product's feature selection (Variant) and a
  frozen point-in-time snapshot (Configuration Baseline) together.
- **engineering/** — versioned engineering content (Requirement, Design,
  Test Case, Document). Every artifact here is an **immutable version
  node** in a git-like DAG: once another artifact links back to it as a
  parent, it is never edited again — evolution happens by creating a new
  artifact and linking it in. See `docs/handover.md` for the
  `logical_id` / `branch` / `version_label` versioning convention shared
  by all four engineering trackers.

## Status

9 of 10 planned trackers are drafted and imported into the sandbox Tuleap
project. `change_request.xml` is the next one, not yet drafted. See
`docs/handover.md` → "Open items / next steps" for the full list.
