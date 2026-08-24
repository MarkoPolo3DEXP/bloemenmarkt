# Handover: Tuleap PLE Engineering Artifact / Configuration Management build

## Objective

Building a Product Line Engineering (PLE) capability on self-hosted Tuleap Enterprise Edition,
using Tuleap's native "trackers" feature to create an ALM-style Engineering Artifact management
system. Aligned with INCOSE PLE recommendations and the ISO/IEC 2655x series (particularly
ISO/IEC 26580). Multiple products draw configurations from one shared Feature Catalog.

**Accepted limitations** (explicitly out of scope):
- No rule engine to validate whether a requested configuration is compliant with business rules
  (e.g. a Variant violating a Requires/Excludes constraint will NOT be flagged automatically).
- No handling of the evolution of the Feature Catalog itself (may be addressed later).

## Environment

- Self-hosted Tuleap **Enterprise Edition**.
- Sandbox project created from the **Issue Tracking** template (deliberately chosen over
  Scrum/Kanban/SAFe templates, which ship with their own predefined "Feature"/Epic/Story trackers
  that would collide semantically and by name with what we're building).
- **No site-admin access** — this drove several design decisions below.
- Connected to GitHub repo `MarkoPolo3DEXP/bloemenmarkt` via a GitHub App integration usable from
  Claude Code (not available from the claude.ai chat interface this handover was written in).

## Key platform constraints discovered (drove real design decisions)

1. **Custom artifact link natures require site-admin access** (site-admin → Trackers → Artifact
   link types is platform-wide, not per-project). We don't have that access, so every tracker uses
   Tuleap's default, unnamed **"Related to"** nature wherever a typed relationship was desired.
   Direction/meaning is documented via title wording and tracker `<description>` text instead of
   an enforced link type. Only Tuleap's *built-in reserved* natures (e.g. `Child`/`Parent`) are
   available without admin rights, and those ARE used for the Feature hierarchy tree.
2. **Only one Artifact Link field is allowed per tracker** (confirmed platform constraint, not a
   UI quirk — the UI greys out a second "Artifact Link" field type with the message "This field
   type cannot be added twice on the same tracker"). This ruled out using separate dedicated link
   fields to disambiguate relationship types (e.g. couldn't add distinct `Requires`/`Excludes`
   link fields on Feature). Workaround used: **reify the relationship as its own tracker/artifact**
   when it needs a type attribute a plain link can't carry (see Feature Constraint below). When no
   type attribute is needed (e.g. Configuration Baseline linking to many pinned artifacts), we
   rely on Tuleap's artifact detail page automatically grouping linked artifacts by tracker type,
   and skip reification.
3. **Tuleap's native Baseline plugin doesn't fit**: Enterprise-only, still "early delivery"/beta,
   and scoped specifically to Backlog **milestones**, not arbitrary artifact sets. We built our
   own `Configuration Baseline` tracker instead of relying on it.
4. **Shortname (`item_name`) collisions with Tuleap's own reserved cross-reference keywords**:
   hit this with `document` (collides with Tuleap's native Documents/docman service) — renamed
   tracker item_name to `doc_artifact`, display name to "Generic Document" to avoid confusion.
   Proactively avoided the same issue for the baseline tracker by naming it item_name
   `config_baseline`, display name "Configuration Baseline" (Tuleap's native Baselines plugin
   likely reserves `baseline`).
5. **No safe way to re-import/update an already-imported tracker's structure via XML** once it
   has artifacts in it. The "Create a new tracker → From an XML file" UI path only *creates* new
   trackers, it doesn't update existing ones, and doesn't copy artifacts. There's a
   `tuleap import-project-xml --update` CLI option in principle, but it requires site-admin/shell
   (`codendiadm`) access we don't have, and the Tuleap team itself has an open request to remove
   that option (reliability concerns). **Practical rule going forward: once a tracker has real
   data in it, add new fields via the GUI (Tracker administration → Fields), not by re-importing
   XML.** Keep the canonical XML file in this repo updated afterward to match, as documentation /
   for future fresh imports elsewhere.
6. Tuleap tracker XML: a `date` field type must NOT have a `<properties default_value_type="0"/>`
   child (caused a "XML does not have the correct format" import error) — omit `<properties>`
   entirely for simple date fields.

## Tracker architecture (all XML files already drafted, most imported and tested)

All trackers use the same system field set (`aid`/`subby`/`subon`/`lud` = id/submitted_by/
submitted_on/last_update_date) plus a single generic `art_link` field named "Links" for all
relationships. Colors are just for visual distinction in the Tuleap UI, no semantic meaning.

### 1. Product — `product` — imported & tested ✅
Top-level anchor: one artifact per product/product family drawing from the shared Feature Catalog.
Fields: `product_name` (title), `product_code`, `status` (Active/End-of-life), `description`.

### 2. Feature — `feature` — imported & tested ✅
The shared Feature Catalog. Cardinality-based (ISO/IEC 26580 style): `feature_type`
(Mandatory/Optional/Alternative/Or). `feature_group_id` (string) disambiguates multiple sibling
Alternative/Or choice-groups under the same parent (needed because a parent can have more than one
independent choice-group among its children, and the tree alone can't tell them apart).
Hierarchy (parent/child) is built using Tuleap's reserved `Child`/`Parent` link nature on the
existing Links field — no extra field needed, and no admin dependency.
Fields: `feature_name` (title), `feature_type`, `feature_group_id`, `description`, `status`
(Proposed/Active/Deprecated).

### 3. Feature Constraint — `feature_constraint` — imported & tested ✅
Reifies a cross-tree Requires/Excludes constraint between two Features as its own artifact
(workaround for the one-Links-field-per-tracker limit — see constraint #2 above). Links to the two
constrained Features via the generic "Related to" nature; direction/meaning lives in the title
text (e.g. "Regenerative Braking requires Electric or Hybrid") since the link itself can't encode
which end is source vs target.
Fields: `constraint_title` (title), `constraint_type` (Requires/Excludes), `notes`.

### 4. Variant — `variant` — imported & tested ✅
A complete feature selection (bill-of-features) for one specific Product — matches the
"configured variant" terminology used in ISO/IEC 26580 itself (verified via web search), though
this differs from the narrower automotive usage of "variant" (a single variation point's resolved
value, e.g. AUTOSAR variant coding) — a clarifying sentence about this is baked into the tracker's
`<description>` in English.
Links to exactly one Product, and to every Feature it selects.
Fields: `variant_name` (title), `product_code` (denormalized copy of the linked Product's code —
see "Denormalization tradeoff" note below), `status` (Draft/Active/Retired), `description`.

### 5. Requirement — `requirement` — imported & tested ✅ (reference implementation of the
   versioning pattern — replicated as-is to Design/Test Case/Generic Document below)
Each artifact is one **immutable version node** in a git-like DAG, not an editable living
document: never edit an artifact once another has linked back to it as a parent; create a new
artifact and link it instead. Multiple parent links = a branch merge. Tuleap does NOT enforce this
immutability technically — it's a process discipline, same category of gap as the "no rule engine"
limitation accepted at the start.
Fields:
- `requirement_title` (title)
- `logical_id` (string) — stable identity shared across every version/branch of "the same"
  requirement, e.g. `REQ-BRK-003`. Never changes.
- `branch` (string, free text, not a predefined list) — e.g. `main`, `UEV-branch`.
- `version_label` (string, NOT integer — deliberately changed from an earlier integer design
  since real version schemes are often alphanumeric/patterned, e.g. `PN006`) — a manually
  maintained sequence label: increments along a branch, RESETS to the branch's starting label
  whenever a node's branch differs from its parent's (i.e. a fork). Not automated yet — would need
  a small script against the Tuleap REST API (or the Enterprise tracker-functions/post-action
  plugin) that looks up the parent's branch+version_label on artifact creation. Flagged as a
  phase-two candidate, not built. NOTE: being a text field, it sorts alphabetically in reports —
  zero-pad (`v01`, `v02`...) if double-digit correct ordering matters.
- `status` (Draft/In review/Approved/Obsolete) — artifact's own review maturity, distinct from the
  immutability discipline above.
- `description`
- `Links` — parent version(s), and Feature(s) this requirement realizes.

### 6. Design — `design` — imported ✅ (structure identical to Requirement)
Same `logical_id`/`branch`/`version_label`/`status` pattern. Title field: `design_title`.
Links → parent version(s), and the Requirement(s) this design **satisfies**.

### 7. Test Case — `test_case` — imported ✅ (structure identical to Requirement)
Same pattern. Title field: `test_case_title`.
Links → parent version(s), and the Requirement(s) this test case **verifies**.
Note: `status` here is the artifact's own review maturity (Draft/In review/Approved/Obsolete), NOT
a pass/fail execution result — execution-result tracking is a distinct, unaddressed concern.

### 8. Generic Document — item_name `doc_artifact`, display name "Generic Document" —
   imported ✅ (structure identical to Requirement)
Renamed from "Document"/`document` specifically to avoid the Tuleap Documents-service shortname
collision (see constraint #4 above). Title field: `document_title`.
Links → parent version(s), and any Requirement/Design/Test Case/Feature/other artifact it documents.

### 9. Configuration Baseline — item_name `config_baseline`, display name "Configuration
   Baseline" — imported ✅
The frozen intersection of exactly one Variant (which features, for one Product) and one point on
the evolution/branch axis: pins the exact version node of every Requirement/Design/Test
Case/Generic Document/Feature Constraint current for that Variant at that point.
**No separate "Baseline Item" sub-tracker** — originally planned, then dropped: Tuleap's artifact
detail page already groups linked artifacts by tracker type automatically, so one Baseline
artifact just links directly (single Links field) to Product + Variant + every pinned artifact,
and they display grouped by type without needing reification (unlike Feature Constraint, which
needed reification because it required a *type* attribute a plain link can't carry).
Treated as immutable once Published — same discipline as the Engineering Artifact trackers.
Fields: `baseline_name` (title, e.g. "UEV A-v1 @ 1.0"), `product_code` (denormalized),
`variant_name` (denormalized), `version_label` (the BASELINE's own release identifier — distinct
from the version_label field inside individual pinned Requirement/Design/etc artifacts),
`baseline_date` (date field — no `<properties>` child, see constraint #6), `status`
(Draft/Published/Superseded), `description`.

### 10. Change Request — NOT YET DRAFTED — next planned tracker
Was going to include: `Affected Artifacts` (Links, multi), `Target Branch`, impact-analysis
text field or report, workflow Submitted → Analyzed → Approved → Implemented/Rejected. Needs to
answer "which variants/products are affected" — likely via a report/query chaining
Change Request → Affected Artifacts → their Feature links → selecting Variants → Product, rather
than a dedicated field, given the single-Links-field constraint. Not designed in detail yet.

## Denormalization tradeoff (applies to `product_code` on Variant and Configuration Baseline,
   `variant_name` on Configuration Baseline)

Per-tracker reports can only filter/display that tracker's own fields — they can't reach across a
link to filter by a field on the *linked* artifact (that needs Tuleap's separate, heavier
cross-tracker search feature). So these text fields duplicate data that's also expressed via a
Links relationship, purely so simple per-tracker reports can filter/sort by it (e.g. "all Variants
for Product UEV") without navigating links. Real tradeoff: nothing keeps these in sync with the
actual link automatically (no formula/computed field for this in Tuleap) — they can drift if
someone changes a link and forgets to update the text field. Accepted as a convenience,
acknowledged as a risk; could be dropped in favor of always using cross-tracker search if the sync
risk turns out to matter more in practice than the convenience.

## Test dataset built so far

### Feature Catalog (3 root trees)
- **Braking Control** (Mandatory) → ABS (Optional), Regenerative Braking (Optional),
  Basic Controller / Advanced Controller (Alternative, group `controller-type`),
  Wheel Speed Sensors / Yaw Sensor / Pressure Sensor (Or, group `sensor-package`)
- **Powertrain** (Mandatory) → Electric / Combustion / Hybrid (Alternative, group
  `powertrain-choice`) — deliberately renamed from "Electric Powertrain" since that name read
  oddly with Combustion as a child of it
- **Diagnostics Level** (Mandatory) → Basic / Extended / Remote (Or, group `diagnostics-level`)

### Feature Constraints (2)
- "Regenerative Braking requires Electric or Hybrid" (Requires) — links Regenerative Braking,
  Electric, Hybrid
- "Basic Controller excludes Electric" (Excludes) — links Basic Controller, Electric

### Products (2)
- Urban EV Platform, code `UEV`
- Commercial Combustion Platform, code `CCP`

### Variants (3)
- **A-v1** (UEV, Active) — Electric, ABS, Regenerative Braking, Advanced Controller,
  Wheel Speed Sensors + Yaw Sensor, Basic + Remote diagnostics. Constraint-clean.
- **B-v1** (CCP, Active) — Combustion, ABS, Basic Controller, Wheel Speed Sensors, Extended
  diagnostics. Constraint-clean (Regenerative Braking deliberately NOT selected).
- **X-v1** (UEV, **Draft**, intentionally invalid) — Electric + Basic Controller: directly
  violates the "Basic Controller excludes Electric" constraint. Left in place on purpose to
  demonstrate that Tuleap will NOT flag this automatically — proves the "no rule engine"
  limitation concretely. Do not promote to Active.

### Requirement DAG — logical_id `REQ-BRK-003` (realizes Feature: Braking Control)
- v1 (branch `main`, version_label `1`, status Approved, no parent — root)
- v2 (branch `main`, version_label `2`, status Approved, parent = v1-main) — normal evolution
- v1 (branch `UEV-branch`, version_label `1`, status Draft, parent = v1-main, NOT v2-main) —
  deliberate fork point, proves non-linear/branching versioning. Not yet merged back.

### Design / Test Case / Generic Document (one each, all linking to REQ-BRK-003 v2-main)
- Design `DES-BRK-003` v1 (main) — "Braking Control — Design"
- Test Case `TC-BRK-003` v1 (main) — "Braking Control — wet-surface stopping distance test"
- Generic Document `DOC-BRK-003` v1 (main) — "Braking Control — Design Rationale Note", also
  linking to DES-BRK-003 v1

### Configuration Baseline (1)
- "UEV A-v1 @ 1.0" — product_code UEV, variant_name A-v1, version_label 1.0, status Published.
  Links → Urban EV Platform (Product), A-v1 (Variant), REQ-BRK-003 v2-main, DES-BRK-003 v1-main,
  TC-BRK-003 v1-main, DOC-BRK-003 v1-main.

## Files (already generated, should be in this repo or attached to this handover)

`product.xml`, `feature.xml`, `feature_constraint.xml`, `variant.xml`, `requirement.xml`,
`design.xml`, `test_case.xml`, `document.xml` (item_name doc_artifact), `config_baseline.xml`.

**Known drift**: `requirement.xml`'s `version_label` field was added to the live Tuleap tracker via
GUI (per constraint #5 above) after the tracker already had artifacts — confirm the XML file's
`version_label` field definition (string, not int) matches what's actually live before treating
the XML as fully authoritative for a fresh re-import.

## Open items / next steps

1. Draft `change_request.xml` (Change Request tracker) — not yet designed in detail, see above.
2. Design the actual impact-analysis report/query for Change Request (cross-tracker: affected
   artifacts → Feature links → selecting Variants → Product).
3. Decide whether to build a second Configuration Baseline (e.g. "UEV A-v1 @ 1.1") to exercise the
   "mostly unchanged, one artifact revised" baseline-diff pattern.
4. Decide whether to exercise a DAG merge node (REQ-BRK-003 v3-main with two parents: v2-main and
   v1-UEV-branch) — sketched but not built, would prove branch-merge support end to end.
5. Revisit whether `product_code`/`variant_name` denormalized fields are worth keeping given the
   sync-drift risk (open question, not yet decided either way).
6. If/when site-admin access becomes available: replace the "Related to" placeholder nature
   everywhere with real custom natures (`is_derived_from`, `implements_feature`, `requires_feature`,
   `excludes_feature`, `selects`, etc.) — this would also let us reconsider whether Feature
   Constraint still needs to be reified as its own tracker, or could become a plain typed link.
