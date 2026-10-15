# P0_IMPLEMENTATION_PLAN.md

Repository: `signalpost-company-agent-main` (branch `main`, HEAD `a1434d9 s006`, as uploaded on 15 Oct 2026)
Scope: P0-2, P0-3, P0-4, P0-5, P0-7. **P0-1 (output hygiene / merge-conflict cleanup) is excluded.**
Status of this document: **planning only. No application code, dependency file, configuration or artifact was modified while writing it. Nothing was executed.**

All paths below are relative to the repository root. Line numbers refer to the uploaded snapshot and may drift by a few lines once earlier phases are applied; always re-locate by function name.

---

## 1. Objective and scope

Make the public envelope written to `out/<run_id>/envelopes.jsonl` (and copied to `result/envelopes.jsonl`) comply with the official Signalpost contract for the five approved P0 items:

| ID | Problem (observed) | Goal |
|---|---|---|
| P0-2 | `modules[*].state` contains `not_found` (also capable of `source_error`, `not_fetched`), outside the official six states | Public module states use only `available, not_available, blocked, not_applicable, ambiguous, failed`, mapped by meaning, from one central function |
| P0-3 | `employees` claim is `availability: "available"` with `value: null` (7 of 9 companies in run `run-20261014-010747-274033`) | No public claim is ever `available` with a missing value; absence becomes `not_available` with the registry evidence preserved |
| P0-4 | Evidence pointing at `https://data.brreg.no/...` is published with `source_class: "company_owned"` | BRREG evidence is always classified as an official registry source |
| P0-5 | OCR needs `pdftoppm` + `tesseract` (+ `nor` language data), undeclared; on a machine without them every OCR company ends in `status: "error"` | One reproducible behaviour: OCR is opt-in, capability is detected once, missing capability is reported explicitly and never breaks the run |
| P0-7 | Employees changed 11 to 10 (`810034882`) and 7 to 5 (`810182482`) between runs, `changes[]` stayed empty | A refresh run given a previous envelope file emits correct, evidence-referenced `changes[]`, and emits nothing when nothing changed or a retrieval failed |

Out of scope: see section 10.

---

## 2. Repository architecture relevant to P0

> `docs/CODEBASE_MAP.md` is partly outdated and itself contains unresolved merge-conflict markers. Its "fat envelope is the live output" statement (section 2, row 2) is **wrong for the current code**: `scripts/build_output_contract.py` defaults to `--envelope-format public` and calls `to_public_envelope()`. Trust the code, not that file.

| Layer | File | Key symbols | Role for P0 |
|---|---|---|---|
| Launcher | `run.py` | `main()` (L260), `configure_tesseract_path()` (L32), `clear_run_outputs()` (L53), command build (L389-404) | Entry point. OCR is **off by default** (`--include-workforce-ocr` opt-in, L275). Will forward `--previous-envelopes` (P0-7). |
| Orchestrator | `scripts/run_agent.py` | `main()` (L200), `run_stage()` (L238), `stage_fingerprint()` (L120), stage 5 OCR (L390-405), stage 5b prior-year (L407-420), stage 6 `convert_cmd` (L426-438), stage 6b validation (L441-452) | Runs every stage as a subprocess; resume markers in `out/<run>/stages/*.done` |
| Stage 1 | `scripts/run_competition_batch.py` | `collect_profile()` (L319), `retry_source_error_modules()` (L225), crash fallbacks `_build_failure_profile()` (L277) and the `except` block (L367-418) | Builds `profile.evidence[module]` records with **internal** statuses |
| BRREG client | `src/norway_company_agent/http.py` | `fetch_json()` (L49), `FetchResult` (L17) | Returns `status=200` on success, `404/410` directly, `status=0` + `error="HTTP <code>"` after exhausting retries, `status=200` + `error="JSONDecodeError"` + `body=None` on bad JSON |
| BRREG modules | `src/norway_company_agent/official.py` | `fetch_official_modules()` (L470), `_classified()` (L209), `normalize_entity()` (L192), `normalize_financials()` (L110), `normalize_financial_history()` (L132) | Converts HTTP results into evidence records |
| Website | `src/norway_company_agent/website.py` | `fetch_website()` (L453); no-website return at **L463**; HTTPError branch L640-646 | Produces the `website` evidence record (including the mislabelled BRREG one) |
| Identity gate | `src/norway_company_agent/identity.py` | `apply_website_identity_gate()` (L454) | `REJECT` rewrites website status to `source_error`; `REVIEW` keeps `available` with `identity_assessment.publishable = False` |
| Evidence factory | `src/norway_company_agent/evidence.py` | `EvidenceStatus` (L7), `evidence()` (L40) | Internal status enum; `source_class = source_type` |
| Vocabulary | `src/norway_company_agent/vocabulary.py` | `MODULE_STATES` (L14), `module_state()` (L66), `module_states()` (L87), `derive_terminal_status()` (L101), `invalid_module_states()` (L121) | **Internal** state vocabulary, currently also emitted publicly |
| Public envelope | `src/norway_company_agent/output_contract.py` | `to_public_envelope()` (L199), `add_evidence()` inner fn (L209-235), registry claim loop (L237-252), `_complete_claims()` (L365), `conversion_failure_envelope()` (L380), `_availability()` (L415), `_is_empty_checked()` (L434), `_website_claim()` (L442), `_claim()` (L496), `_claim_span()` (L514) | Builds claims, evidence, modules, changes |
| Change detection | `src/norway_company_agent/changes.py` | `diff_claims()` (L184), `_diff_array()` (L296), `_change_record()` (L340) | Exists and is called by `to_public_envelope()` **only when `previous_envelope` is passed. Nothing in the pipeline passes it.** |
| Stage 6 | `scripts/build_output_contract.py` | `main()` (L119), `build_envelope()` (L96), `build_fat()` (L75) | Calls `to_public_envelope(fat)` with no `previous_envelope` and no `metrics` |
| Stage 6b | `scripts/validate_envelopes.py` | `check_enums()` (L193), `check_terminal_status()` (L251), `validate()` (L280), `CHECK_NAMES` (L49) | Validates modules against the **internal** enum, so `not_found` passes |
| OCR stage 5 | `archive/v2-crawler/run_annual_report_workforce_connector.py` | `ocr_pdf()` (L118), `collect()` (L152), missing-tool branch (L185-198), `main()` (L253) | LIVE despite the `archive/` path (called by path from `run_agent.py`) |
| OCR stage 5b | `archive/v2-crawler/extract_prior_year_financials.py` | `main()` (L140) | Reads OCR text from the **persistent** `cache/workforce-cache/` |
| OCR diagnostic | `scripts/check_tesseract.py` | `main()` | Off-path diagnostic; already checks the `nor` language pack |
| Docs / deps | `README.md` (L39-43 empty OCR code block), `requirements.txt`, `setup.bat`, `run.bat` | | No system-dependency declaration anywhere |

---

## 3. Current data flow and affected modules

```
run.py main()
  -> scripts/run_agent.py main()
     1  run_competition_batch.py collect_profile()
          official.fetch_official_modules() -> http.fetch_json() -> official._classified()     [P0-2]
          batch.backfill_from_registry_live()
          website.fetch_website()  (L463 BRREG URL + registry_linked_company_website)          [P0-4]
          identity.apply_website_identity_gate()  (REJECT -> source_error)                     [P0-2]
          retry_source_error_modules()
        -> profiles-stage1.jsonl  (profile.evidence[module] = {status: available|not_found|
                                   not_applicable|blocked|source_error, ...})
     2  Tavily / Exa discovery (archive) -> website_discovery / website_discovered records
     3/4 deep crawl + site observations (optional)
     5  run_annual_report_workforce_connector.py (only with --include-workforce-ocr)          [P0-5]
     5b extract_prior_year_financials.py (only if OCR flag AND cache dir exists)              [P0-5]
     6  build_output_contract.py main() -> build_envelope() -> build_fat() -> terminal_envelope()
          -> output_contract.to_public_envelope(fat)            <- previous_envelope NEVER passed [P0-7]
               registry claim loop L247: value may be None, availability stays "available"   [P0-3]
               add_evidence L221: registry_linked_company_website -> "company_owned"         [P0-4]
               modules = vocabulary.module_states(profile)   (internal states leak)          [P0-2]
               run.terminal_status = derive_terminal_status(modules)
               changes = []                                                                   [P0-7]
        -> out/<run>/envelopes.jsonl
     6b validate_envelopes.py (internal enum => not_found passes)                             [P0-2]
  -> run.py copies to result/envelopes.jsonl
```

Two vocabularies already exist and must stay separate:

* **Internal evidence status** (`evidence.EvidenceStatus`, `vocabulary.MODULE_STATES`): `available, not_found, not_applicable, blocked, source_error, not_fetched`. Used by stage 1 retry logic (`retry_source_error_modules`), `batch_state()`, `run_competition_batch.py` report fallback (L550-551 checks `"source_error"`), the fat `registry-envelopes.jsonl`, and the existing test-suite.
* **Public claim availability** (`output_contract.ALLOWED_AVAILABILITY`, L40): already the official six. Claims are already mapped correctly by `_availability()` (L415). Only **module states** and the **terminal-status derivation** leak the internal vocabulary.

Design consequence: **do not change internal statuses.** Add a public mapping layer used only where the public envelope is built and validated.

---

## 4. P0 issue inventory (root causes confirmed in code)

| # | Root cause | Location | Evidence |
|---|---|---|---|
| I-1 | Public `modules` = internal `module_states(profile)` unmapped | `output_contract.py` L323; `vocabulary.py` L66-98 | `group/website/website_discovery.state = "not_found"` in `result/envelopes.jsonl` |
| I-2 | Validator checks modules against internal `MODULE_STATES` | `validate_envelopes.py` L202 via `vocabulary.invalid_module_states()` L133 | `validation-report.json` check 08 passed with `not_found` present |
| I-3 | Terminal status derived from internal names; validator re-derives with the same function | `vocabulary.derive_terminal_status()` L101; `validate_envelopes.py` L257 | Must be re-pointed together with I-1 or check 12 breaks |
| I-4 | Conversion-failure envelope emits internal `source_error` module states | `output_contract.py` L391 | Only in failure path |
| I-5 | HTTP 200 with invalid JSON is classified `available` with an all-`None` normalized value | `http.py` L79-84 + `official.py` L486-492 + `_classified()` L210-211 | `normalize_entity(None)` returns a dict of `None`s, which then becomes available null claims |
| I-6 | Website identity `REVIEW` (unpublishable) keeps module `available`; `REJECT` becomes `source_error` (= "failed") although nothing failed | `identity.py` L466-472; `output_contract._website_claim()` L453-454 already turns REVIEW into an `ambiguous` **claim** | Module state and claim disagree |
| I-7 | Registry claims take `availability` of the whole registry record, not of the field | `output_contract.py` L247-251 | `employees` = `None` but `available` |
| I-8 | `_claim()` never guards `available` + `None` | `output_contract.py` L496-511 | Also reachable via `_observation_availability()` L469-474 (any observation with `metrics: null` becomes available) |
| I-9 | No-website record is built with a BRREG list URL and `source_type="registry_linked_company_website"` | `website.py` L463 | Mapped to `company_owned` by `add_evidence` L221-222 |
| I-10 | `add_evidence` maps by source_class only, never by host | `output_contract.py` L220-222 | Crash fallbacks in `run_competition_batch.py` L296 and L389 can produce the same mislabel |
| I-11 | OCR capability is checked per company with `shutil.which` only (no `nor` language check) and reported as `status: "error"` | connector L185-198, L131 (`-l nor`) | `workforce-report.json`: 7 x `"OCR required but missing executable(s): pdftoppm, tesseract"` |
| I-12 | Prior-year stage reuses OCR text from the persistent `cache/workforce-cache/` regardless of current-run OCR capability | `run_agent.py` L411; prior-year `main()` L162-169 | Output depends on what an earlier run on another machine left in `cache/` |
| I-13 | README OCR instructions are an empty code block; system dependencies undeclared | `README.md` L39-43 | |
| I-14 | No CLI path delivers a previous envelope to stage 6 | `build_output_contract.py` (no arg), `run_agent.py` L426-438, `run.py` L389-404 | `changes: []` in every run |
| I-15 | `diff_claims()` reports value changes when the current retrieval failed, was blocked, or the claim is an unpopulated placeholder | `changes.py` L228-291 | Would create false "disappeared" changes once wired (e.g. OCR on in run N, off in run N+1) |
| I-16 | Change records carry no evidence reference; evidence IDs (`ev-<org>-<n>`) are positional and not stable across runs | `changes.py` L340-363; `output_contract.py` L218 | Old evidence is unreachable from a change |
| I-17 | Older previous files may be fat envelopes without `claims` (the Oct-6 run) | `to_public_envelope()` L329 requires `previous_envelope["claims"]` list | The 11 to 10 example comes from such a file |

---

## 5. Phase dependency order

| Phase | Covers | Depends on | Why this order |
|---|---|---|---|
| 1 | P0-2 state mapping, terminal status, validator enum, HTTP/decoding classification | none | Every later phase reads availability / module state; validator must speak the official enum first |
| 2 | P0-3 null handling, P0-4 source classification | Phase 1 (uses the same `output_contract.py` sections; Phase 1's decode-error fix removes one null source) | Claim values must be honest before they are diffed |
| 3 | P0-5 OCR capability + graceful degradation | none technically; scheduled after 1-2 to keep `output_contract.py` edits grouped | Independent files (`archive/...`, `run_agent.py`, README) |
| 4 | P0-7 refresh change detection | Phases 1 and 2 (diff rules rely on official availability values and on "available implies non-null"); Phase 3 only for the placeholder rule rationale | Largest change; touches CLI plumbing across three entry points |
| 5 | Static review checklist | 1-4 | |

---

## 6. Detailed implementation instructions

### Phase 1: Official state mapping and envelope contract compliance (P0-2)

**Goal.** Public `modules[*].state` and `run.terminal_status` are derived from one central mapping that only emits the six official states, chosen by meaning. Internal statuses and all stage-1 logic stay untouched.

**Existing implementation.**
* `vocabulary.module_state()` L66-84 returns the internal status verbatim and raises on unknown status.
* `vocabulary.module_states()` L87-98 builds the 11 fixed `MODULE_KEYS`; special case: `website_discovered` missing while `website_discovery` exists becomes `not_applicable`.
* `vocabulary.derive_terminal_status()` L101-109 uses `source_error` and `_GAP_STATES = {not_found, blocked, not_fetched}`.
* `output_contract.to_public_envelope()` L323 and L350; `conversion_failure_envelope()` L388-391.
* `validate_envelopes.check_enums()` L202 and `check_terminal_status()` L255-257.
* `official._classified()` L209-214.
* `website.fetch_website()` HTTPError branch L645.

**Mapping table (by meaning, verified against producers).**

| Internal status | Producer(s) and meaning | Public module state |
|---|---|---|
| `available` | BRREG 200 with parsed body; website fetched | `available`, **except** website-family module (`website`, `website_discovered`) whose `value.identity_assessment` is a non-empty dict with `publishable` false (identity `REVIEW`): `ambiguous` |
| `not_found` | BRREG 404/410 (`_classified` L212-213, e.g. `konsernstruktur` 404 = no group); no registry website (`website.py` L463); homepage 404/410 (L645); discovery found no publishable candidate (`run_tavily_discovery.py` L168-175, L193-199) | `not_available` |
| `not_applicable` | `registry` hint record (`batch.profiles_from_inputs` L79-86); discovery skipped because no legal name (Tavily L133-139); derived `website_discovered` (`vocabulary` L93-95) | `not_applicable` |
| `blocked` | SSRF guard, robots.txt, oversize homepage (`website.py` L468, L471, L482) | `blocked` (keep `reason` from `module_state()` L81-83) |
| `source_error` with `value.identity_assessment.decision == "REJECT"` (website family only) | Identity gate decided the site is **not** this company (`identity.py` L466-472). Nothing failed. | `not_available` |
| `source_error` (all other cases) | HTTP 5xx / 429 / network / timeout after retries, decode errors, unsupported content type, crash fallback records | `failed` |
| `not_fetched` (record missing) for `website_discovery` or `website_discovered` while the `website` module is internally `available` | Registry already had a working site, discovery was intentionally skipped (Tavily L123-129 writes no record) | `not_applicable` |
| `not_fetched` for any other non-required module | Stage did not run (no API key, deep crawl skipped) | `not_available` (see unresolved question Q2) |
| `not_fetched` for a `REQUIRED_MODULES` member | Should be impossible (stage 1 always writes a record; crash fallbacks write `source_error`); means a pipeline defect | `failed` |

**HTTP / error classification rules (where they are set).**

| Source | 200 OK | 200 + JSON decode error | 404 / 410 | 401 / 403 / 451 | 429, 5xx, network, timeout |
|---|---|---|---|---|---|
| BRREG (`official._classified`) | `available` | **change:** `source_error` (currently `available`) | `not_found` | `source_error` (unchanged; keeps retry path and makes a refused authoritative source a visible failure) | `source_error` (unchanged) |
| Company website (`website.fetch_website` L645) | `available` | n/a | `not_found` | **change:** `blocked` with note `"HTTP <code>"` (module_state assigns reason `policy`) | `source_error` (unchanged) |

**Step-by-step.**

1. `src/norway_company_agent/vocabulary.py`: add, below `module_states()` (keep everything existing unchanged):
   * `PUBLIC_MODULE_STATES: tuple[str, ...] = ("available", "not_available", "blocked", "not_applicable", "ambiguous", "failed")`. Keep it textually identical to `output_contract.ALLOWED_AVAILABILITY`; do not import `output_contract` here (it imports `vocabulary`, avoid a cycle).
   * `WEBSITE_IDENTITY_MODULES = ("website", "website_discovered")`.
   * `_identity_assessment(record) -> dict | None`: returns `record["value"]["identity_assessment"]` when it is a non-empty dict, else `None`.
   * `public_module_state(internal: dict, *, module: str, record: dict | None, website_internal_state: str) -> dict`: implements the mapping table. Returns `{"state": <public>, "retry_count": internal["retry_count"]}` plus `"reason"` only when the public state is `blocked`. Must raise `UnknownModuleStatusError` for any internal state outside `MODULE_STATES` (preserve the "never silently map unknowns" rule of L69-79).
   * `public_module_states(profile) -> dict[str, dict]`: calls the existing `module_states(profile)` first (so its unknown-status check and `website_discovered` special case still run), then maps each key in `MODULE_KEYS` order via `public_module_state(...)`, passing `website_internal_state = internal["website"]["state"]` and the raw record from `profile["evidence"]`.
   * `derive_public_terminal_status(states) -> str`:
     * `failed` if any `REQUIRED_MODULES` state is `failed`;
     * else `completed_with_gaps` if any `COVERAGE_MODULES` state is in `{"not_available", "blocked", "ambiguous"}`, or a non-required `COVERAGE_MODULES` member is `failed`;
     * else `completed`.
   * `invalid_public_module_states(states) -> list[str]`: copy of `invalid_module_states()` L121-143 but validating against `PUBLIC_MODULE_STATES`. Keep the key-order, `reason` and `retry_count` rules.
2. `src/norway_company_agent/output_contract.py`:
   * Import `public_module_states`, `derive_public_terminal_status` from `.vocabulary`.
   * `to_public_envelope()` L323: `modules = public_module_states(profile)`; L350: `derive_public_terminal_status(modules)`.
   * `conversion_failure_envelope()` L388: `public_module_states(profile)`; fallback L391: `{"state": "failed", "retry_count": 0}`.
   * Optional small refactor (recommended): make `_website_claim()` L451-454 use the same `_identity_assessment` helper semantics so claim `ambiguous` and module `ambiguous` can never disagree. Behaviour must stay identical.
3. `scripts/validate_envelopes.py`:
   * Import `PUBLIC_MODULE_STATES`, `derive_public_terminal_status`, `invalid_public_module_states` instead of the internal ones (keep `MODULE_KEYS`, `SCHEMA_VERSION`, `TERMINAL_STATUSES`).
   * `check_enums()` L202 -> `invalid_public_module_states(...)`.
   * `check_terminal_status()` L255 and L257 -> public functions.
4. `src/norway_company_agent/official.py` `_classified()` L209-214: before the `status == 200` branch, if `result.status == 200 and result.error` (decode error) return `evidence(field, "source_error", ...)` with `note=result.error`. Do not touch 404/410 or other branches.
5. `src/norway_company_agent/website.py` L645: `status = "not_found" if exc.code in {404, 410} else "blocked" if exc.code in {401, 403, 451} else "source_error"`. Leave the note `f"HTTP {exc.code}"` (contains no "robot", so `module_state()` assigns reason `policy`).

**Behaviour to preserve.**
* `vocabulary.module_states()`, `derive_terminal_status()`, `invalid_module_states()`, `batch_state()`, `MODULE_STATES` keep their current semantics; they are used by stage 1, `batch.validate_envelopes()` and the fat `registry-envelopes.jsonl`.
* Module key order and the 11 fixed keys (`MODULE_KEYS`).
* `retry_count` values; `blocked` always has `reason`, no other state has it.
* Claim availability logic (`_availability()`) is already official; do not change it in this phase.
* `--envelope-format fat` output keeps internal states (legacy, not the submission file).

**Edge cases.**
* Identity `REVIEW` on `website_discovered` while `website` is `not_found`: `website_discovered` = `ambiguous`, `website` = `not_available`; terminal `completed_with_gaps`.
* Website `REJECT` after retries: `retry_source_error_modules()` retries `source_error` modules, so the record may carry `retry_count > 0` and a note with retry text. The mapping must read `value.identity_assessment.decision`, not the note. If a retry replaced the value with a non-REJECT `source_error`, map to `failed`.
* `blocked` website gets `reason` computed from note; HTTP 403 note gives `policy`.
* Unknown internal status: must still raise inside `module_states()`; `build_output_contract.py` L167-190 already converts that into a `conversion_failure_envelope` with `failed` claims.

**Expected resulting behaviour.** For the 9 companies in `result/envelopes.jsonl`: `group` becomes `not_available` (8 of 9), `website` and `website_discovery` become `not_available`, `registry` stays `not_applicable`, terminal status stays `completed_with_gaps`, validator check 08 fails if any non-official state is ever emitted again.

**Definition of done (static).** `grep -n '"not_found"\|"source_error"\|"not_fetched"' src/norway_company_agent/output_contract.py scripts/validate_envelopes.py` shows no emission into public `modules`; every write of `modules` in public envelopes goes through `public_module_states()`; validator imports only public helpers for module checks.

---

### Phase 2: Null-value handling and correct source classification (P0-3, P0-4)

**Goal.** (a) No public claim is `available` with a missing value. (b) No BRREG-hosted evidence is published as `company_owned`.

**Existing implementation (P0-3 path).**
`official.normalize_entity()` L192-206 copies `antallAnsatte`, `naeringskode1`, `forretningsadresse`, `sisteInnsendteAarsregnskap` as-is (missing keys become `None`) -> `_classified()` marks the record `available` -> `to_public_envelope()` L247-251 creates every registry claim with the record-level availability -> `_claim()` L496 stores `value=None, availability="available", confidence=0.99`.

**Existing implementation (P0-4 path).**
`website.fetch_website()` L463 returns `evidence("website", "not_found", "registry_linked_company_website", "https://data.brreg.no/enhetsregisteret/api/enheter", ...)` -> `_website_claim()` and `_website_contact_claims()` register it via `add_evidence()` -> L221-222 maps `registry_linked_company_website` to `company_owned`. Same mislabel is reachable from crash fallbacks `run_competition_batch.py` L296 and L389 (website module + BRREG URL).

**Step-by-step (P0-3).**
1. `output_contract.py`: add `_is_missing_value(value) -> bool`: `True` for `None`, for a `str` that is empty after `strip()`, and for an empty `dict`. **`False`, `0`, `0.0` and non-empty containers are real values and must not be treated as missing** (missing is not zero; zero is not missing).
2. `_claim()` L496-511: after the existing `ALLOWED_AVAILABILITY` normalization, add: if `availability == "available"` and `_is_missing_value(value)`: set `availability = "not_available"`, `value = None`, `confidence = None`. Keep `evidence_ids` exactly as passed. This is the single safety net for every claim path (registry, module, observation, website, contact).
3. Registry loop L247-251: compute `value = _nested_get(registry_value, path)` once; pass per-field availability `"not_available" if availability == "available" and _is_missing_value(value) else availability` and the matching `_confidence(...)`. Keep `[registry_id]` as evidence for both cases: the fetched registry record (with its `content_sha256`) is the proof that the field was absent at `retrieved_at`.
4. Do **not** add a `note` or any other key to claims: `validate_envelopes.check_claims()` L113 requires the claim keyset to be exactly `{field, value, availability, confidence, evidence_ids}`.
5. Recommended extension, same defect class (skip only if the user wants a strictly null-only change): `_is_empty_checked()` L434-439 also returns `True` for `field == "financials"` with an empty `records` list and for `field == "financial_history"` with an empty `years` list. Today `normalize_financials()` L111-113 returns `{"records": []}` and the claim is published as `available`.
6. `scripts/validate_envelopes.py`: add check `15_available_claims_have_value` (any claim with `availability == "available"` whose value is `None`, `""` or `{}`), register it in `CHECK_NAMES` and `validate()`. Do not add a raising assertion inside `assert_claim_schema_integrity()`: a raise there turns the whole company into a `conversion_failure_envelope`.

**Step-by-step (P0-4).**
7. `website.py` L463: return `evidence("website", "not_found", "official_registry_live", BRREG_ENTITY_URL, note="Registry field hjemmeside is empty or not a valid URL", source_row_key=organisation_number)` where `BRREG_ENTITY_URL = f"https://data.brreg.no/enhetsregisteret/api/enheter/{organisation_number}"` when `organisation_number` is truthy, else keep the generic list URL. Build the string locally; do not import `official.py` into `website.py` (avoid new import edges).
8. `output_contract.py`: add `_public_source_class(record) -> str` and use it in `add_evidence()` instead of L220-222:
   * `raw = str(record.get("source_class") or record.get("source_type") or "unknown")`
   * `host = (urllib.parse.urlparse(str(record.get("source_url") or "")).hostname or "").lower()`
   * if `host == "brreg.no"` or `host.endswith(".brreg.no")`: return `raw` if `raw.startswith("official_")` else `"official_registry_live"`
   * elif `raw in {"registry_linked_company_website", "company_site"}`: return `"company_owned"`
   * else return `raw`.
9. Leave `run_competition_batch.py` crash-fallback maps (L296, L389) unchanged: the host rule in step 8 already prevents the public mislabel, and the records are synthetic failure markers.
10. `scripts/validate_envelopes.py`: add check `16_registry_sources_not_company_owned` (evidence whose `source_url` host is `brreg.no` / `*.brreg.no` and `source_class == "company_owned"`).

**Behaviour to preserve.** Claim order `CLAIM_FIELD_ORDER`; evidence dedup keys in `add_evidence`; `PUBLIC_EVIDENCE_KEYS`; `bankrupt=false` / `liquidating=false` stay `available`; real company-site evidence (`registry_linked_company_website` with a non-BRREG URL) still becomes `company_owned`; `check_sha256()` semantics (null sha allowed only for evidence not backing an available claim).

**Edge cases.**
* All registry fields missing but registry fetched: all those claims `not_available`, all referencing the registry evidence; evidence is not orphaned (check 05).
* `industry_code` path `("industry", "kode")` when `naeringskode1` is absent: `_nested_get` returns `None` -> `not_available`.
* Observation with `metrics: null` (`_observation_availability()` returns `available`): caught by the `_claim()` guard.
* Website on a domain like `brreg.no.example.com`: `hostname.endswith(".brreg.no")` is false, correct.
* `confidence` must be `None` for every `not_available` claim (validator check 08 and `assert_claim_schema_integrity` rule 8).

**Expected resulting behaviour.** In the 9-company output: 7 `employees` claims become `not_available`, `confidence: null`, still citing `ev-<org>-0`; `ev-<org>-1` becomes `source_class: "official_registry_live"` with an org-specific BRREG URL.

**Dependencies.** Phase 1 (decode-error fix removes the all-`None` registry case at its source; both phases edit `output_contract.py`).

**Definition of done (static).** `_claim()` contains the guard; registry loop computes per-field availability; `add_evidence()` uses `_public_source_class()`; `website.py` L463 no longer uses `registry_linked_company_website`; checks 15 and 16 exist in `CHECK_NAMES` and `validate()`.

---

### Phase 3: OCR dependency and graceful-failure handling (P0-5)

**Decision: Option B (OCR stays out of the default evaluator path; missing capability degrades explicitly).**

| | Option A: declare and manage system deps | Option B: opt-in OCR, capability-gated (selected) |
|---|---|---|
| What it needs | Poppler + Tesseract + `nor` traineddata on every evaluator machine; the repo has no Dockerfile or apt manifest, only `setup.bat` (Windows, pip only) and `run.py`'s Windows path hack (L32-41) | Small code changes in 2 scripts + README text |
| Reproducibility | Depends on evaluator OS packages that pip cannot install | Default command has no system dependency |
| Effort | High (new install path per OS, version pinning outside pip, CI image) | Low |
| Default behaviour today | n/a | Already off by default: `run.py` L275 and `run_agent.py` L209 default `skip_workforce_ocr=True` |
| Risk | Evaluator lacks deps -> same failure as today | Lose OCR-only workforce facts in official runs (these were 0 in the latest run anyway) |

**Existing implementation.** Connector `collect()` checks `shutil.which()` per company (L185-198) and reports `"status": "error"`; `ocr_pdf()` (L118-139) calls `tesseract -l nor` without checking language data, so a missing `nor` pack becomes a `CalledProcessError` caught by the broad `except` (L249-250) as another `"error"`. Nothing crashes the run (the stage is `optional=True` and every company is wrapped), but the outcome depends on the machine and is misreported. Stage 5b runs whenever `cache/workforce-cache/` exists (`run_agent.py` L411), even if this run could not OCR, and reads OCR text cached by earlier runs (`extract_prior_year_financials.py` L162-169).

**Step-by-step.**
1. `archive/v2-crawler/run_annual_report_workforce_connector.py`:
   * Add `detect_ocr_capability() -> dict` next to `ocr_pdf()`: `missing = [n for n in ("pdftoppm", "tesseract") if shutil.which(n) is None]`; if tesseract exists, run `[tesseract, "--list-langs"]` with `capture_output=True, text=True, timeout=15`, wrap in `try/except Exception`; if `"nor"` is not among the listed languages (skip the first header line, as `scripts/check_tesseract.py` L28 does) append `"tesseract-lang:nor"`. Return `{"available": not missing, "missing": missing}`.
   * `main()` L253: call it **once** before the thread pool; pass `ocr_available: bool` into `collect()` as a new keyword argument **with default `True`** (keeps the call signature compatible; `tests/test_annual_report_workforce_connector.py` only uses `extract_candidate`).
   * `collect()` L185-198: replace the per-company `shutil.which` block with `if ocr_required and not ocr_available:` returning status `"ocr_unavailable"` (not `"error"`) with the same detail keys plus `"missing": [...]`. Digital-text PDFs (no OCR required) must continue to be processed exactly as today.
   * Add `"ocr_capability": {...}` to the report dict (L290-297).
2. `scripts/run_agent.py`:
   * Stage 5b gate L411: run prior-year only if `workforce-report.json` exists **and** its `ocr_capability.available` is `true` (read with `try/except (OSError, json.JSONDecodeError)`; on any read problem, skip 5b). Print a one-line reason to stderr when skipped.
   * When `--include-workforce-ocr` is set and capability is unavailable, print one stderr line (e.g. `OCR unavailable (missing: pdftoppm, tesseract); workforce OCR limited to text PDFs, prior-year stage skipped.`). Do not abort.
   * Do not change defaults: OCR stays opt-in.
3. `README.md` L39-43: replace the empty block with the opt-in command `python run.py --organisations <file> --include-workforce-ocr`, state that it is optional, not part of the evaluator command, and requires Poppler (`pdftoppm`), Tesseract and the `nor` language data on `PATH`; mention `python scripts/check_tesseract.py` as the diagnostic. State the evaluator command explicitly as `python run.py --organisations <file>`.
4. `requirements.txt`: no change (system binaries cannot be pinned there). Optionally add a comment line that OCR binaries are optional system tools.

**Behaviour to preserve.** OCR off by default; connector output file names and observation schema; cache file naming; `extract_candidate()` untouched; `run.py configure_tesseract_path()` untouched; stage markers (`run_stage`) unchanged.

**Edge cases.** `tesseract --list-langs` hangs or errors -> capability unavailable; tesseract present but `pdftoppm` missing -> unavailable; `--ocr-pages 0` -> `ocr_required` false (existing L182), so capability is irrelevant; resumed run where `workforce_ocr.done` marker short-circuits stage 5 (`run_stage` L250-266): the gate must still read the existing `workforce-report.json`.

**Expected resulting behaviour.** Default command never touches OCR. With the flag on a machine without OCR tools: text-PDF companies still processed, OCR-needing companies reported as `ocr_unavailable`, prior-year skipped, run completes, envelopes unchanged in shape (workforce / prior_year claims stay `not_available` placeholders).

**Dependencies.** None on code; logically after Phases 1-2.

**Definition of done (static).** No `"status": "error"` path remains for missing OCR tools; capability detected once per run; 5b gated on reported capability; README has a non-empty OCR section and names the evaluator command.

---

### Phase 4: Refresh change detection and stable claim matching (P0-7)

**Goal.** A run started with a previous envelope file emits `changes[]` for real value changes, with old and new evidence references, and emits nothing for unchanged values, failed or blocked retrievals, or claims that were simply not checked in one of the runs.

**What exists and why `changes[]` is empty.**
* `changes.diff_claims()` (L184-293) compares claims by `field` (unique per envelope via `_complete_claims()` `setdefault`), with order-insensitive comparison of nested arrays (`_comparison_value()` L144-163). Scalar `employees` 11 vs 10 **would** produce one `updated` record.
* `to_public_envelope(row, previous_envelope=...)` L329-341 calls it, but `build_output_contract.build_envelope()` L96-112 never passes `previous_envelope`, and no CLI option exists in `build_output_contract.py`, `run_agent.py` or `run.py`. This is the confirmed defect.
* The Oct-6 envelopes (old fat shape: `profile`, no `claims`) cannot be used as `previous_envelope` directly (L329 requires a `claims` list).

**Step-by-step.**

*4.1 CLI plumbing.*
1. `scripts/build_output_contract.py`:
   * Add `--previous-envelopes PATH` (optional) to `main()`.
   * Add `load_previous_envelopes(path) -> dict[str, dict]`:
     * read line by line; skip blank lines; skip lines that fail `json.loads` or are not dicts (print a warning count to stderr; do not abort: stage 6 is required);
     * key by `str(row.get("organisation_number") or (row.get("profile") or {}).get("organisation_number"))`;
     * if a row has a `claims` list (public envelope, or fat with `attach_claims`), use it as is;
     * else if it has a `profile` dict (legacy fat envelope), derive claims with `to_public_envelope(row)` (no previous, no metrics) inside `try/except Exception`; on failure, skip that org with a warning;
     * duplicate organisation numbers: remove the org from the map and warn (ambiguous history: no changes rather than wrong changes).
   * Thread an optional `previous_envelope` through `build_envelope()` into `to_public_envelope(fat, previous_envelope=...)` for the `public` format only. For the `fat` format leave behaviour unchanged.
   * Look up by the same `organisation_number` string already computed at L155-157, so the existing org-mismatch `ValueError` (L331-334) can never fire.
   * Add `"previous_envelopes": <path or null>`, `"previous_envelopes_loaded": <int>`, `"changes_total": <int>` to `contract-report.json` (internal report, not the envelope).
2. `scripts/run_agent.py`:
   * Add `--previous-envelopes` (optional). If given, resolve to an absolute path; if it resolves to the current `envelopes_path` (L425), refuse with `parser.error` (a run cannot be its own previous).
   * Append `["--previous-envelopes", str(path)]` to `convert_cmd` (L426-437).
   * Add `"--previous-envelopes"` to `input_options` in `stage_fingerprint()` L121 so a resumed run re-runs `claims_conversion` when the previous file changes.
3. `run.py`:
   * Add `--previous-envelopes` (type `Path`, optional); resolve relative to `PROJECT_ROOT`; `parser.error` if missing.
   * Because `--fresh` deletes `out/` and `result/envelopes.jsonl` (`clear_run_outputs()` L53-59), copy the previous file to a temporary path outside `out/` and `result/` **before** `clear_run_outputs()` when it lives inside either directory (reuse the pattern of `preserved_source` L297-309), and delete the copy after the run.
   * Forward it in the `run_agent.py` command (L389-404). Also forward it in the `--source-envelopes` branch command (L313-322) so offline rebuilds can produce changes too.
   * Do not auto-default to `result/envelopes.jsonl`: refresh must be explicit (see Q5).

*4.2 Diff correctness (`src/norway_company_agent/changes.py`).*
4. Add `_UNCOMPARABLE_AVAILABILITY = frozenset({"failed", "blocked"})`.
5. Add `_is_placeholder(claim) -> bool`: `claim.get("availability") == "not_available" and not claim.get("evidence_ids")`. These are the fillers created by `output_contract._complete_claims()` L372 when a module or stage did not produce data in that run (OCR off, crawl skipped, no observation).
6. Add `_DEPENDENT_FIELDS = {"phones": "official_website", "emails": "official_website", "website_addresses": "official_website", "social_profiles": "official_website"}` (these are built from the website record in `_website_contact_claims()` L610-709; if the website fetch failed they fall to `not_available` with evidence, which would otherwise look like a real disappearance).
7. In `diff_claims()`, before any change is appended for a field:
   * skip if either side is `_is_placeholder` (not checked in that run);
   * skip if either side's availability is in `_UNCOMPARABLE_AVAILABILITY` (retrieval problem; `errors[]` already records it via `_append_error`);
   * skip if the field is in `_DEPENDENT_FIELDS` and the parent claim (looked up in the same side's claims) has availability in `_UNCOMPARABLE_AVAILABILITY` on either side;
   * keep the existing `_FROZEN_FIELDS` exclusion (`jobs`, `news`).
   * Keep reporting: value changes between two comparable states, and availability changes between comparable states (e.g. `available` <-> `not_available` with evidence, `available` <-> `ambiguous`).
8. Evidence references (subject to Q4, minimal additive keys):
   * extend `diff_claims(organisation_number, previous_claims, current_claims, *, previous_evidence=None, previous_run_id=None)`; positional signature stays compatible;
   * `_change_record()` gains keyword arguments and adds three keys to every record: `current_evidence_ids` (copied from the current claim, resolvable inside the current envelope), `previous_evidence` (list of the previous envelope's evidence objects referenced by the previous claim, copied with exactly `PUBLIC_EVIDENCE_KEYS`; empty list if unavailable; copied because previous IDs do not resolve in the new envelope and are positional), and `previous_run_id`;
   * array item changes (`_diff_array`) carry the same parent-claim references.
9. `output_contract.to_public_envelope()` L335-339: pass `previous_evidence=previous_envelope.get("evidence") or []` and `previous_run_id=((previous_envelope.get("run") or {}).get("run_id") or previous_envelope.get("run_id"))`. Call order is already correct: the diff runs before `envelope["evidence"]` is stripped at L361, and current claims reference the current `evidence_items` IDs.
10. Determinism: keep `sorted(...)` iteration (L225, L310); do not include `retrieved_at`, confidence or evidence IDs in the comparison (the comparison remains on `value` + `availability` only).

*4.3 Validation.*
11. `scripts/validate_envelopes.py`: add check `17_changes_well_formed`: each change has `change_type` in `{"added", "updated", "removed"}`, `organisation_number` equal to the row, every id in `current_evidence_ids` resolves in the row's `evidence`, and no two change records share the same `(field, array_item_key, change_type)`. Do not count `previous_evidence` items as orphan evidence (they are not in `evidence[]`; `check_orphans()` L145-152 only looks at `evidence[]`, so no change needed there).

**Behaviour to preserve.**
* Without `--previous-envelopes`: `changes` stays `[]` (first-run semantics, L340-341).
* `diff_claims()` existing semantics for comparable claims: scalar equality, order-insensitive arrays, stable item keys (`_item_key()` L58-107), frozen jobs/news.
* Envelope top-level keys `PUBLIC_TOP_LEVEL_KEYS` (L190) unchanged; claim keyset unchanged.
* A refresh never modifies current claims (no carrying forward of stale values into `claims[]`): the previous value is only visible inside `changes[]`.

**Edge cases.**
* Same snapshot rerun: identical values and availability -> zero changes; changed evidence IDs or `retrieved_at` do not matter.
* `810034882`: previous (Oct-6 fat, derived claims) `employees=11`, current `10` -> one `updated` record with `previous_value 11`, `current_value 10`, `current_evidence_ids ["ev-810034882-0"]`, `previous_evidence` = the derived Oct-6 registry evidence.
* Previous `employees=11` (available), current registry field missing: after Phase 2 the current claim is `not_available` **with** registry evidence -> reported as `updated` availability change (a confirmed registry change, correct).
* Current `registry_live` failed: all registry claims `failed` -> no changes for them (old values are not declared gone).
* OCR or crawl stage on in the previous run, off now: placeholders -> no changes.
* New claim field introduced in a later schema (`CLAIM_FIELD_ORDER` change): present only on one side and that side is a placeholder -> skipped; present with real data -> `added`, correct.
* Previous file contains an org not in the current input: ignored. Current org not in previous file: no changes.
* Two roles for the same person inside `roles.roles`: the nested list is compared as a whole sorted structure via `_comparison_value()`, so no key collision affects correctness (blob splitting is out of scope).
* Previous file with merge-conflict markers or truncated lines: those lines are skipped with a warning (P0-1 cleanup is separate; the loader must simply not crash).

**Dependencies.** Phase 1 (official availability vocabulary in modules; `failed`/`blocked` semantics), Phase 2 (no `available`+`null`, so availability transitions are meaningful).

**Definition of done (static).** `--previous-envelopes` exists in `run.py`, `run_agent.py` (and is fingerprinted), and `build_output_contract.py`; `build_envelope()` passes `previous_envelope`; `diff_claims()` implements the skip rules and evidence references; validator check 17 exists; no change to envelope top-level keys.

---

### Phase 5: Final static code-review checklist for the approved P0 scope

See section 9. Phase 5 is a review step only: no code changes, no execution.

---

## 7. File-by-file change map

| File | Function / section | Phase | Change | Why | Done when (inspection) |
|---|---|---|---|---|---|
| `src/norway_company_agent/vocabulary.py` | new `PUBLIC_MODULE_STATES`, `WEBSITE_IDENTITY_MODULES`, `_identity_assessment`, `public_module_state`, `public_module_states`, `derive_public_terminal_status`, `invalid_public_module_states` | 1 | Add public mapping layer; keep internal functions unchanged | I-1, I-3, I-6 | Functions exist; internal ones untouched |
| `src/norway_company_agent/output_contract.py` | `to_public_envelope` L323, L350; `conversion_failure_envelope` L388-391 | 1 | Use public mapping / terminal derivation | I-1, I-3, I-4 | No public module state outside the six |
| `src/norway_company_agent/official.py` | `_classified` L209-214 | 1 | 200 + decode error -> `source_error` | I-5 | Branch present before the 200 branch |
| `src/norway_company_agent/website.py` | `fetch_website` HTTPError L645 | 1 | 401/403/451 -> `blocked` | HTTP classification | Expression updated |
| `scripts/validate_envelopes.py` | `check_enums` L202, `check_terminal_status` L255-257 | 1 | Validate public enum / public derivation | I-2 | Imports public helpers |
| `src/norway_company_agent/output_contract.py` | new `_is_missing_value`; `_claim` L496-511; registry loop L247-251; optional `_is_empty_checked` L434-439 | 2 | Null guard, per-field availability | I-7, I-8 | Guard present; per-field availability |
| `src/norway_company_agent/output_contract.py` | new `_public_source_class`; `add_evidence` L220-222 | 2 | Host-based classification | I-10 | `add_evidence` calls helper |
| `src/norway_company_agent/website.py` | `fetch_website` L463 | 2 | `official_registry_live` + org-specific BRREG URL | I-9 | No `registry_linked_company_website` with BRREG URL |
| `scripts/validate_envelopes.py` | `CHECK_NAMES`, `validate`, new checks 15, 16 | 2 | Detect regressions in reports | I-7..I-10 | Checks registered |
| `archive/v2-crawler/run_annual_report_workforce_connector.py` | new `detect_ocr_capability`; `collect` L185-198; `main` L253-299 | 3 | One capability check, `ocr_unavailable` status, report field | I-11 | No per-company `which` block; report has `ocr_capability` |
| `scripts/run_agent.py` | stage 5b gate L411; stderr notice | 3 | Gate prior-year on reported capability | I-12 | Gate reads `workforce-report.json` |
| `README.md` | L39-43 | 3 | Document opt-in OCR, system tools, evaluator command | I-13 | Section non-empty |
| `scripts/build_output_contract.py` | `main`, `build_envelope`, new `load_previous_envelopes`, report keys | 4 | Load and pass previous envelopes | I-14, I-17 | `previous_envelope` passed for public format |
| `scripts/run_agent.py` | argparse L201-211, `convert_cmd` L426-437, `stage_fingerprint` L121 | 4 | Forward + fingerprint `--previous-envelopes` | I-14 | Flag forwarded and hashed |
| `run.py` | argparse L263-283, fresh handling L353-360, commands L313-322 and L389-404 | 4 | Accept, preserve across `--fresh`, forward | I-14 | Flag forwarded in both branches |
| `src/norway_company_agent/changes.py` | `diff_claims` L184-293, `_diff_array` L296-337, `_change_record` L340-363, new helpers | 4 | Skip rules, evidence references | I-15, I-16 | Rules and keys present |
| `src/norway_company_agent/output_contract.py` | `to_public_envelope` L335-339 | 4 | Pass previous evidence and run id into `diff_claims` | I-16 | Keyword args passed |
| `scripts/validate_envelopes.py` | new check 17 | 4 | Changes well-formed | I-16 | Check registered |

Files that must **not** be modified in this P0 scope: `src/norway_company_agent/evidence.py` (internal enum), `src/norway_company_agent/batch.py`, `src/norway_company_agent/identity.py`, `scripts/run_competition_batch.py`, discovery / crawl scripts under `archive/v2-crawler/` other than the OCR connector, `requirements.txt` (except an optional comment), `tests/`.

---

## 8. Data-contract and compatibility considerations

* **Public envelope keys** (`PUBLIC_TOP_LEVEL_KEYS`, `PUBLIC_RUN_KEYS`, `PUBLIC_EVIDENCE_KEYS`, claim keyset) are unchanged by all phases. The only additive schema change is three keys inside each `changes[]` record (Phase 4, Q4).
* **`modules[*]` shape** stays `{state, retry_count[, reason]}`; only the allowed `state` values change.
* **Terminal status semantics** change in two documented cases: (1) a website that is reachable but identity `REVIEW` now makes the record `completed_with_gaps` instead of `completed`; (2) a required module with no record at all becomes `failed` instead of `completed_with_gaps`. Everything else derives the same terminal status as today.
* **Fat format** (`--envelope-format fat`, `registry-envelopes.jsonl`) keeps internal states. It is not the submission artifact. Do not "fix" it in this scope.
* **Evidence IDs** stay positional (`ev-<org>-<n>`, validator check 07). Stable cross-run IDs are a P2 item; Phase 4 works around it by copying previous evidence into the change record.
* **Existing tests** (`tests/test_phase3_envelope.py` L109-110, L122-152, L205; parts of `tests/test_phase6_changes.py`) assert internal states or old diff behaviour on public envelopes and will become stale. Per instructions they are neither run nor edited in this scope (see section 10).
* **Validator report** gains checks 15, 16, 17; `validation-report.json` consumers that iterate `checks` keep working.

---

## 9. Static code-review completion checklist

Static inspection only. Do not run tests, the pipeline, or any script.

**Phase 1**
- [ ] `vocabulary.py`: internal `MODULE_STATES`, `module_state`, `module_states`, `derive_terminal_status`, `invalid_module_states`, `batch_state` are byte-for-byte unchanged in behaviour.
- [ ] `public_module_state` covers every internal state and raises on unknown ones.
- [ ] REJECT detection reads `value.identity_assessment.decision`, REVIEW detection reads `publishable is False`.
- [ ] `to_public_envelope` and `conversion_failure_envelope` never assign `modules` from `module_states()` directly.
- [ ] `validate_envelopes.py` uses public helpers for checks 08 and 12.
- [ ] `_classified` treats 200 + `error` as `source_error`.
- [ ] `website.py` HTTPError maps 401/403/451 to `blocked`.

**Phase 2**
- [ ] `_claim()` guard downgrades `available` + missing value to `not_available`, `value None`, `confidence None`, keeps `evidence_ids`.
- [ ] `False` / `0` are not treated as missing.
- [ ] No new keys on claims.
- [ ] `add_evidence` classifies by host first; BRREG hosts never yield `company_owned`.
- [ ] `website.py` no-website record uses `official_registry_live` and an org-specific URL.
- [ ] Checks 15 and 16 registered in `CHECK_NAMES` and `validate()` in the same order.

**Phase 3**
- [ ] OCR default remains off in `run.py` and `run_agent.py`.
- [ ] Capability detected once per connector run, including `nor` language data, all subprocess calls time-limited and exception-safe.
- [ ] Missing capability yields `ocr_unavailable`, never `error`; text PDFs still processed.
- [ ] Prior-year stage gated on `workforce-report.json` `ocr_capability.available`.
- [ ] README states the evaluator command and the optional OCR prerequisites.

**Phase 4**
- [ ] `--previous-envelopes` accepted by `run.py`, forwarded by `run_agent.py`, fingerprinted in `stage_fingerprint`, consumed by `build_output_contract.py`.
- [ ] `--fresh` cannot delete the previous file before it is read.
- [ ] Loader tolerates bad lines, legacy fat rows, duplicates, missing orgs.
- [ ] `diff_claims` skips placeholders, `failed`/`blocked`, and website-dependent contact fields when the website side is uncomparable.
- [ ] Change records contain `current_evidence_ids`, `previous_evidence`, `previous_run_id`; all `current_evidence_ids` resolve in the same envelope.
- [ ] No previous file -> `changes: []`.
- [ ] Check 17 registered.

**Global**
- [ ] No files outside section 7 were modified.
- [ ] No new third-party dependency.
- [ ] No test file created or edited; nothing executed.

---

## 10. Explicit exclusions and deferred work

Excluded by instruction: P0-1 output hygiene and all merge-conflict cleanup (including the markers in `run.bat`, `docs/CODEBASE_MAP.md`, `docs/Daily_Memory_24_September.md` and `out/latest-run/`); scaling to more companies; batch performance; website discovery / recall; social profiles; jobs; news; splitting financial, role or workplace blob claims; generic `claim_span` quality; P1 and P2 items; UI; refactors.

Deferred items observed during inspection (record only, do not implement now):
* `operations` per envelope is always zero because `build_output_contract.py` never passes `metrics` to `to_public_envelope()` (P2).
* `registry` module is permanently `not_applicable` (input hint record); valid enum, semantics questionable (P2).
* `search_discovered_company_website` is not mapped to `company_owned` (classification consistency, P2).
* Stable cross-run evidence IDs (P2).
* Existing tests listed in section 8 need updating when testing is authorized.
* **Security, outside P0 scope but urgent:** the uploaded zip contains a populated `.env` (Exa, Tavily, Brave keys). It is git-ignored but was shipped in the archive. The keys should be rotated by the owner. Do not print or copy them.

---

## 11. Instructions for the future AI coding agent

1. Implement **only** the phase the user names ("Implement Phase N only"), then stop and report.
2. Before editing, re-read the target functions in the current files; line numbers here are hints. If a symbol described here does not exist or behaves differently, stop and report instead of improvising.
3. Do not run tests, `pytest`, `unittest`, smoke tests, benchmarks, the pipeline, `run.py`, `run_agent.py`, any connector, or any network/API call. Do not create or edit test files. This holds until the user explicitly authorizes execution.
4. Do not modify files not listed for that phase in section 7. Do not add dependencies. Do not rename existing functions or change existing signatures except where this plan adds optional keyword arguments.
5. Do not touch P0-1 artifacts (conflict markers, `out/`, `result/`), even if encountered.
6. Keep internal statuses (`evidence.EvidenceStatus`, `vocabulary.MODULE_STATES`) intact; public mapping happens only in the public-envelope and validator layers.
7. Do not add keys to claims, evidence, `run` or top-level envelope. The only permitted additive keys are the three change-record keys in Phase 4.
8. After finishing a phase, report: files changed (with function names), what was changed, any deviation from this plan and why, and the section 9 checklist items for that phase marked as statically reviewed. Never state that the code was tested or verified by execution.
9. Never print, copy or commit values from `.env`.

---

## 12. Unresolved questions and assumptions

| # | Question | Assumption used in this plan |
|---|---|---|
| Q1 | Does the official six-state contract apply to `modules[*].state` as well as claim availability? The public harness doc lists the states but no field-level schema. | Yes; modules are mapped. |
| Q2 | What is the honest public state for a non-required module that was not run in this configuration (`not_fetched`, e.g. no API key)? | `not_available`, except discovery modules when a website is already available (`not_applicable`) and required modules (`failed`). Revisit if the contract defines a "not attempted" convention. |
| Q3 | Should a reachable but unverified website (identity `REVIEW`) count as a gap? | Yes (`ambiguous` module, `completed_with_gaps`). |
| Q4 | Does the official contract define the change-record schema ("refresh metadata")? | Unknown. Plan keeps existing keys and adds `current_evidence_ids`, `previous_evidence`, `previous_run_id`. If the contract specifies names, rename only these three. |
| Q5 | How does the official evaluator trigger a refresh: does it pass a previous envelope file, a directory, or rerun with prior state on disk? | Explicit `--previous-envelopes` flag; no implicit default. Needs confirmation from the official evaluation contract. |
| Q6 | Should BRREG HTTP 403/429 be `blocked` instead of `failed`? | `failed` (retried, visible). Website 401/403/451 are `blocked`. |
| Q7 | Is treating `""` and `{}` as missing acceptable for all registry fields? | Yes; `False`/`0` remain real values. |
| Q8 | Should empty `financials.records` / `financial_history.years` be `not_available` (Phase 2 step 5)? | Recommended; marked optional. |
| Q9 | Is the fat envelope ever submitted? | No; `result/envelopes.jsonl` is the public format (`build_output_contract.py` default `public`). |
| Q10 | Previous files from the Oct-6 run are legacy fat rows without `claims`; deriving claims from their `profile` uses today's conversion code, not the code of that day. | Acceptable for change detection on registry scalars; noted as a limitation. |
