# Signalpost Hackathon: Expectations, Evaluation Criteria, and Repository Comparison Specification

**Purpose:** Use this document as the stable rubric for auditing our Signalpost company-research agent, its `envelopes` output, supporting artifacts, and competing repositories. It is designed to be reused as more repositories and result folders are supplied.

**Prepared:** 14 October 2026  
**Competition deadline shown on the official page:** 21 October 2026  
**Primary reference:** [Official Signalpost challenge page](https://builderr.ai/challenges/signalpost)  
**Supporting participant materials used:** the supplied `Full Brief`, `signalpost-agent-playbook.md`, `signalpost-learning-harness`, and `signalpost-sources.md`.

> **Important distinction:** This is an evaluation and comparison specification, not a claim that our repository has passed the checks. No source-code repository or `envelopes` output from our agent or competitors was included in the material used to prepare this document. Actual strengths, bugs, coverage, and score estimates must be established from those artifacts.

---

## 1. Executive interpretation: what the hackathon is actually asking us to build

Signalpost is not merely a web scraper, company directory, or summarization chatbot. It is a reproducible research agent that receives a Norwegian organisation number, resolves the exact legal entity, finds useful public information through permitted sources, records claim-level evidence, returns a complete result even when information is unavailable, and refreshes the profile without losing history or inventing changes.

The central design principle is:

**First prove that a fact belongs to the exact legal entity and is supported by evidence. Then maximize coverage.**

The competition gives 50 of 100 points to recall and coverage, but an increase in data volume is not automatically an improvement. Unsupported claims, invalid evidence, duplicate facts, or a material wrong-company match can reduce quality or prevent a run from qualifying. A smaller profile with precise identity and verifiable evidence is safer than a rich profile contaminated with another company's facts.

### The agent must do seven jobs

1. **Accept runtime input:** Work from the organisation numbers supplied to the agent during the official run. Do not depend on a precomputed list of profiles or on choosing companies yourself.
2. **Resolve identity:** Anchor the company to authoritative Norwegian registry data, then prove whether a candidate website, brand, person, or profile belongs to that exact entity.
3. **Discover and extract:** Find legal identity, annual-account information, leadership, registered workplaces, official website and company-owned profiles, hiring, and dated public activity where available.
4. **Attach evidence:** Every material claim needs a permitted source and provenance. Search-result rank alone is not evidence for a published fact.
5. **Represent absence honestly:** Return an explicit status for unavailable, blocked, inapplicable, ambiguous, or failed information. Never replace missing information with zero or omit the company result.
6. **Refresh safely:** Preserve previous snapshots and values; identify material changes using evidence from both the previous and new states; avoid duplicate records and false changes on reruns.
7. **Run reproducibly at batch scale:** Work from a clean install, with a pinned code version, declared dependencies, one evaluator command, declared APIs/models/licences and expected cost, within the official time and resource budget.

---

## 2. Competition rules and operational requirements

The challenge page is the source of truth if an informal summary conflicts with the official challenge instructions. The following facts are derived from the supplied participant brief and the official challenge page.

| Requirement | What it means for our repository | Evidence to inspect |
|---|---|---|
| Input is a Norwegian organisation number | The official batch determines which companies are processed. Input must not be silently replaced by our own manifest. | CLI entry point, batch reader, input parser, README command, run logs |
| Must handle unseen companies | Runtime discovery must work even if the entity was not precomputed or previously researched. | Code paths, caches, registry lookups, candidate-generation flow, tests with unfamiliar inputs |
| Batch processing is required | One run must process the whole supplied batch, not only one company at a time as a manual workflow. | Batch orchestration, concurrency/queues, output count reconciliation |
| Exactly one terminal envelope per input | A company with no usable facts still needs a result and explicit status. Missing result rows are not equivalent to `not_available`. | `envelopes` output, input/output organisation-number reconciliation, error handling |
| Output must be reproducible | Pin the exact submitted commit and dependencies; provide one command an evaluator can paste. | README, lockfile, install instructions, CLI arguments, clean-environment report |
| 100-company smoke test is required | Provide a result or report proving one complete result per input for a 100-company test. | Smoke-test artifacts, counts, run summary, representative envelopes |
| Precomputed profiles do not determine the competition score | Large local output folders can help development but must not substitute for runtime research on the official batch. | Data dependencies, cache behavior, code that loads manifests or saved profiles |
| Time and resource budgets apply | The agent must finish the official batch and emit all results within the budget; no official time limit is specified in this document, so do not invent one. | Runtime metrics, retries, timeouts, per-domain limits, memory and API usage |
| Declared technologies and cost | State models, APIs, licences/access terms and expected cost per official run. | README/submission notes, config, source connectors, API clients, cost metrics |
| Exact version matters | An official evaluation applies to the frozen submitted version, not a later unsubmitted version. | Commit hash, lockfile, frozen prompts/model config/source policy |
| Security and integrity | No fabricated evidence, unsafe source handling, leaked secrets, evaluator tampering, or non-reproducible personal account credentials. | Secret scanning, URL handling, dependency config, connector policies, logs |

### Submission package checklist

A submission-ready package should identify:

- Repository URL.
- Exact commit hash to evaluate.
- One copy-pasteable evaluator command.
- Dependency installation and version-pinning instructions.
- A 100-company smoke-test result or report.
- Models, APIs, source connectors, access licences/terms and any external account requirements.
- Expected third-party/API cost for an official batch.
- Agent name and contact details for the results.

### Version schedule and public status

The supplied brief says the first submission is version 1, with up to four revised exact commit hashes before 18 October 2026, for five versions total. The challenge page displayed a 21 October close date when checked on 14 October 2026. Since deadlines, board data, and revision policy are time-sensitive, recheck the official page before acting on them.

The live page currently showed **37 submissions, 28 assessed, 0 qualified**, with the board text marked **Reviewed 5 October 2026**. Its leading displayed score was 60.80/100. Treat those as a dated snapshot, not a guarantee that the board has not changed since this document was generated. Source: [Signalpost challenge page](https://builderr.ai/challenges/signalpost).

---

## 3. Scoring rubric: use the weights exactly

| Dimension | Weight | Officially stated intent | What our audit should verify |
|---|---:|---|---|
| Recall and coverage | 50 points | Find more of the checked information, across more companies, against the shared checked reference collection. | Coverage by company and information family; unique supported claim count; missing categories; results on identical inputs; candidate and connector success rates. |
| Precision and evidence | 30 points | Ensure facts belong to the exact company and have a valid source and date. Unsupported or wrong facts lose points; a material wrong-company publication can block qualification. | Exact-entity match; claim-to-source support; source permission and durability; retrieval time; effective/reporting period; evidence snippet/span; duplicate and contradiction handling. |
| Synthesis | 12 points | Explain what the company does, what changed, and what remains unknown, with sources supporting conclusions. | Summary is supported by claims; latest accounts and people are interpreted correctly; change summaries are evidenced; unknowns/gaps are visible; no hallucinated conclusions. |
| UX | 8 points | Let a user find, compare, and verify information on desktop and mobile. | Readability, navigation, category coverage, source access, provenance visibility, status clarity, comparison of previous/current facts, responsive layout. |
| **Total** | **100 points** | Official-run qualification threshold: **65/100 overall**. The rubric dimensions are not separate qualification thresholds. | Never claim official qualification from a local proxy score. |

### Recall calculation that must not be misunderstood

For each information type, the challenge brief states that the coverage score combines:

- **70% company coverage:** how many companies with a checked opportunity were covered for that information type.
- **30% individual-claim coverage:** how many checked facts were found.

The brief's example: if a checked collection has 50 job postings across 20 companies, and an agent finds 30 postings across 15 companies, claim coverage is 60% and company coverage is 75%. The blended result for that information type is:

`0.70 × 75% + 0.30 × 60% = 70.5%`

That is an information-type coverage calculation feeding the 50-point recall/coverage dimension, **not** a direct 70.5/100 total score. The supplied brief also states that if a batch has fewer than 15 positive company-field opportunities across at least three external field families, recall is reported as not measured for that batch rather than as 0%.

### Do not reverse-engineer the whole scoring formula from public leaderboard rows

The public page describes high-level weights and the company/fact blend, but the complete detailed per-field formula and the exact evaluator implementation are in the official evaluation contract. Do not claim to calculate the official score exactly unless that contract, its schema, and the checked reference data are available. A local report should label any score as a **proxy estimate**, explain the formula and assumptions, and separate measured outputs from inferred scores.

### What the public board suggests, without proving

The current official page's dated 5 October snapshot shows several entries with strong synthesis and UX values but much lower recall values. This makes broad, supported coverage a plausible high-value improvement target. It does **not** prove that a particular connector or architecture will win, nor does it mean we should trade away company identity or evidence quality for raw volume.

---

## 4. Required profile coverage: information families to look for

Each repository will be assessed for implementation, execution evidence, and observed output. The existence of a connector in code is not proof that it successfully produces usable claims.

| Information family | Expected substance | Strong evidence | Common failure mode |
|---|---|---|---|
| Legal identity | Organisation number, legal name, legal form, registered address, industry/classification, latest filing year and other registry facts supported by the contract. | Official registry record and consistent entity identifiers. | Name-only match; outdated registered address; facts from a similarly named company. |
| Public identity and brand | Verified website, public brand/aliases, company-owned profiles, and clear relationship to the registered entity. | Organisation number on the site or several independent, consistent corroborating signals, then reverse verification. | Treating a parent, subsidiary, franchise, sister company or public brand as the exact legal entity. |
| Annual accounts and financial history | Latest available filing, relevant values, reporting period, currency/units, and history where available. | Official annual-account record/PDF/API and clear reporting-period association. | Mixing fiscal years; fabricated zero for missing values; misreading PDF columns; confusing revenue, operating result and net result. |
| Leadership and roles | Current/registered management and board roles where supported, with effective date or source date. | Official roles/register evidence; public identities linked back to the exact company. | Outdated job title; person with same name; confusing group-level and subsidiary roles. |
| Registered workplaces and locations | Number of registered subunits/workplaces and location details where available. | Official subunit records and/or verified company location pages with relationship clearly labelled. | Counting web-office pages as registered subunits, or publishing group office locations as this entity's locations. |
| Official website and owned profiles | Website identity decision, pages discovered, relevant links and verified company-owned social/video destinations where permitted. | Site-linked profiles plus identity corroboration; archived/snapshotted source evidence. | Publishing a candidate domain before proving ownership; treating search results as evidence. |
| Hiring and jobs | Current or dated job postings, role, location, publication/close date when available, job URL and freshness. | Employer career page or permitted/licensed job source, checked at retrieval time. | Expired postings presented as current; duplicates from aggregators; job belonging to a different group entity. |
| News and public activity | Dated announcements, product launches, public updates or company activity from permitted sources. | Company newsroom or licensed/permitted source with publication date and target entity check. | Undated facts, social scraping without permission, repeated reposts counted as distinct claims. |
| Claim-level evidence | Each claim points to an identifiable source and has retrieval time, reporting/effective date where relevant, and provenance. | Source URL/ID plus immutable snapshot hash and a selector, text span, table cell or PDF page. | One generic source list for a whole profile with no way to verify individual claims. |
| Refresh and change history | Current value, previous supported value, first/last observed time, change type/materiality, evidence for old and new values. | Re-running unchanged input is idempotent; true changes are emitted and prior history remains intact. | Overwriting history; emitting false changes because source retrieval failed; duplicating the same claim on each run. |
| Availability and failure state | Clear state for found, absent, blocked, irrelevant, ambiguous, or failed data. | Distinct enum values and explanatory reasons where useful. | Empty string/zero meaning multiple things; omission of the entire company envelope after a failure. |

### Profile usefulness test

The product should support basic decisions such as whether to apply to, sell to, partner with, or invest in a company. Therefore, the final summary should clearly answer, where evidence allows: what the company does, who leads it, where it operates, how the latest filed figures look, whether it appears to be hiring, what has changed, and which important information remains unknown.

---

## 5. `envelopes` is the primary result artifact, but not the only artifact to inspect

The user's result file named `envelopes` is the key observation layer because it represents what the agent emitted. It must be examined alongside the code and run telemetry. A result file alone cannot explain why a claim was missed, whether the code path was executed, what the crawler requested, or whether the output was produced from a cached/precomputed record.

### Inspect all available artifacts

| Artifact | Questions to answer |
|---|---|
| `envelopes.jsonl` or equivalent | Is there exactly one terminal result for every input? Are statuses valid? Are claims supported? Are fields absent, empty, zero or duplicated? Does each claim identify its source/date/reporting period? |
| `profiles.jsonl` or equivalent | How is the product-facing profile structured? Does its content agree with envelopes? Is summary text grounded in claims? |
| `run-summary.json` | Total input count, completed/success/failed counts, run identifier, output paths, runtime, exceptions and final exit status. |
| `run-metrics.json` | Companies/sec, source or API counts, network time, extraction counts, field-family coverage, retries, costs and timing percentiles where tracked. |
| `request-log.jsonl` | Requested/final URL, HTTP status, redirects, timings, errors, source class, retries, and whether actual evidence was fetched. Exclude secrets. |
| `viewer.html` or UI assets | Can users find and verify facts? Are sources, missing states, change history and key fields legible on desktop and mobile? |
| Raw snapshots / cached pages | Are the cited contents the actual fetched content? Are hashes/timestamps stored? Are snapshots immutable? |
| Registry/API connector output | Was authoritative identity fetched and retained? Are organisation numbers preserved as strings where appropriate? |
| Annual-account parsing output | Are all values tied to a reporting period and unit/currency? Are tables/PDF page references preserved? |
| Tests and fixtures | Do tests cover wrong-company cases, missing values, blocked sources, no results, retry behavior, duplicate claims and refresh idempotency? |
| Config, lockfiles, README and run scripts | Can a clean machine install and execute one reproducible command without undocumented local state? |

### Reconcile counts; do not accept summary claims blindly

For every run, compare:

- Unique organisation numbers in input.
- Number of terminal envelopes emitted.
- Number of unique organisation numbers in envelopes.
- Missing inputs and unexpected outputs.
- Duplicate terminal envelopes per company.
- Status counts.
- Claim counts and unique claim keys.
- Evidence-bearing claims versus claims without sufficient evidence.
- Summary metrics versus values recalculated from the actual files.

If `run-summary.json` says 1,000 succeeded but only 964 envelopes are in the output, report the mismatch. Do not silently trust the summary. A status of `failed` is still a terminal result; dropping the envelope is a batch-completeness failure.

### Evidence required per claim

The official material requires a claim's source and retrieval time, plus the relevant reporting period where applicable. The supplied playbook recommends preserving deeper provenance as well:

- Claim value and normalized value where relevant.
- Organisation number and exact-entity decision supporting the claim.
- Source URL or source identifier.
- Requested URL and final URL; redirect chain and response status.
- Retrieval timestamp.
- Effective date or reporting period when relevant.
- Immutable content snapshot/hash.
- Extractor/parser version and extraction method.
- Evidence location: selector, JSON-LD property, character span, table cell, or PDF page.
- Source class and policy/licence/access basis.
- Availability state and reason if the claim cannot be published.

Not every property above is explicitly shown as a mandatory output-envelope field in the public high-level description; they are detailed audit recommendations from the supplied playbook and source policy. Compare the actual output against the complete official evaluation contract before labelling individual schema omissions as formal disqualifiers.

---

## 6. Identity resolution and source policy: hard safety gates

### Identity is not the same as name similarity

The organisation number is the stable key. Legal name, domain, brand, profile handle, address and leader names are clues, not interchangeable identifiers.

The supplied playbook recommends an evidence graph along these lines:

`legal entity → official site candidate → public brand/aliases → leaders/founders → company profile candidate → reverse proof`

Useful support includes the organisation number on the website or a strong combination of legal name, registered address, phone, leadership and independent official evidence. Parent companies, subsidiaries, franchises, sister companies and portfolio brands must be labelled as relationships rather than collapsed into one entity.

**Comparison rule:** record wrong-company publication separately from ordinary missing coverage. A new strategy is not an improvement if it increases the amount of data while introducing material wrong-entity facts.

### Source ladder

The supplied `signalpost-sources.md` and playbook prioritize the following source tiers:

1. Official Norwegian registers and annual accounts.
2. Verified company-owned sites and feeds.
3. Official or licensed platform APIs.
4. Permitted public pages with recorded provenance and access conditions.
5. Search/discovery providers for candidate generation only.

Search results are discovery hints, not the final evidence for a published claim. The underlying permitted source should be revisited and preserved.

### Restricted sites and connectors

The supplied source policy warns that LinkedIn, Meta, Glassdoor, Indeed and similar platforms have access restrictions. The presence of an open-source or unofficial scraper does not establish permission to use it. Private experiments may explore candidate discovery, but a scored submission needs an access method that is permitted, declared and reproducible; published claims must be independently verified from a durable permitted source.

Audit each connector for:

- Declared purpose: candidate discovery or claim evidence.
- Access method and source terms/robots policy as applicable.
- Authentication method and whether credentials are reproducible for the evaluator.
- Rate limits, per-domain budget and retry/backoff behavior.
- Durable, attributable evidence returned from the permitted source.
- Safe URL handling, including redirects and private/internal network destinations.
- No secrets in logs, artifacts, or version control.

### Abstention is a correct outcome when identity is uncertain

If the exact-entity relationship is unclear, the agent should return `ambiguous` or `not_available` as appropriate instead of publishing a likely but unsupported profile. Abstention must still be measured: it is not a way to hide poor coverage.

---

## 7. Availability-state and envelope contract

Use the state names in the supplied brief exactly unless the complete current evaluation contract explicitly says otherwise:

| State | Intended meaning | Audit checks |
|---|---|---|
| `available` | The relevant information was found. | There is a supported value and usable evidence. |
| `not_available` | Research was attempted, but the information was not found. | It is not being used to disguise a crash or skipped lookup. |
| `blocked` | The source refused or prevented access. | Record enough context to distinguish access denial from absence. |
| `not_applicable` | The question does not apply to this company. | The reason is logically valid; it is not just an easy fallback. |
| `ambiguous` | The agent could not verify the exact company/claim. | It avoids publishing an unsupported candidate as fact. |
| `failed` | Processing broke for this company or task. | Error is surfaced; one terminal envelope still exists. |

### Missing is not zero

Do not conflate:

- Zero revenue with unavailable revenue.
- No registered board role with a failed roles lookup.
- No jobs found with blocked job-source access.
- An inapplicable category with one that was not searched.
- A blank field with `not_available` unless the schema explicitly defines that behavior.

Numeric zero can be a valid real fact only when the source supports it. A code path that defaults missing financial fields to `0` is a high-priority audit concern.

### One result per input is a hard operational requirement

At the end of a batch, every input organisation number must map to exactly one terminal envelope. The envelope's internal fields may state that some data families are blocked, missing or ambiguous; that does not justify omitting the whole envelope.

The exact envelope schema must be read from the actual repository and the current official evaluation contract. Do not invent required JSON keys just because a particular competitor uses them.

---

## 8. Refresh, history and idempotency

Signalpost profiles are expected to remain current. The supplied playbook recommends refresh as a diff, rather than overwriting the current profile with whatever a new crawl happens to return.

A strong refresh system stores:

- Current supported value.
- Previous supported value.
- First and last observed timestamps.
- The old and new evidence references.
- A typed change, such as `new_role`, `closed_job`, `new_location`, `new_filing` or `changed_description`.
- Change materiality or a clear rule for highlighting meaningful changes.
- Refresh attempt status and reason if the source could not be checked.

### Required behaviors to test

1. **Same-snapshot rerun:** Running again against unchanged source content must not create duplicate claims or a false change.
2. **True source change:** A changed source value must produce a new current value and a change record with evidence supporting both sides.
3. **Fetch failure:** A transient error must not erase a previously supported value. Show the refresh failure while keeping the old value labelled as last known.
4. **Source disappears:** Distinguish a confirmed removal from a source that merely failed to load.
5. **Outdated jobs/news:** Use publication, close and retrieval dates to avoid presenting old activity as current.
6. **Annual-account history:** Keep reporting periods separate; a new filing should not overwrite all prior history.
7. **Stable claim keys:** Use a deterministic identity for claims so repeated refreshes can be idempotent.
8. **Snapshot immutability:** A source hash should describe the actual captured content and remain unchanged if parsing logic later changes.

The official brief requires idempotent refresh, earlier evidence preservation and avoidance of duplicates. The more detailed mechanics above are the supplied playbook's recommended implementation approach.

---

## 9. Suggested technical architecture: a reference, not a mandatory stack

The playbook explicitly says this is a reference architecture, not a mandated framework. A component should be retained only when blind evaluation shows that it improves supported coverage without increasing wrong-company or unsupported-claim rates.

### Recommended pipeline

1. Read runtime organisation number(s).
2. Fetch authoritative registry identity and filing baseline.
3. Generate candidate official website/domain(s).
4. Verify the website/domain-to-entity relationship.
5. Crawl deterministically: sitemap/static HTML and important known routes first.
6. Extract structured data, then DOM/clean text, then deterministic rules, then model-assisted candidates where needed.
7. Validate field types, dates, units and entity relationships.
8. Attach every published claim to preserved evidence and a status.
9. Generate supported synthesis from approved claims only.
10. Compare against prior supported state and emit refresh changes.
11. Write exactly one terminal envelope per input, plus run summary and audit metrics.
12. Reconcile counts and exit with a clear batch-level result.

### Stack examples from the playbook

These are examples, not requirements:

- Scrapy for crawling queues, throttling, retries, deduplication and per-domain budgets.
- Sitemap and static HTML before browser rendering.
- `extruct` for JSON-LD, OpenGraph and microdata.
- Trafilatura for readable page text.
- Playwright only when a deterministic check shows the page is a JavaScript shell.
- Pydantic or an equivalent typed validator before publication.
- Plain PDF text extraction first; layout parsing or OCR only for difficult annual reports.
- Registry and annual-account APIs/bulk records for official Norwegian facts.
- Pytest, DuckDB/Parquet, and tracing tools for reproducible evaluation and attempt analysis.

### Important architectural trade-off

A more complicated stack is not automatically better. Browser rendering, OCR and LLM extraction introduce cost, time, variability and additional failure modes. Use them as fallbacks when measurable evidence says they recover important supported facts that simpler deterministic paths miss. Do not promote a crawler strategy merely because it downloaded more pages.

---

## 10. How we will compare our repo against each competitor

The comparison needs two separate tracks:

- **Implementation capability:** What the code is designed to do, the controls it contains, its source choices and how it handles failures.
- **Observed output behavior:** What its actual `envelopes`, profiles, logs, metrics, snapshots and UI prove it did.

Neither track can substitute for the other. Code can contain a connector that never runs, while an output file can appear strong because it contains cached or hand-precomputed profiles. Every major conclusion should include a file path, function/class, output example, metric or test that supports it.

### Phase A: establish our own baseline first

Before looking at competitors, record:

- Repository name, commit hash, branch and source snapshot date.
- Entry point and one documented run command.
- Input format and exact test-company IDs.
- Number of input companies and output envelopes.
- Counts by status and category.
- Number of claims by information family.
- Evidence-field completeness and URL/snapshot/span validity where inspectable.
- Identity-resolution method and examples of accepted/rejected candidate domains.
- Runtime, companies/sec, API calls, request counts, errors and cost if available.
- Refresh behavior, history and idempotency evidence.
- All known code paths that produce or modify envelopes.

Keep a baseline manifest so changes can be compared with later revisions.

### Phase B: create a file-and-code inventory for every repository

For every repo, find:

1. Entrypoints and argument parsing.
2. Input/batch loading.
3. Main orchestration and concurrency.
4. Registry and financial connectors.
5. Website/domain candidate discovery and identity gates.
6. Crawling, page priority, sitemap, browser fallback and PDF flow.
7. Extraction, normalization, validation and claim deduplication.
8. Evidence/snapshot persistence.
9. Envelope/profile serialization.
10. Error and availability-state handling.
11. Refresh/history/diff logic.
12. Metrics/logging and output-directory management.
13. UI/viewer code.
14. Tests, fixtures, install/lock files and README.

Then map each information family and requirement to the actual source files/functions. Avoid judging a repo solely by file names or README claims.

### Phase C: normalize the result artifacts

Use a common conceptual record for analysis without forcing repositories to adopt an invented schema:

- Repository label and commit.
- Run ID/date and input-list hash.
- Organisation number.
- Terminal status.
- Information family.
- Claim value and reporting/effective period.
- Claim source URL/source ID.
- Retrieval timestamp.
- Identity decision/evidence.
- Source snapshot/evidence span if present.
- Claim key/deduplication identity if present.
- Refresh/change metadata if present.

Keep original artifacts unchanged. Normalization is an analysis layer only.

### Phase D: ensure the comparison is fair

**Best case:** Run each agent against the exact same input batch, with the same resource limits and equivalent allowed access, and compare outputs under the same reference rubric.

If the competitors' envelopes contain different companies or only different sample sizes:

- Compare schema completeness, status discipline, evidence design, code architecture, tests, and operational robustness.
- Do not claim that one has better recall from raw total claim counts.
- Do not compare raw profile volume without normalizing per company and per available category.
- State clearly that recall/precision comparison is inconclusive without shared inputs and trusted reference labels.

If the official checked reference collection is unavailable, we can compute observable coverage proxies but cannot truthfully reconstruct the official 50-point recall score.

### Phase E: build the comparison matrix

For each requirement, score the evidence rather than aesthetics:

| Field | Meaning |
|---|---|
| Requirement | A specific hackathon requirement or an explicitly labelled recommended best practice. |
| Our repo implementation | Files/functions and behavior found. |
| Our output evidence | Concrete examples and counts from envelopes/logs/metrics. |
| Competitor implementation | Same inspection for each competitor. |
| Competitor output evidence | Same observed-output checks. |
| Compliance state | `PASS`, `PARTIAL`, `FAIL`, `NOT PROVEN`, or `NOT APPLICABLE`. |
| Confidence | High/medium/low with reason. |
| Impact | Disqualifying/critical, high scoring, medium scoring, low scoring, or maintainability. |
| Recommendation | Specific change and file/module boundary if supported by evidence. |
| Validation test | An acceptance test that would demonstrate the change works. |

Never mark something as `PASS` merely because it is described in documentation. `NOT PROVEN` is appropriate when there is no execution evidence.

---

## 11. Proposed audit metrics and tests

The metrics below are a **local engineering audit framework** aligned to the challenge, not a promise that Builderr uses these exact submetrics or thresholds.

### Batch integrity

- `input_count`
- `terminal_envelope_count`
- `unique_input_org_count`
- `unique_output_org_count`
- `missing_org_count`
- `unexpected_org_count`
- `duplicate_terminal_envelopes`
- `valid_terminal_status_rate`
- `batch_completion_rate = terminal_envelope_count / input_count`, checked for duplicate handling rather than accepted as a stand-alone number

Target behavior: each unique input appears exactly once in the terminal outputs. A `failed` envelope counts as a returned result; an omitted result does not.

### Coverage proxies

For each information family, record:

- Companies with at least one supported claim.
- Total supported claims.
- Unique claims after deduplication.
- Claims per company, including zero-claim companies.
- Percentage of companies with the family `available`.
- Percentage marked `not_available`, `blocked`, `not_applicable`, `ambiguous`, `failed`.
- Category-specific crawl/extraction success rate.
- If the same input batch is shared: coverage delta vs our baseline.

Avoid reward-hacking metrics such as pages downloaded or raw posts found. They are diagnostic counts, not evidence of useful recall.

### Precision, identity and evidence proxies

On a hand-reviewed or otherwise trusted sample, count:

- Wrong-company publications, by severity.
- Supported claims / reviewed published claims.
- Claims with source URL/ID.
- Claims with retrieval time.
- Claims with effective/reporting period when relevant.
- Claims with a resolvable evidence span or snapshot reference.
- Claims sourced only from discovery/search results.
- Exact duplicates and near-duplicates.
- Contradictory current values without a conflict policy.
- Numeric claims with correct units/currency/period.
- Claims whose source is permitted and reproducible.

**Must-not-regress guardrail:** zero new material wrong-company publications in a candidate strategy's promotion test. The learning-harness guide recommends zero new material wrong-company publications, no meaningful drop in claim precision, 100% evidence completeness for published material claims, a declared coverage improvement, and budget compliance before promotion. Treat the exact thresholds as local promotion policy unless they also appear in the complete official evaluator contract.

### Refresh tests

- Same input and same source snapshot twice: no duplicate claims, no false change.
- Changed company site text: correct changed field plus old/new evidence.
- New annual filing: new period appears without overwriting prior periods.
- Source blocked during refresh: old supported value remains, failure visible.
- Job closes or disappears: correct state change, not a network-error deletion.
- Website candidate identity weakens or conflicts: no unsafe publication.

### Operational tests

- Clean clone at pinned commit.
- Empty virtual environment.
- Only documented install step, then one documented run command.
- Batch includes companies with rich data, no website, blocked source, ambiguous identity and a processing error.
- Batch scale at 100 and a larger local size such as 1,000, subject to available budget.
- Controlled timeout/retry scenario.
- Output-directory resume/fresh-run behavior.
- No leaked keys or personal-account credentials in artifacts/logs.
- One terminal envelope even for a company-level failure.

### Synthesis and UX tests

- Every factual summary sentence can be traced to one or more approved claims.
- Financial summary labels reporting period, units and currency.
- Unknowns/gaps appear visibly instead of being filled with guesses.
- Material changes include the prior and current value with evidence.
- Reviewer can verify a claim quickly from the profile to the source.
- Desktop and mobile views expose the same essential facts and verification links.

---

## 12. How to prioritize improvements

Use this order. It reflects both explicit competition risk and the scoring weights.

### P0 — Qualification and batch blockers

Fix first:

- Missing or duplicate result envelopes.
- Agent ignores the supplied input batch or requires precomputed target companies.
- Clean install/run command fails.
- Unpinned or undocumented runtime dependencies.
- Material wrong-company publishing.
- Fabricated financial values or unsupported factual claims.
- Missing source/retrieval time/reporting period for material claims.
- Refresh destroys old evidence or generates duplicate records.
- Unsafe or non-reproducible source access; leaked secrets.

### P1 — Recall and coverage improvements with evidence preserved

After P0 is controlled, measure missing fields and categories by company. Prioritize strategies with a reasoned path to raise supported coverage across **more companies**, not only more claims about a few already-rich companies. Possible experiments include official registry fields, role/subunit endpoints, sitemap and targeted page discovery, career/news routes, structured-data extraction, and annual-account PDF fallbacks. Do not assume any particular connector is beneficial until evaluated.

### P2 — Precision, provenance and identity controls

Improve exact-entity linking, reject weak candidates, preserve source snapshots, add claim-level evidence references, type-check outputs, normalize financial periods/units, and reconcile conflicting sources. If a candidate source contributes high volume but low exact-company confidence, tighten its publication gate or keep it candidate-only.

### P3 — Synthesis and UX

Generate summaries only from supported claims; make unknowns explicit; distinguish current and historical data; show useful changes; allow simple source verification and category comparison; check desktop/mobile layout.

### P4 — Cost and latency optimization

Once quality is measured, tune retries, concurrency, caching and source routing. A strategy should be promoted only when the benefit is measured on a comparable corpus, and source budgets remain satisfied. The public tie-break order states: fewer wrong-company facts, higher weighted company recall, then lower declared third-party API cost.

### Change-promotion rule

For every proposed strategy change, create a short experiment record:

- Hypothesis.
- Exact code/config/prompt version.
- Fixed development batch and evaluation date.
- Baseline and challenger metrics.
- Manual review of suspected wrong-company facts.
- Precision/evidence guardrails.
- Coverage improvement and per-company distribution.
- Runtime, calls, errors and cost.
- Decision: promote, reject, or inconclusive.
- Rollback path.

Do not tune to an unofficial guess of the hidden batch. Keep development, validation and final sets separate where practicable, with no organisation or host overlap, as recommended by the playbook.

---

## 13. Information to collect when repositories arrive

Use this intake checklist so the comparison does not quietly make invalid assumptions. Unknown answers may be marked `UNKNOWN`; do not block the entire analysis when evidence is available in files.

### Repo identity and comparison scope

1. Which uploaded repository is **our own repository**? Give it a fixed label such as `OUR_REPO`.
2. What label should be used for each competitor, and which files belong to each one?
3. Are the uploaded code and outputs from the exact same commit/version, or could outputs have been created by a different revision?
4. Should the comparison be static-only, or is running code allowed? Default: inspect statically first and do not execute unknown code without explicit permission.
5. Is the goal maximum official score, robust compliance, lower runtime/cost, or a balanced result? Default: qualify first, then maximize score without increasing wrong-company risk.

### Result comparability

6. Were all agents run against the same organisation-number input list?
7. If yes, where is that input list and what is its hash/version?
8. Do all runs have comparable access to APIs, network and models, or are some outputs from cached/offline runs?
9. Are `envelopes` files final terminal outputs, intermediate outputs or samples?
10. Which file is authoritative if `envelopes`, `profiles`, summary and metrics disagree?
11. Are any precomputed profiles or previously researched company lists being loaded into runtime results?
12. Are official reference labels/gold claims available for a measured recall/precision test? If not, use proxy metrics and state limitations.

### Execution constraints

13. What command produced each run, and what date/commit was used?
14. Are API credentials available in a safe test environment? Never paste real secrets into the conversation or commit them.
15. What time, network, memory or API budget should be assumed for local benchmarking? If unknown, report the limitation.
16. Are we allowed to test clean installation or execute a subset of code, or is the user going to run all tests themselves?
17. Does the user want changes recommended only, or patch-ready file/module proposals as well? Default: report first; do not edit or test the user repo unless explicitly requested.

### Evidence and priorities

18. Which output categories currently matter most to the user: financials, leadership, locations, website identity, jobs, news, refresh, or all equally? Default: all required categories, with priority guided by measurable gaps.
19. Are there known bugs, current benchmark reports or official feedback to include?
20. Should recommendations minimize architectural changes or allow a major rewrite when the evidence justifies it? Default: smallest change that demonstrably improves quality and meets the official contract.

The eventual comparison report must record answers and unresolved unknowns at the top so later updates remain consistent.

---

## 14. Reusable master prompt for analysing our repo and multiple competitor repos

Copy the prompt below into the conversation where repository files are supplied. It is deliberately staged so an early competitor upload does not trigger a premature final ranking.

```text
ROLE
You are a rigorous repository auditor and competition evaluator for the Builderr Signalpost company-research hackathon. Your job is to compare OUR_REPO against one or more competitor repositories and their real output artifacts, then produce an evidence-grounded Markdown report that tells me what is missing in OUR_REPO, what is genuinely strong, what can be improved, and which changes are most likely to improve qualification and score.

AUTHORITATIVE MATERIAL
Use the following first, in this order:
1. The supplied Signalpost participant brief and current official challenge page: https://builderr.ai/challenges/signalpost
2. The supplied signalpost-agent-playbook.md
3. The supplied signalpost-sources.md
4. The supplied signalpost-learning-harness
5. The official evaluation contract if provided: https://builderr.ai/docs/signalpost-evaluation-harness.md
6. Actual repository code, configuration, tests, envelopes, metrics, logs, snapshots and UI output.
Do not replace explicit competition requirements with generic best practices. Label each finding as (a) official requirement, (b) supplied playbook/source-policy recommendation, or (c) your own proposed engineering check.

CORE OBJECTIVE
Our goal is to submit a reproducible agent that takes runtime Norwegian organisation numbers, resolves the exact legal company, finds useful public facts from permitted sources, emits source-linked claims with dates, represents missing/blocked/ambiguous/failed information honestly, refreshes profiles without losing history, and returns exactly one terminal envelope per input under the official time/resource budget.
The score is 50 recall/coverage, 30 precision/evidence, 12 synthesis and 8 UX. Qualification requires 65/100 on an official run. A material wrong-company publication can block qualification, so never recommend sacrificing identity precision merely to increase output volume.

STAGED WORKFLOW
Stage 0: When these instructions are first used, read the available brief and reference files. Confirm the known rubric and list any important evidence still missing. Ask only for missing information that cannot be determined from the files. Do not ask me to restate details already supplied.
Stage 1: I will upload OUR_REPO code and all available result files first. Treat `envelopes` as the primary output artifact, but inspect every relevant file: source modules, batch runner, registry and website connectors, output schemas, profiles, run-summary, run-metrics, request logs, raw snapshots, viewer/UI, tests, README, dependency files and configuration. Build an OUR_REPO baseline and report what is proven, what is only claimed by documentation, and what is not proven. Do not compare against a competitor before one is supplied.
Stage 2: I will upload competitor repositories one at a time. Assign stable labels COMPETITOR_1, COMPETITOR_2, and so on unless I provide names. Inspect each in the same way, append its findings to the common comparison matrix, and retain the OUR_REPO baseline. Do not forget prior repositories when a new one arrives.
Stage 3: Wait to issue the final cross-repository ranking until I explicitly say `ALL REPOS PROVIDED — COMPARE NOW`. Until then, give only the latest repo's inventory and updated preliminary matrix. Do not treat a missing upload as proof that a component does not exist if the repository was incomplete.
Stage 4: When I say `ALL REPOS PROVIDED — COMPARE NOW`, deliver the final comparative report and prioritize changes to OUR_REPO. If a useful analysis can already be produced, do it without needless follow-up questions.

INSPECTION RULES
- Do not assume a feature exists because a README mentions it. Trace it to code and, wherever possible, actual execution output.
- Do not assume a feature works because a module exists. Check whether the entrypoint calls it and whether outputs/logs show it executed.
- For every meaningful conclusion, include the concrete file path, function/class, relevant output example, metric or test that supports it. If line numbers are available, cite them.
- Treat code evidence and observed output evidence as separate columns.
- Keep original artifacts immutable. Any normalized comparison records are analysis-only.
- Inspect all files related to `envelopes`, not only that file. Reconcile actual input IDs, envelope IDs, run summary, metrics and request logs.
- Report `PASS`, `PARTIAL`, `FAIL`, `NOT PROVEN` or `NOT APPLICABLE`. Use `NOT PROVEN` when the required output or execution evidence is missing.
- Separate hard/official failures from optional engineering weaknesses and speculative improvements.
- Never fabricate benchmarks, company data, source URLs, run results, scores, code behavior or competitor features.
- Do not execute unknown code, install packages, call external APIs, or expose secrets unless I explicitly authorize the action and the available environment supports it. Static inspection is the default.
- Do not print or repeat secret values. Identify where secrets appear and recommend safe handling.

COMPARABILITY RULES
- First determine whether outputs were generated from the same organisation-number batch, comparable code versions, source access, models and run budgets.
- If inputs are different, compare architecture, schema, evidence quality design, tests and operational resilience, but do not claim a recall winner from raw claim counts.
- If the checked gold/reference collection is unavailable, do not claim an official recall, precision or total score. Calculate only clearly labelled local proxies, explain their denominator and limitations, and state that the true official score cannot be reconstructed from the available evidence.
- Normalize coverage by input-company count and information family. Distinguish company coverage from number of individual claims. Deduplicate claims before counting them.
- Review a sample of claims for exact-company attribution, source support, reporting period, freshness, units/currency and duplicates. A source URL alone is not proof that the page supports the claimed fact.
- Treat wrong-company publications as a separate severe metric. More facts is not better if more facts belong to the wrong entity.

REQUIRED REVIEW AREAS
1. Official-run readiness: runtime batch input, unseen companies, exactly one terminal envelope per input, clean installation, pinned dependencies, one command, declared models/APIs/licences/cost, budget behavior, frozen version and submission artifacts.
2. Envelope contract: schema, required sections as specified by the current contract, valid statuses, missing-versus-zero handling, uniqueness and input/output reconciliation.
3. Coverage: legal identity, annual accounts and history, leadership, registered workplaces, official website/company-owned profiles, jobs, dated public activity, claim evidence and refresh metadata.
4. Identity: organisation-number anchor, registry identity, candidate-domain discovery, exact-entity proof, brand/group/subsidiary boundaries, candidate rejection and abstention behavior.
5. Sources: official/owned/licensed/permitted source hierarchy, search results as candidate discovery only, platform access policy, provenance, redirects, content hash, snapshot and evidence span.
6. Extraction: structured data, DOM, readable text, deterministic rules, model-assisted extraction, schema/type validation, conflict retention, units, periods and currencies.
7. Refresh: stable claim keys, idempotency, snapshot immutability, old/current values, refresh failure behavior, true/false change detection and history.
8. Operations: concurrency, timeouts, retries, throttling, per-domain budgets, caching, resume/fresh behavior, errors, runtime, calls, cost and one-terminal-result guarantees.
9. Synthesis: supported company overview, what changed, known gaps, source-backed conclusions and no unsupported narrative.
10. UX: ease of discovery, source verification, comparison, missing-state clarity, change history and desktop/mobile layout.
11. Tests and documentation: adverse cases, test reproducibility, run instructions, declared limitations and source restrictions.

MUST-CHECK EDGE CASES
- Similar or identical legal names belonging to different companies.
- Brand/parent/subsidiary/franchise confusion.
- Website domain candidate without convincing exact-entity proof.
- Missing company website.
- Empty registry/API response and temporary source failure.
- Blocked source, 404, redirect, malformed HTML or JavaScript-only content.
- Annual-account PDF with several reporting periods, columns, units or currencies.
- No jobs found versus job source blocked versus expired job.
- Duplicate facts from multiple routes or feeds.
- Same snapshot reprocessed twice.
- Source content changes between runs.
- Batch timeout or one company crashes while others finish.
- Missing financial value must not become numeric zero.
- A company fails processing but must still receive one terminal `failed` envelope.

LOCAL AUDIT METRICS
Calculate where the artifacts support it:
- Input companies; terminal envelopes; unique input/output IDs; missing IDs; unexpected IDs; duplicate envelopes; status counts.
- Per-category company coverage and unique supported claim counts.
- Claim-level source and retrieval-date completeness; reporting-period completeness where relevant; evidence-span/snapshot coverage where available.
- Wrong-company findings and reviewed claim support rate on a manually audited sample.
- Duplicate rate and contradictory claims.
- Runtime, companies/sec, request counts, error/retry counts and API cost if recorded.
- Refresh idempotency and false-change outcomes if replay artifacts/tests exist.
State which metrics are calculated directly and which are estimates. Do not turn an unknown metric into zero.

OFFICIAL SCORE HANDLING
Use 50/30/12/8 as the official top-level weights. For recall, preserve the stated 70% company-coverage and 30% claim-coverage distinction for each information family. Do not calculate a fabricated exact total if the checked reference set and full official contract are unavailable. A local weighted score is allowed only as a labelled proxy with a transparent formula. Never label a proxy as an official score or qualification result.

PROMOTION AND RECOMMENDATION RULES
Prioritize in this order:
P0: qualification blockers, missing/duplicate terminal envelopes, clean-run failures, wrong-company publication, fabricated/unsupported material facts, missing required provenance, unsafe/non-reproducible source access, refresh data loss.
P1: measurable increases in supported recall across more companies and information families.
P2: precision, exact identity, source evidence, normalization, contradiction/deduplication controls.
P3: synthesis and desktop/mobile verification UX.
P4: latency, cost and complexity tuning after quality is controlled.
Recommend the smallest changes supported by evidence. Each recommendation must name the likely module/file, implementation idea, expected score dimension, risk, dependencies and an acceptance test. If architecture changes are speculative, mark them as experiments rather than facts.
A strategy should not be called better simply because it fetches more pages or emits more claims. Use a fixed shared evaluation batch and strict wrong-company/precision guardrails where possible.

OUTPUTS REQUIRED FOR THE FINAL REPORT
Create a detailed Markdown report with these sections:
A. Executive conclusion: biggest verified strengths, biggest gaps and highest-leverage next actions.
B. Scope, repo labels, commit IDs, input comparability, assumptions and evidence limitations.
C. Official requirement compliance matrix with exact supporting evidence.
D. Per-repository architecture and call-flow map from organisation number to final envelope.
E. Envelope/result artifact audit and input/output reconciliation.
F. Per-category implementation and observed coverage comparison.
G. Identity/source/evidence quality comparison, including reviewed failure examples.
H. Refresh/history/idempotency audit.
I. Batch reliability, reproducibility, runtime/cost and security review.
J. Synthesis and UX review.
K. Side-by-side comparison table across OUR_REPO and every competitor.
L. Strengths of OUR_REPO supported by evidence; weaknesses/gaps supported by evidence.
M. Prioritized roadmap: P0/P1/P2/P3/P4, specific changes, files/modules, expected impact, risk and acceptance tests.
N. Suggested experiment plan and promotion gates.
O. Unknowns that require a shared run, gold labels, additional artifacts or human review.
P. Final verdict that separates (1) compliance status, (2) observed local proxy metrics, and (3) official score/qualification, which must remain unknown unless genuinely supported.

WRITING STANDARD
Be exact and critical. Do not praise vague effort. Do not hide uncertainty. Do not confuse “not implemented” with “not visible in the supplied files.” Use concise tables, detailed evidence, and reproducible acceptance criteria. Make actionable recommendations that could change what we implement next.
```

---

## 15. Repository handoff checklist

When uploading files, include as much of this as available. If a folder is too large, send a repository archive plus output artifacts; if files are sent separately, label which repo they belong to.

### For OUR_REPO first

- [ ] All source files, especially batch runner and envelope serialization path.
- [ ] `envelopes` output file and a stable sample if the full file is large.
- [ ] `run-summary.json`, `run-metrics.json`, request logs, profile output and viewer/UI.
- [ ] Exact input company list used to create those outputs.
- [ ] Current command, commit hash and date of run.
- [ ] README, requirements/lockfile, tests and configuration with secrets removed.
- [ ] Refresh snapshots or replay fixtures if available.

### For each competitor

- [ ] Stable competitor label.
- [ ] Code and dependencies, preferably from one exact commit.
- [ ] `envelopes` or equivalent output.
- [ ] Any summary/metrics/request logs and UI artifacts.
- [ ] Input batch and run metadata, if available.
- [ ] Notes on whether code can be executed or should be inspected statically only.

### Before final ranking

- [ ] Confirm all repositories have been supplied.
- [ ] Confirm whether result batches match.
- [ ] Confirm which code versions produced each output.
- [ ] Confirm if there is a verified reference/gold collection.
- [ ] Send the exact phrase: `ALL REPOS PROVIDED — COMPARE NOW`.

---

## 16. Source register and interpretation notes

### Primary and supporting sources

1. **Official challenge page:** [https://builderr.ai/challenges/signalpost](https://builderr.ai/challenges/signalpost)  
   Used for high-level scoring, deadline, submission package, batch requirements, qualification, public board snapshot and source links. The page's displayed public board text was marked as reviewed 5 October 2026 when checked on 14 October 2026.
2. **Global Builderr rules:** [https://www.builderr.ai/guidelines](https://www.builderr.ai/guidelines)  
   Used for the general principle that the challenge page controls, exact submission versions matter, stated boundaries must be followed, and evidence must not be fabricated.
3. **Official Signalpost profile/sample site:** [https://builderr.ai/signalpost](https://builderr.ai/signalpost)  
   Useful for seeing how evidence, identity decisions, source retrieval time, company categories, gaps and summaries are presented in the product UI.
4. **Official evaluation contract link from the challenge page:** [https://builderr.ai/docs/signalpost-evaluation-harness.md](https://builderr.ai/docs/signalpost-evaluation-harness.md)  
   Required for exact output schema and detailed evaluator rules. Its direct Markdown content could not be fetched by the web reader during preparation, so this document does not pretend to reproduce every schema field or hidden evaluator detail.
5. **Supplied `Full Brief`:** Participant brief containing universe, scoring details, terminal states, refresh expectations, run rules and revision schedule.
6. **Supplied `signalpost-agent-playbook.md`:** Reference architecture and common traps, including identity resolution, deterministic crawl, source ladder, evidence snapshots, layered extraction, diff refresh and evaluation control loop.
7. **Supplied `signalpost-sources.md`:** Source restrictions, official sources, publication rules and evidence-over-volume principle.
8. **Supplied `signalpost-learning-harness`:** Strategy registry, saved attempts, score ordering, promotion gate and freezing before daily evaluation.

### Interpretation labels used in this document

- **Official requirement:** Explicitly stated in the challenge page or supplied participant brief.
- **Playbook recommendation:** Recommended by supplied Signalpost playbook/source-policy/learning-harness materials but not necessarily a hard requirement of the public top-level rubric.
- **Local audit proposal:** A practical metric or test proposed here to help us compare and improve implementations. It should not be represented as an official Builderr threshold unless verified in the evaluation contract.

### Final rule

The comparison is only as strong as the evidence supplied. The goal is not to find a repository that looks sophisticated; it is to identify which implementation demonstrably produces more **supported, exact-company, current and reproducible information** while meeting the official run contract.
