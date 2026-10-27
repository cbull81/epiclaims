# `epiclaims` — Design Specification

**Audience:** this document is addressed to the coding agent that will build this package. It is the result of a structured design interview with the package's author (a graduate student building this as a final project for a statistical computing course, using their own experience with CMS research-identifiable Medicare files as the reference use case). Where this spec is silent or ambiguous, stop and ask rather than guess — several sections below call out specific places where guessing is actively dangerous (invented ICD-10 code lists, invented CMS file layouts).

## 1. Mission

Claims data (health insurance claims, including CMS Medicare Research Identifiable Files) already supports standard notifiable-disease surveillance: case counts over time, usually at a coarse geography (state, sometimes county). **`epiclaims` exists to add granularity that standard surveillance doesn't provide**: stratifying disease cases by patient comorbidities, by complications arising from the case itself, and by fine geography (down to ZIP code) — while handling the statistical disclosure risk that granularity creates.

The package is **not** for the trivial version of this problem. "Does this claim have a qualifying diagnosis code" is a one-line SQL `WHERE` clause and is explicitly out of scope as a standalone feature. Every feature in this spec exists because it is hard to get right with ad hoc code: episode construction (deduplicating one real infection into one case despite many claims), comorbidity/complication extraction with correct time-direction semantics, small-geography stratification, and disclosure-safe publication.

## 2. Scope boundaries

**In scope for v1:**
- A canonical claims data model, independent of source
- A CMS Research Identifiable Files (RIF) adapter, as the first concrete adapter
- Episode construction from raw claims (deduplicating claims into cases)
- Four built-in disease case definitions: **influenza, COVID-19, chlamydia, salmonella**
- Comorbidity extraction (pre-existing conditions), via Elixhauser derivation from claims, with CCW chronic-condition-flag passthrough when available
- Complication/outcome extraction (conditions arising *from* the case): code-based complications, inpatient-care escalation, mortality
- Time series construction: weekly/daily/monthly case counts, arbitrarily stratified by geography, comorbidity, and/or complication status
- An explicit, opt-in small-cell suppression utility
- Two plot types (time series, prevalence bar chart) via matplotlib
- A public, realistic synthetic data generator (no real CMS data may ever appear in this repository)

**Explicitly out of scope for v1** (document these as limitations, do not silently drop them):
- ICD-9-CM support (package is ICD-10-CM only)
- Charlson comorbidity index (Elixhauser only)
- Geographic choropleth mapping (requires a geospatial dependency stack disproportionate to this package's core)
- Rate computation (cases ÷ population) — package produces **counts only**; dividing by a user-supplied denominator is left to the user
- A command-line interface — this is a library, used from Python/pandas, not a shell tool
- `polars` support — `pandas` is the only dataframe dependency

**Planned, not yet built** (call this out in README as a roadmap, distinct from "permanently out of scope"):
- ICD-9-CM support, for users with pre-Oct-2015 Medicare claims
- Charlson index as a second comorbidity option
- Choropleth mapping

## 3. Glossary

Precise definitions matter here; several of these terms are easy to conflate and conflating them was an explicit failure mode caught during design.

- **Claim**: one row of billing data — one encounter, stay, or service line, carrying a beneficiary identifier, a service date, a claim type/setting, zero or more diagnosis codes, and geography fields.
- **Case**: one real episode of a disease in one beneficiary, as reconstructed from one or more claims. A single hospitalization can generate many claims; they should collapse into **one case**, not several.
- **Case definition**: the rule that determines which claims qualify as evidence of a given disease — a set of ICD-10-CM code prefixes, optionally restricted by claim type and/or diagnosis position.
- **Anchor date**: the service date (admission date, for inpatient claims) of the **earliest** qualifying claim in an episode. This is the case's date for all downstream time series purposes.
- **Reinfection window** (`reinfection_window_days`): the number of days after the anchor date during which further qualifying claims are folded into the *same* case rather than starting a *new* one. For diseases with meaningful biological immunity (flu, COVID) this approximates the expected non-reinfection period. For diseases without lasting immunity (chlamydia, salmonella) it instead approximates how long one treatment episode's claims typically trail the index encounter — it is *not* claiming biological immunity for those diseases, and the spec and docstrings must say so explicitly to avoid misleading a reader. A qualifying claim appearing *after* this window is a **new, separate case** (reinfection), never silently merged into the first.
- **Comorbidity**: a condition known to be present in the beneficiary **before** the case's anchor date (backward-looking, lookback window).
- **Complication**: an outcome or condition arising **after and because of** the case (forward-looking, follow-up window). Distinct from a comorbidity in both time direction and meaning — do not conflate them in code, naming, or documentation.
- **Stratification**: grouping cases by one or more dimensions (geography, comorbidity flags, complication flags) when building a time series. Dimensions may be combined arbitrarily.
- **Cell suppression**: replacing small counts (below a threshold) in a stratified output with a non-identifying placeholder, to reduce re-identification risk before external publication. This package treats suppression as a **separate, explicit, opt-in step**, never an automatic side effect of aggregation.

## 4. Canonical data model

The core engine (episode construction, comorbidity/complication extraction, stratification, plotting) operates **only** on this canonical schema, never on any source-specific format. All source-specific logic lives in adapters (Section 8).

There are **three canonical inputs**, two of which are optional:

### 4.1 Claims table (required)

Wide / claim-grain: **one row per claim.**

| Column | Type | Notes |
|---|---|---|
| `beneficiary_id` | string/int (opaque ID) | |
| `claim_id` | string/int | unique per claim |
| `service_date` | date | admission date for inpatient claims |
| `discharge_date` | date, nullable | inpatient only |
| `claim_type` | categorical string | e.g. `"inpatient"`, `"outpatient"`, `"carrier"` — adapter-defined controlled vocabulary, documented per adapter |
| `diag_codes` | list[str] | ICD-10-CM codes, no decimal-point stripping; position 0 = primary diagnosis |
| `state` | string, nullable | |
| `county` | string, nullable | |
| `zip` | string, nullable | 5-digit ZIP; finest grain the package supports |

### 4.2 Chronic conditions table (optional)

Beneficiary-level, pre-computed condition flags (e.g. CMS CCW-style). One row per beneficiary, one boolean/date column per condition. Any adapter may supply this; it is not CMS-exclusive. Exact condition columns are adapter-defined but must be documented wherever supplied.

### 4.3 Outcomes table (optional)

Beneficiary-level. Minimum required column: `date_of_death` (nullable). Used only for the mortality complication signal (Section 7.3). This is explicitly **not** claims-shaped data — do not try to force it into the claims table.

## 5. Core abstraction: the shared code-matcher

Case definitions, comorbidity definitions, and complication definitions are **the same underlying matching operation** — "does this beneficiary have a claim matching this code set, within this time window, relative to this date" — applied in three different temporal directions. Implement **one** shared matcher class/config and reuse it three ways; do not build three parallel systems.

Suggested shape (naming/ergonomics are the agent's judgment; the *contract* below is not):

```python
@dataclass
class CodeMatcher:
    code_prefixes: list[str]       # ICD-10-CM prefixes, e.g. ["J09", "J10", "J11"]
    claim_types: list[str] | None = None   # None = any claim type
    diagnosis_position: Literal["any", "primary"] = "any"
```

- A **case definition** wraps a `CodeMatcher` plus a `reinfection_window_days: int`. For the four built-in definitions (Section 6), this value ships with a cited default. For **user-supplied custom case definitions, this parameter is required with no package-wide default** — do not invent a fallback number; forcing an explicit choice is intentional, since the right value is a real epidemiological judgment, not a programming convenience.
- A **comorbidity definition** wraps a `CodeMatcher` plus a `lookback_days: int` (default `365`, matching standard claims-based comorbidity-measurement convention), applied **backward** from the case anchor date.
- A **complication definition** wraps a `CodeMatcher` plus a `followup_days: int`, applied **forward** from the case anchor date.

## 6. Episode construction

Given a case definition and a claims table:

1. Find all claims for a beneficiary matching the `CodeMatcher`.
2. Sort by `service_date`.
3. The first matching claim opens an episode; its `service_date` (admission date if inpatient) is the **anchor date**.
4. Any subsequent matching claim within `reinfection_window_days` of the anchor date belongs to the **same** episode.
5. A matching claim **beyond** that window closes the previous episode and opens a **new** one (new anchor date) — this is a new case, not a continuation. Do not merge across the boundary under any circumstance; silently merging reinfections corrupts the surveillance signal this package exists to produce.
6. Output: one row per case, with `beneficiary_id`, `case_anchor_date`, and the claims belonging to that episode (for downstream comorbidity/complication lookups).

Do **not** implement a "one qualifying claim = one case, no dedup" mode. That version of the problem is already trivial in SQL and is explicitly out of scope (Section 2).

## 7. Built-in disease definitions

Ship four built-in case definitions: **influenza, COVID-19, chlamydia, salmonella**. These were deliberately chosen to span distinct transmission mechanisms (respiratory, sexual contact, foodborne) so the episode/comorbidity/complication machinery gets exercised against genuinely different shapes, not three variations on the same disease.

**Critical instruction — read before implementing this section:** do not invent ICD-10-CM code lists, claim-type restrictions, diagnosis-position rules, or reinfection-window values from model memory and present them as settled. Each built-in definition's code set and parameters must be sourced from a current, named **CDC/CSTE surveillance case definition** (or an equivalently citable, published source), with the citation (organization, document title, version/year, and URL if available) recorded in the docstring/source attribute of that definition. If you cannot locate or verify a current citable source during implementation, implement the definition with a clearly marked placeholder code set, flag it with a `# TODO: verify against current CDC/CSTE case definition before use` comment, and surface this to the user rather than shipping an unverified list silently. Every built-in must also carry a user-facing docstring note: **"Built-in definitions are a documented starting point citing CDC/CSTE guidance, not a certified clinical product — review before using for any purpose beyond this course project."**

Each built-in definition specifies its own:
- ICD-10-CM code prefixes
- Claim-type restriction, where CDC/CSTE guidance suggests one (e.g., a definition reliant on lab-confirmed outpatient encounters would restrict claim types accordingly rather than defaulting to "any")
- Diagnosis-position rule (default: any position, per Section 9 rationale — only override to `primary_only` where there's a specific reason)
- `reinfection_window_days`, chosen per-disease: flu/COVID should reflect a real (if partial) immunity/reinfection period; chlamydia/salmonella should reflect realistic claim-trailing duration for one treatment episode, **not** an immunity claim, and documentation must say so explicitly to avoid misleading a reader into thinking these diseases confer lasting immunity.

The README should point users at the same CDC/CSTE case-definition standard as the authoritative source to use when constructing their own custom case definitions, not just for the four built-ins.

## 8. Adapters

### 8.1 Adapter contract

An adapter is any function/class that reads a source-specific data format and produces the three canonical tables (Section 4). The core engine must never import or depend on adapter-specific logic.

### 8.2 CMS RIF adapter (in scope for v1)

Translates CMS Research Identifiable Files into the canonical schema:
- **MedPAR** (inpatient stays) and **Carrier/Outpatient** (physician/facility line-items) → claims table. A beneficiary's claims across these file types during one infection must be combined into a single claims table before episode construction runs.
- **CCW chronic conditions table**, when supplied by the user → chronic conditions table (Section 4.2), used preferentially over Elixhauser-derivation (Section 9.2).
- **Denominator/MBSF file**, when supplied by the user → outcomes table (Section 4.3), specifically `date_of_death`, for the mortality complication signal.

**Important instruction:** exact CMS RIF file layouts, column names, and diagnosis-field conventions (e.g., how many diagnosis-code fields MedPAR carries, their exact column names, SSA vs. FIPS geography coding) are technical details this spec does not hardcode, because they vary by CMS file version/year and the author has direct working knowledge of their specific extracts that this spec does not. **Before writing adapter parsing logic, ask the project author for the exact column layout of their current RIF extracts** (or for CMS's RIF technical documentation for the relevant file year) rather than guessing column names. An adapter built against invented column names will silently fail or silently produce wrong output against real data — treat this as a hard stop, not a place to proceed on a best guess.

The CMS RIF adapter must gracefully handle an absent CCW table (fall back to Section 9.2's Elixhauser derivation) and an absent outcomes/denominator file (skip the mortality complication signal, document that it was skipped, do not error).

## 9. Comorbidities

### 9.1 Mechanism

Comorbidity definitions reuse the shared `CodeMatcher` (Section 5), applied backward from the case anchor date within `lookback_days` (default 365).

### 9.2 Built-in: Elixhauser

Ship the **AHRQ Elixhauser comorbidity index** (ICD-10-CM mapping) as the built-in comorbidity set, derived from the claims table via the shared matcher mechanism. Cite the AHRQ source/version in the implementation. Charlson is a documented future addition, not built for v1.

### 9.3 CCW passthrough (CMS adapter only, with fallback)

When the CMS RIF adapter is given a CCW chronic-conditions table, prefer those **pre-computed, pre-validated** flags over deriving Elixhauser from raw claims — CMS has already done that work, and it's a cheaper, higher-quality signal when available. When no CCW table is supplied, fall back to Elixhauser-derivation from claims. This fallback must be automatic and documented, not an error condition — CCW access is not guaranteed for every user.

## 10. Complications

Three distinct mechanisms, all applied **forward** from the case anchor date. Do not collapse these into one function — they pull from different data and have different failure modes if confused.

1. **Code-based complications**: reuse the shared `CodeMatcher` (Section 5), forward-looking within `followup_days`. E.g., "pneumonia following a flu case" or "sepsis following a salmonella case" — user- or built-in-supplied code sets, same sourcing caution as Section 7 applies if any are shipped as built-ins.
2. **Inpatient-care escalation**: a claim-type/setting signal, not diagnosis-code matching — did a later claim within the follow-up window represent inpatient care for a case that was not already inpatient at its anchor. Implemented against the claims table's `claim_type` field directly.
3. **Mortality**: a date comparison, not claims-based at all — did `date_of_death` (from the outcomes table, Section 4.3) fall within `followup_days` of the case anchor date. Requires the optional outcomes table; must degrade gracefully (documented as unavailable, not an error) when that table isn't supplied.

## 11. Geographic stratification, time series, and suppression

### 11.1 Time series aggregation

- `freq` parameter: `"week"` (default, MMWR-style), `"day"`, or `"month"`.
- Output is **case counts only**. Rate/population-normalized output is explicitly out of scope (Section 2) — document this as "bring your own enrollment denominator and divide" in the README.
- Stratification dimensions (geography: state/county/zip; comorbidity flags; complication flags) may be **combined arbitrarily** — accept a list of grouping columns and group-by on all of them simultaneously. This is the direct mechanism behind the package's stated thesis (Section 1) and must not be restricted to one dimension at a time.

### 11.2 Cell suppression

Implement a **separate, explicitly-named, opt-in function** (e.g. `apply_cell_suppression(df, count_col, threshold=11)`) that replaces counts below `threshold` with a non-identifying placeholder (e.g. `"<11"`, matching the common CMS DUA convention of a floor of 11). This function is **never called automatically** by the aggregation/stratification functions — stratified output defaults to raw counts, because a researcher's own internal, IRB-approved analysis legitimately needs to see true small counts, and the package should not fight that authorized use. Suppression is a deliberate step the user takes before external publication/sharing, and documentation must say so prominently, including a caveat that simple threshold suppression is a baseline control, not a complete disclosure-avoidance solution (e.g., it does not implement complementary/secondary-cell suppression) — state this limitation rather than imply the function makes output fully safe to publish under all circumstances.

## 12. Plotting

Matplotlib only, returning `Axes` objects (not just saving to file) so output composes into larger figures.

1. **Time series line plot** — cases per time bin, one line per stratification group when the input is stratified.
2. **Prevalence bar chart** — e.g. percentage of cases with each comorbidity or complication flag, ranked.

Geographic choropleth mapping is explicitly deferred (Section 2) — do not pull in `geopandas` or any boundary-file dependency for v1.

## 13. Synthetic data generator

Ship as **public API** (e.g. `epiclaims.synthetic`), not a private test fixture — this package's entire premise depends on DUA-gated data that can never appear in this public repository, so a user must be able to evaluate and explore the package without their own CMS extract in hand.

Requirements:
- Produces all three canonical tables (claims, optionally chronic conditions, optionally outcomes) with correct dtypes and plausible value ranges — this generator is also the executable specification of the canonical schema, so keep it tightly aligned with Section 4 as the schema evolves.
- Signal should **resemble real epidemiological dynamics**, not pure noise: seasonal curves for flu/COVID-shaped diseases, plausible age-correlated comorbidity prevalence, plausible claim-type distributions — good enough that demo plots look meaningful, not statistically validated against real surveillance data.
- Output must be **unmistakably marked as synthetic** — a clear naming convention, prominent docstring warnings, and ideally a watermark-style column or identifier — so a shared demo file can never be mistaken for real beneficiary data.
- Used by: the test suite (pytest, Section 14), the README quickstart, and any user wanting to try the package before requesting their own CMS data access.

## 14. Testing

- `pytest`, testing exclusively against the synthetic data generator (Section 13). **Real CMS data must never appear in this repository, in test fixtures, in CI, or in example notebooks**, per CMS DUA restrictions on research-identifiable files.
- Cover at minimum: episode construction (including the reinfection-window boundary case), each built-in disease definition's code-matching, Elixhauser derivation vs. CCW passthrough (and the fallback between them), all three complication mechanisms, arbitrary multi-dimension stratification, and the suppression utility's threshold behavior.

## 15. Packaging

- Distribution/import name: **`epiclaims`**
- Python: **3.10+**
- License: **MIT**
- Dependencies: **`pandas`** (core), **`matplotlib`** (plotting). No `polars`, no `geopandas`, no CLI framework.
- **No command-line interface** — pure Python library API, used from notebooks/scripts.
- Published to **PyPI**, source on a public GitHub repository (course deliverable requires a public artifact).

## 16. Course context (for scheduling, not a design constraint on the code)

- 20% of course grade; AI-assisted build explicitly permitted and expected.
- Proposal due Mon Oct 26, 2026 (this spec is written to double as that proposal's technical backbone).
- Checkpoint: Mon Nov 23, 2026.
- Final due: Thu Dec 17, 2026, 3:30 PM.
- Required deliverables: the public PyPI package + GitHub repo, plus a 5–10 minute recorded video presentation (not addressed by this spec — a separate task closer to the deadline).

## 17. Open items requiring the author's input before/during implementation

Do not proceed past these silently — flag and ask:

1. Exact CMS RIF file column layouts for the author's specific extracts (Section 8.2).
2. Confirmation/sourcing of the CDC/CSTE case definitions for the four built-in diseases (Section 7) — citations must be verified against current CDC/CSTE guidance at implementation time, not assumed from training data.
3. Whether the author's CCW extract's exact chronic-condition column names/coding should be hardcoded into the CMS adapter or documented as a configurable mapping.
