# Signalpost: Sunil vs Abhinav vs Shailesh, detailed artifact comparison + Sunil fix plan

**Date:** 14 Oct 2026 · **OUR_REPO = Sunil** · **COMPETITOR_1 = Abhinav (abhinavlenka)** · **COMPETITOR_2 = Shailesh**
**Basis:** sirf zip ke output artifacts (code abhi nahi mila). Har claim neeche file path ya computed number se backed hai. Code milne ke baad module-level fixes confirm honge.

---

## 0. Seedhi baat (verdict)

1. **Sunil ka submission abhi submit karne layak nahi hai.** Latest run sirf **9 companies** pe hai, `out/latest-run/` ke **13 files mein git merge-conflict markers** (`<<<<<<< HEAD`) pade hain, module states official enum ke bahar hain (`not_found`), 7 companies mein `employees` ko `available` bola hai jabki value `null` hai, aur OCR wala poora stage machine pe tesseract na hone se mar gaya. Ye P0 hain, recall se pehle.
2. **Abhinav sabse strong artifact set hai:** 1000/1000 envelopes, 0 FAIL mere self-check mein, per-fact claims, per-field evidence span, website 22.5% companies pe, social 11.3%, refresh reruns idempotent. Board pe 58.31 (Recall 9.32 / Evidence 28.99 / Synth 12 / UX 8). Matlab uska format official evaluator ne accept kiya hai. **Uska weak point recall hai (9.32/50)**, jo board ke 13 se 17 wale log se bhi kam hai.
3. **Shailesh ke zip mein envelopes hi nahi hain**, sirf reports. Uska agent basically registry-only hai: 1000 companies mein sirf 39 external observations (max 3.9%). Envelope contract NOT PROVEN.
4. **Sunil ke paas kuch asli advantages hain jo Abhinav ke paas nahi:** live registry (fresh), **3 saal ke financial records**, **group/konsern structure**, accounting obligation, request-log with failure_class, 14-check validation report, stage checkpoints. Lekin ye sab ek-ek **blob claim** mein ghusaye hue hain, isliye fact-count (recall ka 30% hissa) mein waste ho raha hai.
5. Teeno ka refresh proof **same evaluator fixture** hai (Shailesh `refresh-demo.json` aur Abhinav `refresh-replay.json` byte-for-byte same content, same hashes). Real data pe change detection kisi ne prove nahi kiya. Sunil ke real data mein change tha (employees 11 to 10) aur `changes[]` khali hai.
6. Teeno same starter kit ke fork lag rahe hain (same module names, same `accounting_obligation` ruleset URL aur sha `b94e22c6...`, same fixture). Isliye jeet ka farak sirf usme hai jo tumne base ke upar add kiya. Abhi Sunil ka add kiya hua (Tavily, Exa, deep crawl, OCR, news) **latest run mein zero facts de raha hai.**

---

## 1. Zip mein kya hai (inventory)

| Item | Shailesh | Abhinav | Sunil |
|---|---|---|---|
| Envelopes file | ❌ nahi | ✅ `entry-1000-envelopes.jsonl` (1000) | ✅ `run-20261014-010747-274033/envelopes.jsonl` (9), `out/envelopes.jsonl` same file |
| Profiles | ❌ | ✅ 1000 | ✅ 9 |
| Run report / summary | ✅ 4 run reports (10, 100, 100 sim, 1000) | ✅ `entry-1000-report.json` | ✅ `run-summary.json`, `run-metrics.json`, `registry-report.json`, `contract-report.json`, `validation-report.json` |
| Request log | ❌ | ❌ | ✅ 78 rows (lekin URL field nahi hai) |
| Snapshots | ❌ | ❌ | ✅ sirf purane Oct-6 run mein (16 html) |
| Refresh artifacts | fixture only | ✅ previous-profiles + 2 refresh runs (100 each) + fixture | ❌ kuch nahi |
| 100-company smoke | ✅ live100 + evalsim100 | ✅ refresh runs 100 (alag list) | ❌ (sirf 9 / 20) |
| Input list | ❌ | ✅ `entry-companies.jsonl` 1000 | 9 orgs (`all-orgs.txt`), **0 overlap** with Abhinav ke 1000 |
| Corruption | koi nahi | koi nahi | ❌ `latest-run/` mein 13 files conflict markers ke saath, do runs (Oct-6, Oct-9) mixed |

**Comparability:** teeno ke input batches alag hain. Isliye "kaun recall mein jeeta" ye raw counts se nahi bola ja sakta. Neeche coverage % **har repo ke apne batch pe** normalized hai. Sunil ke 9 companies sab `810xxxxxx`, sab bina registry website ke: biased, chhota sample.

---

## 2. Official requirement compliance matrix

Legend: PASS / PARTIAL / FAIL / NOT PROVEN

| # | Requirement (official) | Shailesh | Abhinav | Sunil | Sunil evidence |
|---|---|---|---|---|---|
| R1 | Exactly one terminal envelope per input | NOT PROVEN (envelopes nahi) | **PASS** 1000/1000, unique | PASS (9/9) | `validation-report.json` check 13 |
| R2 | 100-company smoke test result | PASS (report) | PASS (100 refresh run) | **FAIL** | max run 20 (Oct-6), latest 9 |
| R3 | Allowed states only: `available, not_available, blocked, not_applicable, ambiguous, failed` | NOT PROVEN | **PASS** | **FAIL** | `group`, `website`, `website_discovery` = `not_found` (26 module-states) |
| R4 | Missing != zero / available must have value | NOT PROVEN | PASS (empty registry field = `not_available` + note) | **FAIL** | 7/9 `employees` `available` with `value: null` |
| R5 | Source + retrieval time per claim | NOT PROVEN | PASS | PASS | har available claim ke paas evidence |
| R6 | Reporting period on financial claims | NOT PROVEN | **PASS** (`reporting_period` har financial claim pe) | PARTIAL | period sirf blob ke andar `records[].period` mein |
| R7 | Evidence actually supports the claim (span) | NOT PROVEN | **PASS** (`"antallAnsatte: (empty)"`, `"Styrets leder: X"`) | **FAIL** | har claim ka span same: `"SANDNES ELEKTRISKE AS, organisation number 810034882"`. `employees=10` ka proof span mein nahi |
| R8 | Source class honest | NOT PROVEN | PARTIAL (BRREG CSV URL `company_owned_website` label, sirf not_available social claims pe) | **FAIL** | BRREG URL `source_class: company_owned` (9/9, website claims) |
| R9 | Refresh: changes with old/new evidence | fixture only | PARTIAL (idempotent rerun proven, real change nahi) | **FAIL** | employees 11 to 10 (810034882), 7 to 5 (810182482), `changes: []` |
| R10 | Idempotent rerun, no duplicates | fixture only | PASS (claims 100/100 same). Evidence IDs 0/100 stable | NOT PROVEN | koi rerun artifact nahi |
| R11 | Reproducible clean run, declared deps | PARTIAL ($0, brreg only) | PARTIAL ($0, no external API) | **FAIL** | OCR ko `pdftoppm`, `tesseract` chahiye, undeclared; 7/9 error. Oct-6 machine pe chala, Oct-14 machine pe nahi |
| R12 | Batch scale within budget | PASS 1000 in 8m49s | PASS 1000 in 39m53s | **NOT PROVEN / risky** | 9 companies 4m13s = ~28 s/company; registry_live call ~12 s each |
| R13 | Declared APIs + cost | PASS ($0) | PASS ($0) | PARTIAL | Exa $0.063 logged, Tavily cost log nahi; dono ne 0 websites diye |
| R14 | No leaked secrets / personal paths | PASS | PASS | PARTIAL | koi API key nahi mili; lekin `C:\Users\sunil.sonu\...` aur `C:\Users\sunil\Desktop\...` 14 files mein |
| R15 | Artifacts internally consistent | PASS | PASS | **FAIL** | `latest-run/` mixed runs + conflict markers; `--include-deep-crawl` command mein tha, `stages_run` mein deep_crawl nahi |

---

## 3. Coverage per information family (har repo apne batch pe, % companies jahan `available`)

| Family | Shailesh (100 live, completeness report) | Abhinav (1000) | Sunil (9, Oct-14) | Sunil Oct-6 (20, stale) |
|---|---|---|---|---|
| Legal identity (name, form, industry, address, bankrupt, liquidation) | 100% | 99.9% | 100% | 100% |
| Industry description (text) | ? | 99.9% (`industry`) | **0%** (profile mein `industry_label` hai, claim nahi banaya) | - |
| Employees (registry) | ? | 14.4% honest | 2/9 real, 7/9 fake-available | - |
| Annual accounts latest | 100% | 99.7% (7 alag claims/company) | 100% (1 blob) | 100% |
| **Multi-year financial values** | ? | ❌ sirf latest year | ✅ **3 years** (2023 to 2025) blob mein | 15 prior-year OCR obs |
| Filed years / PDF list | ? | 100% | 100% (blob) | 100% |
| Roles / leadership | 100% | 99.9% (**3962 role claims**) | 100% (1 blob, 2 to 21 roles andar) | 100% |
| Registered workplaces | 100% | 74.5% (**872 claims**) | 100% (1 blob) | 100% |
| **Group / konsern** | ❌ (module hai, output nahi dikha) | ❌ nahi | ✅ 1/9 (8 = genuine 404) | - |
| Accounting obligation | ? | ❌ claim nahi | ✅ 100% | ✅ |
| Official website (verified) | 6% | **22.5%** (225) + 13.8% ambiguous | **0%** | 1/20 (aur woh contact page URL tha) |
| Social profiles | 0% | **11.3%** (259 profiles) | 0% | 0% |
| Jobs / hiring | 0% | 0% | 0% | 0% |
| News / dated activity | 1% | 0% | 0% | 0% (20 articles normalize hue, 0 output) |
| Workforce from annual report | ? | ❌ | 0% (OCR crash) | 15/20 |
| Refresh changes (real data) | 0 | 0 | 0 | - |

**Fact granularity (recall ke 30% claim-coverage hisse ke liye critical):**

| Metric | Abhinav | Sunil |
|---|---|---|
| Claims per company (avg) | 24.2 (min 13, max 110) | 25 (fixed) |
| Unme `available` | ~20.2 | ~14.1 |
| Roles kaise | 1 claim per person-role | 1 claim, andar list |
| Financials kaise | 7 claims (revenue, operating_result, profit_before_tax, annual_result, assets, equity, debt) + period + currency | 1 claim, 3 years ka list |
| Workplaces kaise | 1 claim per subunit | 1 claim, andar list |
| Not-available placeholders | evidence ke saath ("antallAnsatte: (empty)") | 5 claims (jobs, news, activity, workforce, prior_year) **bina evidence**, plus 5 website-family claims ek galat-labelled evidence pe |

Example 810034882: Sunil ke paas 6 roles + 3 locations + 3 years x 7 metrics = **30 individual facts** already data mein hain, lekin envelope mein woh **3 claims** dikhte hain. Agar evaluator fact-by-fact match karta hai (format proven accepted Abhinav ka hai), to ye seedha recall ka nuksaan hai. Ye exact evaluator behavior NOT PROVEN hai, lekin risk lene ka koi reason nahi.

---

## 4. Evidence / precision quality

| Check | Abhinav | Sunil |
|---|---|---|
| Evidence fields | `id, source_url, source_class, retrieved_at, content_sha256, claim_span` (+ `source_row_key`, `extraction_method`, `proof`) | `id, source_url, source_class, retrieved_at, content_sha256, claim_span` |
| Span specific to value | ✅ `"naeringskode1.kode: 00.000"` | ❌ generic name + orgnr har claim pe |
| Evidence per field | alag IDs (registry fields ke liye ~9 per company) | ek evidence ID poore registry ke liye |
| Website identity proof | `identity_proofs`: `organisation_number`, `legal_name_on_site`, `registry_phone`, `registry_email`, `unique_legal_name_domain` | Oct-14 mein koi website nahi; Oct-6 mein gate tha (ACCEPT/REVIEW/REJECT + quarantine) |
| Search results as evidence | `website_candidate_search` sirf not_available / ambiguous pe use hua, published claims pe nahi ✅ | Tavily/Exa raw results persist nahi kiye ✅ |
| Freshness | registry bulk CSV 15 Sep, roles bulk 13 Sep (2 din purana) | live API, same-run retrieval ✅ (Sunil yahan better) |
| Missing sha | 1670 (sab not_available / website-candidate) | 17 (9 website + 8 group 404) |

**Abhinav ke precision risks (review karna, copy mat karna blindly):** 150/225 websites `name_guess` se aaye. Sample mein `HTE HOLDING AS -> hte.com`, `HORST HOLDING AS -> https://www.horst.com/password` (Shopify password page), `WYSSEN NORGE AS -> wyssen.com` (parent group site), `GÅSEID UTVIKLING AS -> planor.no` (email domain se). Holding company ko brand site se jodna parent/subsidiary confusion hai. Board pe uska evidence 28.99/30 hai, to zyada saza nahi mili, lekin Sunil copy kare to **holding-company guard** lagaye (section 7, P1-3).

---

## 5. Operations, reproducibility, cost

| | Shailesh | Abhinav | Sunil |
|---|---|---|---|
| 1000-company runtime | 8m49s | 39m53s | not run; current rate 253 s / 9 cos = ~28 s per company, i.e. ~7.8 h for 1000 agar yahi rate raha |
| Requests | 5610 (sab BRREG) | 5745 (official 2002, robots 1900, site 1843) | 78 for 9 cos (BRREG 54, Exa 9, Tavily 9, site 6) |
| Per-company budget | limits in report | max 14 req/company (39 `failed` = "company allowance exhausted") | not stated |
| 3rd-party cost | $0 | $0 | Exa $0.063 / 9 cos, Tavily not logged; **output gain 0** |
| Registry strategy | live API | **bulk CSV (1.17M rows scanned) + bulk roles/subunits snapshot** | live API per company (~12 s/call) |
| Per-envelope `operations` | ? | real (`requests: 8, runtime_ms: 17973`) | **all zero** (9/9) |
| Validation | 5 checks | report `validation.passed` | 14 checks (best design, lekin bugs pakad nahi paaye) |

Sunil ka `validation-report.json` `passed: true` bolta hai jabki `not_found` enum, null-available, aur mislabelled source class sab andar hain. **Validator khud weak hai**: check 08 `enum_values_allowed` apne internal enum se check kar raha hai, official 6-state enum se nahi.

---

## 6. Sunil ke specific bugs (proof ke saath)

| ID | Severity | Bug | Proof (file) | Fix |
|---|---|---|---|---|
| B1 | **P0** | `out/latest-run/` mein merge-conflict markers, 2 runs mixed | `latest-run/envelopes.jsonl` 32 lines = 9 (Oct-9) + 20 (Oct-6) + markers; 13 files affected | `out/` ko git se hatao (`.gitignore`), `latest-run` har run pe fresh likho ya symlink/pointer file rakho (`.latest-run-id` already hai) |
| B2 | **P0** | Module state `not_found` | `modules.group/website/website_discovery.state` | mapping: `not_found -> not_available`, HTTP error -> `failed`, 403/429 -> `blocked` |
| B3 | **P0** | `employees` `available` with `null` | 7/9 envelopes | null ho to `availability: not_available`, `note: "antallAnsatte empty in registry"`, evidence span `"antallAnsatte: (empty)"` |
| B4 | **P0** | BRREG URL labelled `company_owned` | `ev-<org>-1` har envelope | website-absent evidence ko `official_registry_live` class do |
| B5 | **P0** | OCR undeclared system dependency | `workforce-report.json`: `"OCR required but missing executable(s): pdftoppm, tesseract"` 7/9 | ya to Dockerfile / README mein `poppler-utils tesseract-ocr tesseract-ocr-nor` install declare karo, ya evaluator command se OCR hata do. Missing binary pe state `not_available` + reason, `error` nahi |
| B6 | **P0** | 100 / 1000 run nahi | max 20 | official entry list pe full run + 100 smoke |
| B7 | **P0** | Refresh change detect nahi | 810034882 employees 11 (Oct-6) -> 10 (Oct-14), 810182482 7 -> 5, dono `changes: []` | `--previous <envelopes>` input, field-wise diff, `changes[]` mein `field, old_value, new_value, old_evidence_id, new_evidence_id, detected_at` |
| B8 | P1 | Blob claims | `financials`, `financial_history`, `roles`, `locations`, `group` | section 7 P1-1 |
| B9 | P1 | Generic claim_span | 6/9 envelopes mein 3+ scalar claims ki value span mein nahi | span = `"<brreg_field>: <value>"` jaise Abhinav |
| B10 | P1 | `industry_label` claim nahi bana | profile mein hai, envelope mein nahi | free fact, add karo |
| B11 | P1 | Paid search 0 yield | Exa 9/9 abstain, Tavily 8 abstain + 1 quarantined; Oct-6 Tavily 1/18 verified, Exa 0/17 | deterministic candidates pehle (section 7 P1-3) |
| B12 | P1 | News pipeline drop | Oct-6 `crawl-report.json` `total_normalized_news: 20`, `news-report.json` `observations: 0` | normalized -> observation -> claim mapping fix |
| B13 | P1 | Deep crawl silently skip | command `--include-deep-crawl`, `stages_run` mein nahi | stage hamesha log karo: `skipped: reason=no_verified_website` |
| B14 | P2 | `operations` all zero | 9/9 | request-log se per-org requests/runtime/cost sum karke likho |
| B15 | P2 | `registry` module `not_applicable` | 9/9; note kehta hai input metadata hint-only | module ko `input_metadata` rename karo ya `registry` = `available` (live result) |
| B16 | P2 | request-log mein URL nahi | `request-log.jsonl` keys | `requested_url`, `final_url`, `redirects` add (secrets strip karke) |
| B17 | P2 | Absolute Windows paths | 14 files | reports mein relative paths |
| B18 | P2 | Oct-6 website value contact page tha | `https://beckmann-norway.com/en/policies/contact-information` | official_website = canonical origin (`https://beckmann-norway.com/`), page URL sirf evidence mein |
| B19 | P2 | not_available claims bina evidence | jobs, news, activity, workforce, prior_year | attempt evidence do (kya check kiya, kab) ya module state se cover karo |

---

## 7. Sunil fix plan (baith ke karne wala checklist)

Order mat todna. P0 khatam kiye bina recall pe kaam karna bekaar hai, kyunki invalid run score hi nahi hota.

### P0: qualify hone layak banao

- [ ] **P0-1 Output hygiene.** `out/` ko `.gitignore` mein daalo, repo se `git rm -r --cached out`. `latest-run/` ko har run start pe delete karke recreate karo. Run folder hi source of truth.
  *Acceptance:* `grep -rl '^<<<<<<<' out/` khali.
- [ ] **P0-2 Official enum.** `claims_conversion` (aur jahan module states set hote hain) mein ek single mapper: `{complete|ok -> available, not_found|empty -> not_available, http_403|http_429|robots_forbidden -> blocked, exception -> failed}`. Apna `envelope_validation` check 08 ko official 6 values pe hardcode karo.
  *Acceptance:* self-check mein `not in official enum` 0.
- [ ] **P0-3 Null guard.** Claim builder mein rule: `value is None -> availability = not_available` (kabhi `available` nahi). Validator mein naya check `15_available_claims_have_value`.
- [ ] **P0-4 Source class fix.** BRREG ke kisi bhi URL ka class `official_*` hi ho. Validator check `16_source_class_matches_host`.
- [ ] **P0-5 OCR decision.** Do mein se ek: (a) `Dockerfile` / README mein apt packages + `pytesseract` version pin + startup pe `shutil.which` check, ya (b) official command se `--include-workforce-ocr` hatao. Missing binary pe module `not_available` with reason.
  *Acceptance:* clean container mein `workforce-report.json` mein `error` 0.
- [ ] **P0-6 Scale.** 12 s per registry call serial nahi chalega. Options: BRREG bulk `enheter/lastned` CSV + `underenheter/lastned` ek baar download (Abhinav ka approach, 1.17M rows), roles/regnskap live API concurrency 8 to 16 with backoff. Fir **official 1000 entry list** (Abhinav ke zip mein `entry-companies.jsonl`, 1000 rows; tumhari file sirf 9 rows ki hai) aur ek 100 smoke run.
  *Acceptance:* 1000/1000 envelopes, self-check 0 FAIL, runtime aur requests report mein.
- [ ] **P0-7 Refresh.** `run_agent.py --previous out/<old-run>/envelopes.jsonl`. Stable claim key = `org + field + sub_key` (role ke liye `role_code + name`, workplace ke liye subunit orgnr, financial ke liye `metric + period_end`). Diff -> `changes[]`. Fetch fail hua to purani value `last_known` rakho, change emit mat karo.
  *Acceptance:* (1) same snapshot do baar = 0 changes, 0 duplicates; (2) Oct-6 vs Oct-14 pe 810034882 `employees 11 -> 10` change emit ho; (3) `python signalpost_selfcheck.py --envelopes new.jsonl --previous old.jsonl` 0 FAIL.

### P1: recall badhao (asli score yahan hai: board ka top recall sirf 17.45/50)

- [ ] **P1-1 Claims explode karo** (sabse sasta, sabse bada gain):
  - `financials` -> per year per metric: `financials.revenue` etc. with `reporting_period {start, end}`, `currency`, `account_type`. 3 years x 7 metrics. Missing metric = `not_available`, kabhi 0 nahi. Abhinav sirf latest year deta hai, **ye tumhara edge hai.**
  - `roles` -> `role` claim per person, `effective_at = last_changed`, span `"Styrets leder: <name>"`.
  - `locations` -> `registered_workplace` per subunit.
  - `group` -> per relation claim (`parent` / `subsidiary`, orgnr, `grunnlag` %, `dato`). Abhinav ke paas group hai hi nahi.
  - `financial_history` -> `financials.filed_years` + documents list.
  - Free facts: `industry` (label), `registry_declared_website`, `municipality_number`, `accounting_obligation`.
- [ ] **P1-2 Field-specific evidence.** Har registry field ka apna evidence ID aur span `"<json_path>: <value>"`. Same `content_sha256` reuse kar sakte ho.
- [ ] **P1-3 Website discovery deterministic-first.** Order: registry `hjemmeside` -> registry email domain (gmail/hotmail etc. block list) -> name slug guesses (`AS/ASA/HOLDING/NORGE` strip, `.no` then `.com`) -> tab Tavily/Exa. Accept sirf jab proof ho: orgnr on site (strong), ya legal name on site + (registry phone ya email ya address) match. **Holding guard:** legal name mein `HOLDING` ho aur site pe orgnr nahi to `ambiguous`. `/password`, parked, for-sale pages reject. Abhinav ne $0 mein 22.5% liya, tumhare paid providers ne 0 diya.
- [ ] **P1-4 Social profiles** sirf verified site ke links se (`facebook, instagram, linkedin, youtube, x, tiktok`), relation `linked_from_verified_company_website`. Profile pages scrape mat karo (source policy).
- [ ] **P1-5 News fix (B12)** + har article ka `published_at`, URL, title claim. Undated = drop.
- [ ] **P1-6 Jobs (koi bhi competitor ke paas nahi = differentiator).** Verified site pe careers route + `JobPosting` JSON-LD + site ke andar linked ATS pages. Har job: title, location, `date_posted`, `valid_through`, URL. Expired ko current mat dikhao. Koi bhi third-party job board use karne se pehle `signalpost-sources.md` policy check karo; ye experiment hai, proven nahi.
- [ ] **P1-7 OCR wapas laao (agar P0-5a liya)**: workforce + prior-year. Lekin agar regnskap API already 3 years de raha hai to prior-year OCR redundant hai; sirf us case mein chalao jab API mein year missing ho.

### P2: precision aur provenance

- [ ] Stable evidence IDs (`sha256(org|source_url|field|content_sha256)[:16]`), retrieved_at se nahi. Abhinav ke refresh mein evidence IDs 0/100 stable the, tum yahan usse aage nikal sakte ho.
- [ ] request-log mein `requested_url`, `final_url`, redirect chain.
- [ ] Per-envelope `operations` real numbers.
- [ ] Website value = canonical origin; page URL evidence mein.
- [ ] Abhinav ke flagged cases jaise holding/brand ko apne test fixtures mein daalo (wrong-company tests).

### P3: synthesis / UX

Kisi bhi zip mein viewer ya summary text nahi hai, phir bhi board pe log 12/12 synthesis aur 8/8 UX le rahe hain. Matlab ye hissa shayad evaluator khud claims se render karta hai. NOT PROVEN. Pehle P0 to P2, fir official contract (`npm run check:signalpost*` validators) dekh ke decide karo.

---

## 8. Abhinav se kya copy karna, kya nahi

| Copy karo | Mat karo |
|---|---|
| Per-fact claims (role, workplace, financial metric) | `name_guess` websites bina holding guard |
| `reporting_period` + `currency` har financial claim pe | Sirf latest year financials (tum 3 years de sakte ho) |
| Empty registry field = `not_available` + note + evidence | BRREG URL ko `company_owned_website` label karna |
| Field-specific `claim_span` | Bulk roles snapshot jo run se 2 din purana ho (freshness note karo ya live lo) |
| `identity_proofs` list har website claim pe | Retrieval time se bane unstable evidence IDs |
| Bulk registry download for speed, per-company request cap | |
| `website_candidate_search` sirf negative/ambiguous decisions ke liye | |

## 9. Sunil ke strengths (inhe mat todna)

- Live registry, same-run retrieval: freshness Abhinav se better.
- 3-year financial records already fetch ho rahe hain.
- Group/konsern structure aur accounting obligation: Abhinav ke paas nahi.
- Identity gate with ACCEPT / REVIEW / REJECT + quarantine (Oct-6 `crawl-report.json`), paid search ne kuch publish nahi kiya jab proof nahi tha. Precision ke liye sahi behavior.
- Request-log with `failure_class`, stage `.done` checkpoints with input sha, 14-check validator framework. Bas validator ko official rules pe tight karo.

---

## 10. Self-check script kaise chalana

Saath mein `signalpost_selfcheck.py` diya hai (stdlib only). Maine isse dono outputs pe test kiya: **Abhinav 0 FAIL, Sunil 33 FAIL** (aur `--out-dir out` ke saath conflict-marker FAILs alag se).

```bash
python signalpost_selfcheck.py \
  --envelopes out/<run-id>/envelopes.jsonl \
  --input entry-companies.jsonl \
  --out-dir out \
  --previous out/<previous-run-id>/envelopes.jsonl   # refresh test, optional
```

Ye check karta hai: conflict markers, invalid JSON, duplicate / missing / extra orgs, official enum, available-with-null, evidence refs, source_url / retrieved_at, sha missing, value-not-in-span, BRREG-as-company_owned, blob claims, financial reporting_period, zero operations, registry `not_applicable`, aur `--previous` ke saath changed-value-but-empty-`changes[]`. Target: **0 FAIL, aur WARN mein blob / span wale khatam.**

---

## 11. Unknowns (repo aane ke baad close honge)

1. Sunil ka actual code: `scripts/run_agent.py`, `claims_conversion`, `envelope_validation`, discovery aur crawl stages. Fix locations abhi stage names se infer kiye hain.
2. Official envelope schema: Abhinav ka shape (modules flat strings, `legal_name` top-level) board pe assessed hai. Sunil ka shape (`schema_version`, modules `{state, retry_count}`) accepted hai ya nahi, NOT PROVEN. Safe raasta: official validator chala ke confirm karo, warna Abhinav wala shape follow karo.
3. Evaluator blob claims ko facts mein todta hai ya nahi: unknown, isliye explode karna hi safe hai.
4. Board pe "Abhinav" yahi abhinavlenka hai, ye assumption hai (naam match).
5. Sunil ke 9 companies ki websites genuinely exist karti hain ya nahi: maine 2 search kiye (Sandnes Elektriske AS, Alstray AS), koi company-owned site nahi mili, sirf registry / utdanning.no listings. Isliye 0% website yahan wrong-abstain prove nahi hota. Isi liye full 1000 run zaroori hai.
6. Official run ka time budget public page pe number mein nahi diya.

**Sources:** [Signalpost challenge page](https://builderr.ai/challenges/signalpost) (scoring, states, leaderboard snapshot 5 Oct 2026), [Signalpost evaluation harness doc](https://builderr.ai/docs/signalpost-evaluation-harness.md) (envelope contents, states), [BRREG lookup 810034882](https://virksomhet.brreg.no/en/oppslag/enheter/810034882), [utdanning.no 810094532](https://utdanning.no/finnlarebedrift/bedrift/810094532/).
