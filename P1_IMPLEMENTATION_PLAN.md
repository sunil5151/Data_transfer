# P1_IMPLEMENTATION_PLAN.md

Repository: `signalpost-company-agent-main`, branch `main`, HEAD `79c625e` (contains `7a88ff2 feat: implement Phase 1 and Phase 2 contract compliance`), uploaded 16 Oct 2026.
Scope: **P1-1 (one claim per fact, from official BRREG data only)** and **P1-2 (field-specific evidence spans)**.
Status of this document: **planning only. Nothing was modified or executed while writing it.**

Line numbers refer to the uploaded snapshot. Always re-locate code by function name before editing.

---

## 1. Objective and scope

### 1.1 Why this work

Recall is 50 of 100 points. Of those, 30% is individual-claim coverage (how many checked facts were found). Today the agent fetches many official facts but packs them into a few blob claims:

| Data already fetched from BRREG | Today in the envelope | After this plan |
|---|---|---|
| Up to 3 annual-account records x 7 metrics (`normalize_financials`) | 1 claim `financials` with a `records` list | Up to 21 claims `financials.<metric>`, each with `reporting_period` |
| All registered roles (`normalize_roles`), e.g. 21 roles for `810098252` | 1 claim `roles` with a list | 1 claim `role` per active role |
| All sub-units (`normalize_locations`) | 1 claim `locations` with a list | 1 claim `registered_workplace` per sub-unit |
| Group structure (`konsernstruktur`), e.g. 5 subsidiaries for `810094532` | 1 claim `group` with the raw BRREG body | 1 claim `group_relation` per related entity |
| `naeringskode1.beskrivelse`, `forretningsadresse.kommunenummer`, `postadresse`, `hjemmeside` (already in `normalize_entity`) | Not published as claims | 4 new registry claims |

Every registry claim also cites the same generic span today (`"<NAME>, organisation number <ORG>"`), which never contains the value. P1-2 gives each claim its own evidence item whose `claim_span` contains the source field and the value (for example `antallAnsatte: 10`).

The competitor whose envelope shape was assessed on the official board (Abhinav, Evidence 28.99/30) uses exactly this granularity (`role`, `registered_workplace`, `financials.revenue` with `reporting_period`, extra optional claim keys). That is the evidence that optional claim keys and repeated claim fields are accepted by the evaluator.

### 1.2 In scope

* P1-1: claim contract extension, member-claim builders, refresh compatibility, wiring into the envelope, four extra registry claims.
* P1-2: per-field evidence for registry claims, value-bearing spans for member claims and the two remaining module singletons.

### 1.3 Explicitly out of scope (decided by the user)

* Anything about websites: website discovery, domain guessing, Tavily or Exa, website contact or social claims, website news, website jobs, careers pages, deep crawl. **Do not touch `website.py`, `identity.py`, `discovery.py`, `crawl_events.py`, `page_signals.py`, `_website_claim()` or `_website_contact_claims()`.**
* OCR observations (`workforce`, `prior_year_financials`), stays as is.
* P0-1 output hygiene, scale or batch performance, operations metrics, UI, tests.

---

## 2. Current repository status (static inspection)

### 2.1 P0 work present in code

| P0 item | Where it is visible | Observed |
|---|---|---|
| P0-2 state mapping | `vocabulary.py` `public_module_state()`, `public_module_states()`, `derive_public_terminal_status()`, `invalid_public_module_states()`; used in `output_contract.to_public_envelope()` and `validate_envelopes.py` | Present |
| P0-3 null guard | `output_contract._is_missing_value()` (L207), `_claim()` guard (L539-542), per-field registry availability (L272-279), `_is_empty_checked()` financials/history (L469-472), validator check 15 | Present |
| P0-4 source class | `_public_source_class()` (L217), `website.py` no-website record uses `official_registry_live`, validator check 16 | Present |
| P0-5 OCR | `detect_ocr_capability()` in the connector, `ocr_unavailable` status, prior-year gate in `run_agent.py`, README section | Present |
| P0-7 refresh | `--previous-envelopes` in `run.py`, `run_agent.py` (fingerprinted), `build_output_contract.py` `load_previous_envelopes()`; `changes.py` skip rules and evidence references; validator check 17 | Present |

The user reports these as verified. This plan builds on them and must not undo any of them.

### 2.2 Still open (not part of this plan, listed so the coding agent does not trip over them)

* **P0-1 is still open.** Unresolved merge-conflict markers remain in `run.bat` (the Windows wrapper of the evaluator command), `docs/CODEBASE_MAP.md`, `docs/Daily_Memory_24_September.md` and 9 files under `out/latest-run/`. Do not edit them in this plan.
* No 100 or 1000 company run exists for the current code. The newest envelopes in the repo are still from `run-20261014-010747-274033` (9 companies, pre-P0 shape).
* `.env` with provider keys is still inside the uploaded archive. Do not print, copy or commit it.
* Existing tests in `tests/` already assert pre-P0 behaviour and will assert pre-P1 behaviour too. They are not run or edited here.

---

## 3. Relevant architecture and data flow

```
stage 1  run_competition_batch.py collect_profile()
           official.fetch_official_modules()  -> profile.evidence[module] = evidence record
             registry_live.value = normalize_entity(...)     keys: name, legal_form, employees, bankrupt,
                                                              liquidating, website, industry{kode,beskrivelse},
                                                              business_address{...}, postal_address{...},
                                                              latest_submitted_accounts
             financials.value    = {"records": [ {record_id, account_type, period{fraDato,tilDato},
                                                  currency, revenue, operating_result, profit_before_tax,
                                                  annual_result, assets, equity, debt}, ... up to 3 ]}
             financial_history.value = {"years": [...], "pdfs": [{"year","url"}]}
             roles.value         = {"roles": [ {name, organisation_number, role_code, role, group_code,
                                                group, last_changed, inactive} ]}
             group.value         = raw BRREG konsernstruktur body, observed shape:
                                   {organisasjonsnummer, navn, organisasjonsform,
                                    children: [ {organisasjonsnummer, navn, dato, grunnlag,
                                                 knytningsform{kode,beskrivelse}, nivaa,
                                                 organisasjonsform, parentNavn,
                                                 parentOrganisasjonsnummer}, ... ]}
             locations.value     = {"locations": [ {organisation_number, name, address{...},
                                                    industry{...}, employees} ]}
stage 6  build_output_contract.py -> output_contract.to_public_envelope(fat, previous_envelope=...)
           registry claim loop          L262-279   (REGISTRY_CLAIM_FIELDS L154-164)
           website claim + contacts     L281-305   (OUT OF SCOPE, do not touch)
           MODULE_CLAIM_FIELDS loop     L307-319   (blob claims built here)
           observation loop             L321-348   (OUT OF SCOPE)
           _complete_claims()           L395-407   (one claim per CLAIM_FIELD_ORDER slot)
           changes.diff_claims()        changes.py L210  (keyed by claim.field)
           assert_evidence_integrity(), assert_claim_schema_integrity()  L26, L56
stage 6b validate_envelopes.py check_claims() L112-127 (exact field list and exact 5-key claim keyset)
```

Hard constraints found in the current code that this plan must change in a controlled way:

1. `validate_envelopes.check_claims()` requires claim keys to be exactly `{field, value, availability, confidence, evidence_ids}` and the claim field list to equal `CLAIM_FIELD_ORDER` exactly. Repeated fields are impossible today.
2. `_complete_claims()` keeps only the **first** claim per field (`setdefault`).
3. `changes.diff_claims()` builds `{field: claim}` dicts. With repeated fields the last one silently wins, so refresh would be wrong. **Refresh compatibility must land before member claims are emitted** (phase order below).

---

## 4. Target claim contract (P1-1)

### 4.1 Optional claim keys

Required keys stay `field, value, availability, confidence, evidence_ids`. Three optional keys are added:

| Key | Type | When present |
|---|---|---|
| `claim_key` | non-empty string, unique within the envelope | Required on every member claim (section 4.2). Not used on singleton claims. |
| `reporting_period` | `{"start": str or null, "end": str}` | Financial member claims |
| `effective_at` | string date | Role claims (`last_changed`), group relations (`dato`) when known |

No other new key may be added to claims. Evidence keys (`PUBLIC_EVIDENCE_KEYS`), top-level keys, `run` keys and module shape stay unchanged. `SCHEMA_VERSION` stays `"1.0"` (see Q2).

### 4.2 Claim families

`CLAIM_FIELD_ORDER` keeps the same slot names. Four slots become **families**:

| Family slot | Member field(s) | `claim_key` format | Member `value` |
|---|---|---|---|
| `financials` | `financials.revenue`, `financials.operating_result`, `financials.profit_before_tax`, `financials.annual_result`, `financials.assets`, `financials.equity`, `financials.debt` | `financials.<metric>:<account_type>:<period_end>` | `{"amount": <number>, "currency": <str or null>}`, or `null` when the metric is missing (claim becomes `not_available`) |
| `roles` | `role` | `role:<role_code>:<organisation_number or name>` | `{name, organisation_number, role_code, role, group_code, group}` |
| `locations` | `registered_workplace` | `registered_workplace:<subunit organisation_number>` | `{organisation_number, name, address, industry, employees}` |
| `group` | `group_relation` | `group_relation:<parent organisation_number or "-">:<organisation_number>` | `{organisation_number, name, parent_organisation_number, parent_name, link_code, link_description, ownership_basis, level}` |

Family rule (validator and builder): for each family slot an envelope contains **either** one or more member claims and no claim whose field equals the slot name, **or** exactly one claim whose field equals the slot name (a placeholder: `not_available`, `failed`, `blocked`, `not_applicable`, or a legacy blob when the source shape is not recognised, see 6.2 group fallback) and no members.

Singleton slots (every other entry of `CLAIM_FIELD_ORDER`) keep exactly one claim each.

### 4.3 New registry singleton claims

Appended at the **end** of `REGISTRY_CLAIM_FIELDS` (so existing slot order is unchanged):

| Claim field | Normalized path in `registry_live.value` | BRREG source path (for spans) |
|---|---|---|
| `industry` | `("industry", "beskrivelse")` | `naeringskode1.beskrivelse` |
| `municipality_number` | `("business_address", "kommunenummer")` | `forretningsadresse.kommunenummer` |
| `postal_address` | `"postal_address"` | `postadresse` |
| `registry_declared_website` | `"website"` | `hjemmeside` |

All four are already produced by `official.normalize_entity()` (L192-206). **Do not change `official.py`.** Missing values become `not_available` automatically through the existing P0-3 logic.

---

## 5. Phase dependency order

| Phase | Content | Depends on | Output change |
|---|---|---|---|
| 1 | Contract infrastructure: new module with constants, optional claim keys in `_claim()`, family-aware `_complete_claims()`, schema assertion, validator `check_claims()` | none | None (no member claims exist yet) |
| 2 | Pure member-spec builders in the new module | 1 | None (not wired yet) |
| 3 | Refresh compatibility: family-aware `diff_claims()`, legacy blob explosion, schema-evolution skip rule, validator check 17 update | 1, 2 | None for current output; correct refresh once Phase 4 lands |
| 4 | P1-1 wiring: emit member claims from `to_public_envelope()`, add 4 registry claims | 1, 2, 3 | **Yes** |
| 5 | P1-2: per-field registry evidence and value-bearing spans | 4 | **Yes** (evidence only) |
| 6 | Static review checklist | 1-5 | None |

**Phase 4 must never be applied without Phase 3.** If an agent is told "Implement Phase 4 only" and Phase 3 is not present in the code (no `explode_legacy_claims` import in `changes.py`), it must stop and report.

---

## 6. Detailed implementation instructions

### Phase 1: Claim contract infrastructure

**Goal.** Allow optional claim keys and claim families everywhere claims are built and validated, without changing today's output.

**Step 1.1 Create `src/norway_company_agent/claim_builders.py`** with only constants for now. It must not import anything from the package (it will be imported by `output_contract.py`, `changes.py` and the validator; keeping it import-free prevents cycles).

```python
"""Claim families and member-claim builders for official BRREG data (P1-1).

Pure functions only. No imports from this package.
"""
from __future__ import annotations

from typing import Any

FINANCIAL_METRICS: tuple[str, ...] = (
    "revenue", "operating_result", "profit_before_tax", "annual_result",
    "assets", "equity", "debt",
)

CLAIM_FAMILIES: dict[str, tuple[str, ...]] = {
    "financials": tuple(f"financials.{metric}" for metric in FINANCIAL_METRICS),
    "roles": ("role",),
    "locations": ("registered_workplace",),
    "group": ("group_relation",),
}

MEMBER_TO_FAMILY: dict[str, str] = {
    member: family for family, members in CLAIM_FAMILIES.items() for member in members
}

OPTIONAL_CLAIM_KEYS: frozenset[str] = frozenset({"claim_key", "reporting_period", "effective_at"})
```

`FINANCIAL_METRICS` must match the keys written by `official.normalize_financials()` (L110-129): `revenue, operating_result, profit_before_tax, annual_result, assets, equity, debt`. Verify by reading that function; do not edit it.

**Step 1.2 `output_contract.py` imports.** Add:
```python
from .claim_builders import CLAIM_FAMILIES, MEMBER_TO_FAMILY, OPTIONAL_CLAIM_KEYS
```

**Step 1.3 `_claim()` (L530-549).** Add keyword-only parameters and append optional keys only when not `None`:

```python
def _claim(
    field: str,
    value: Any,
    availability: str,
    confidence: float | None,
    evidence_ids: list[str],
    *,
    claim_key: str | None = None,
    reporting_period: dict[str, Any] | None = None,
    effective_at: str | None = None,
) -> dict[str, Any]:
    if availability not in ALLOWED_AVAILABILITY:
        availability = "failed"
    if availability == "available" and _is_missing_value(value):
        availability = "not_available"
        value = None
        confidence = None
    claim = {
        "field": field,
        "value": value,
        "availability": availability,
        "confidence": confidence,
        "evidence_ids": evidence_ids,
    }
    if claim_key is not None:
        claim["claim_key"] = claim_key
    if reporting_period is not None:
        claim["reporting_period"] = reporting_period
    if effective_at is not None:
        claim["effective_at"] = effective_at
    return claim
```

Keep the existing P0-3 guard exactly as it is. Every existing call site keeps working because the new parameters are keyword-only with `None` defaults.

**Step 1.4 `_complete_claims()` (L395-407).** Replace the body with a family-aware version. Docstring: "Return one entry per CLAIM_FIELD_ORDER slot. A family slot is either its member claims (sorted by field, claim_key) or one placeholder claim."

```python
def _complete_claims(claims, *, missing_availability="not_available"):
    singles: dict[str, dict[str, Any]] = {}
    members: dict[str, list[dict[str, Any]]] = {family: [] for family in CLAIM_FAMILIES}
    unknown: list[dict[str, Any]] = []
    for claim in claims:
        field = claim["field"]
        if field in MEMBER_TO_FAMILY:
            members[MEMBER_TO_FAMILY[field]].append(claim)
        elif field in CLAIM_FIELD_ORDER:
            singles.setdefault(field, claim)
        else:
            unknown.append(claim)
    ordered: list[dict[str, Any]] = []
    for slot in CLAIM_FIELD_ORDER:
        family_members = members.get(slot) or []
        if family_members:
            seen_keys: set[str] = set()
            for claim in sorted(family_members, key=lambda c: (c["field"], str(c.get("claim_key") or ""))):
                key = str(claim.get("claim_key") or "")
                if key in seen_keys:
                    continue
                seen_keys.add(key)
                ordered.append(claim)
            continue  # a placeholder for this slot is dropped when members exist
        ordered.append(
            singles.get(slot)
            or _claim(slot, None, missing_availability, _confidence(missing_availability), [])
        )
    ordered.extend(unknown)
    return ordered
```

Behaviour with no members (today): identical to the current function.

**Step 1.5 `assert_claim_schema_integrity()` (L56-151).** Inside the per-claim loop, after rule 1 (missing required keys), add:
* rule 1b: `extra = set(claim) - CANONICAL_CLAIM_FIELDS - OPTIONAL_CLAIM_KEYS`; if non-empty, append a violation.
* rule 1c: if `claim.get("field") in MEMBER_TO_FAMILY` and `claim_key` is not a non-empty string, append a violation.
Update the docstring to list the optional keys. Do not add any other rule.

**Step 1.6 `scripts/validate_envelopes.py`.**
* Import `CLAIM_FAMILIES`, `MEMBER_TO_FAMILY`, `OPTIONAL_CLAIM_KEYS` from `norway_company_agent.claim_builders`.
* Replace `check_claims()` (L112-127) with:

```python
def check_claims(rows: list[dict[str, Any]]) -> list[str]:
    out: list[str] = []
    required = set(CANONICAL_CLAIM_FIELDS)
    allowed = required | set(OPTIONAL_CLAIM_KEYS)
    slots = list(CLAIM_FIELD_ORDER)
    known_fields = set(slots) | set(MEMBER_TO_FAMILY)
    for row in rows:
        org = _org(row)
        claims = _claims(row)
        key_counts: Counter = Counter()
        for claim in claims:
            keys = set(claim)
            field = claim.get("field")
            if not required <= keys or not keys <= allowed:
                out.append(f"{org}: claim {field!r} keys {sorted(keys)} not within required {sorted(required)} + optional {sorted(OPTIONAL_CLAIM_KEYS)}")
            if field in MEMBER_TO_FAMILY:
                claim_key = claim.get("claim_key")
                if not isinstance(claim_key, str) or not claim_key:
                    out.append(f"{org}: member claim {field!r} has no claim_key")
                else:
                    key_counts[claim_key] += 1
            if "reporting_period" in claim:
                period = claim["reporting_period"]
                if not isinstance(period, dict) or set(period) != {"start", "end"} or not period.get("end"):
                    out.append(f"{org}: claim {field!r} reporting_period {period!r} malformed")
            if "effective_at" in claim and not isinstance(claim["effective_at"], str):
                out.append(f"{org}: claim {field!r} effective_at must be a string")
        out.extend(f"{org}: claim_key {key!r} used {count} times" for key, count in key_counts.items() if count > 1)
        field_counts = Counter(claim.get("field") for claim in claims)
        for slot in slots:
            if slot in CLAIM_FAMILIES:
                member_count = sum(field_counts[member] for member in CLAIM_FAMILIES[slot])
                placeholder_count = field_counts[slot]
                if not ((member_count >= 1 and placeholder_count == 0) or (member_count == 0 and placeholder_count == 1)):
                    out.append(f"{org}: family {slot!r} has {member_count} member(s) and {placeholder_count} placeholder(s)")
            elif field_counts[slot] != 1:
                out.append(f"{org}: claim field {slot!r} appears {field_counts[slot]} times, expected 1")
        unknown = sorted(str(field) for field in field_counts if field not in known_fields)
        if unknown:
            out.append(f"{org}: unknown claim fields {unknown}")
    return out
```
* `CLAIM_KEYSET` (L48) becomes unused. Leave it in place (harmless) or delete it; do not reuse it.
* Order of claims is no longer checked by the validator; `_complete_claims()` still produces a deterministic order.

**Preserve.** P0-3 guard; `ALLOWED_AVAILABILITY`; all other checks; `CLAIM_FIELD_ORDER` content.

**Done when (static).** `claim_builders.py` exists with the four constants and no package imports; `_claim()` has the three keyword-only parameters; `_complete_claims()` handles families; schema assertion has rules 1b and 1c; `check_claims()` matches the code above.

---

### Phase 2: Member-spec builders (pure functions)

**Goal.** Convert each blob value into a list of **specs**. A spec is a plain dict: `{"field", "value", "claim_key", "span"}` plus optional `"reporting_period"` and `"effective_at"`. Specs are not claims: `output_contract.py` (Phase 4) turns them into claims and evidence. `changes.py` (Phase 3) uses the same functions so claim keys always match between runs.

Append to `claim_builders.py`:

```python
def _text(value: Any) -> str:
    return "" if value is None else str(value).strip()


def financial_member_specs(value: Any) -> list[dict[str, Any]]:
    """One spec per (record, metric). Missing metric -> value None (claim becomes not_available)."""
    specs: list[dict[str, Any]] = []
    records = value.get("records") if isinstance(value, dict) else None
    for record in records or []:
        if not isinstance(record, dict):
            continue
        period = record.get("period") if isinstance(record.get("period"), dict) else {}
        start = _text(period.get("fraDato")) or None
        end = _text(period.get("tilDato"))
        if not end:
            continue  # a figure without a reporting period is not published
        account_type = _text(record.get("account_type")) or "UNKNOWN"
        currency = _text(record.get("currency")) or None
        for metric in FINANCIAL_METRICS:
            amount = record.get(metric)
            if isinstance(amount, bool) or not isinstance(amount, (int, float)):
                amount = None
            shown = "(empty)" if amount is None else f"{amount} {currency or ''}".strip()
            specs.append({
                "field": f"financials.{metric}",
                "value": None if amount is None else {"amount": amount, "currency": currency},
                "claim_key": f"financials.{metric}:{account_type}:{end}",
                "reporting_period": {"start": start, "end": end},
                "span": f"Regnskap {_text(record.get('record_id'))} {account_type} {start or '?'}..{end}: {metric} = {shown}",
            })
    return specs


def role_member_specs(value: Any) -> list[dict[str, Any]]:
    """One spec per active role. Deregistered (inactive) roles are skipped."""
    specs: list[dict[str, Any]] = []
    seen: set[str] = set()
    roles = value.get("roles") if isinstance(value, dict) else None
    for item in roles or []:
        if not isinstance(item, dict) or item.get("inactive"):
            continue
        code = _text(item.get("role_code"))
        who = _text(item.get("organisation_number")) or _text(item.get("name"))
        if not code or not who:
            continue
        key = f"role:{code}:{who}"
        if key in seen:
            continue
        seen.add(key)
        spec: dict[str, Any] = {
            "field": "role",
            "value": {name: item.get(name) for name in ("name", "organisation_number", "role_code", "role", "group_code", "group")},
            "claim_key": key,
            "span": f"{_text(item.get('role')) or code}: {_text(item.get('name')) or who}",
        }
        if _text(item.get("last_changed")):
            spec["effective_at"] = _text(item.get("last_changed"))
        specs.append(spec)
    return specs


def location_member_specs(value: Any) -> list[dict[str, Any]]:
    """One spec per registered sub-unit with an organisation number."""
    specs: list[dict[str, Any]] = []
    seen: set[str] = set()
    locations = value.get("locations") if isinstance(value, dict) else None
    for item in locations or []:
        if not isinstance(item, dict):
            continue
        org = _text(item.get("organisation_number"))
        if not org or org in seen:
            continue
        seen.add(org)
        address = item.get("address") if isinstance(item.get("address"), dict) else {}
        place = " ".join(part for part in (_text(address.get("postnummer")), _text(address.get("poststed"))) if part)
        specs.append({
            "field": "registered_workplace",
            "value": {name: item.get(name) for name in ("organisation_number", "name", "address", "industry", "employees")},
            "claim_key": f"registered_workplace:{org}",
            "span": f"Underenhet {org} {_text(item.get('name'))}" + (f", {place}" if place else ""),
        })
    return specs


_GROUP_SKIP_KEYS = frozenset({"organisasjonsform", "knytningsform"})


def _group_nodes(node: Any, parent_org: str, parent_name: str, out: list[tuple[dict[str, Any], str, str]]) -> None:
    if isinstance(node, list):
        for child in node:
            _group_nodes(child, parent_org, parent_name, out)
        return
    if not isinstance(node, dict):
        return
    org = _text(node.get("organisasjonsnummer"))
    name = _text(node.get("navn"))
    if org:
        out.append((node, parent_org, parent_name))
    for key, child in node.items():
        if key in _GROUP_SKIP_KEYS or not isinstance(child, (list, dict)):
            continue
        _group_nodes(child, org or parent_org, name or parent_name, out)


def group_member_specs(value: Any, organisation_number: str) -> list[dict[str, Any]]:
    """One spec per entity related to the target in the konsernstruktur body.

    The target itself is skipped. Relation direction is taken from the node's own
    parentOrganisasjonsnummer when present, otherwise from its position in the tree.
    """
    nodes: list[tuple[dict[str, Any], str, str]] = []
    _group_nodes(value, "", "", nodes)
    target = _text(organisation_number)
    specs: list[dict[str, Any]] = []
    seen: set[str] = set()
    for node, tree_parent_org, tree_parent_name in nodes:
        org = _text(node.get("organisasjonsnummer"))
        if not org or org == target:
            continue
        parent_org = _text(node.get("parentOrganisasjonsnummer")) or tree_parent_org
        parent_name = _text(node.get("parentNavn")) or tree_parent_name
        link = node.get("knytningsform") if isinstance(node.get("knytningsform"), dict) else {}
        key = f"group_relation:{parent_org or '-'}:{org}"
        if key in seen:
            continue
        seen.add(key)
        spec: dict[str, Any] = {
            "field": "group_relation",
            "value": {
                "organisation_number": org,
                "name": _text(node.get("navn")) or None,
                "parent_organisation_number": parent_org or None,
                "parent_name": parent_name or None,
                "link_code": _text(link.get("kode")) or None,
                "link_description": _text(link.get("beskrivelse")) or None,
                "ownership_basis": _text(node.get("grunnlag")) or None,
                "level": node.get("nivaa"),
            },
            "claim_key": key,
            "span": (
                f"Konsernstruktur: {_text(node.get('navn'))} ({org}), "
                f"{_text(link.get('beskrivelse')) or 'relasjon'} under {parent_name or '-'} ({parent_org or '-'})"
                + (f", grunnlag {_text(node.get('grunnlag'))}" if _text(node.get("grunnlag")) else "")
            ),
        }
        if _text(node.get("dato")):
            spec["effective_at"] = _text(node.get("dato"))
        specs.append(spec)
    return specs


def build_member_specs(family: str, value: Any, organisation_number: str) -> list[dict[str, Any]]:
    if family == "financials":
        return financial_member_specs(value)
    if family == "roles":
        return role_member_specs(value)
    if family == "locations":
        return location_member_specs(value)
    if family == "group":
        return group_member_specs(value, organisation_number)
    return []


def explode_legacy_claims(claims: list[dict[str, Any]], organisation_number: str) -> list[dict[str, Any]]:
    """Turn available legacy blob claims (field == family slot) into member claims.

    Used by change detection so a previous envelope in the old blob shape compares
    item-by-item with a current envelope in the member shape. Claims that are not
    available blobs, and blobs whose shape is not recognised, are returned unchanged.
    """
    out: list[dict[str, Any]] = []
    for claim in claims:
        if not isinstance(claim, dict):
            continue
        field = claim.get("field")
        if field in CLAIM_FAMILIES and claim.get("availability") == "available" and isinstance(claim.get("value"), dict):
            specs = build_member_specs(str(field), claim["value"], organisation_number)
            if specs:
                for spec in specs:
                    available = spec["value"] is not None
                    member: dict[str, Any] = {
                        "field": spec["field"],
                        "value": spec["value"],
                        "availability": "available" if available else "not_available",
                        "confidence": claim.get("confidence") if available else None,
                        "evidence_ids": list(claim.get("evidence_ids") or []),
                        "claim_key": spec["claim_key"],
                    }
                    for optional in ("reporting_period", "effective_at"):
                        if optional in spec:
                            member[optional] = spec[optional]
                    out.append(member)
                continue
        out.append(claim)
    return out
```

**Edge cases the code above already handles (do not remove):** missing `period.tilDato` (record skipped), metric `0` (kept, real zero), metric missing (`value None`), boolean in a numeric field (treated as missing), inactive roles, duplicate roles for the same person and code, sub-units without organisation number, the target itself appearing in its own group tree, unknown group shape (returns `[]`).

**Done when (static).** All functions exist with these names and signatures; no package imports were added to `claim_builders.py`; nothing calls them yet except nothing (Phase 3 and 4 will).

---

### Phase 3: Refresh compatibility for member claims

**Goal.** `diff_claims()` compares singletons by `field` (as today) and family members by `claim_key`, explodes legacy blobs from older envelopes, and skips fields that only exist on one side because the schema grew.

**Existing.** `changes.py` `diff_claims()` (L210-~363) builds `prev_by_field` / `curr_by_field`, applies P0 skip rules (`_is_placeholder`, `_UNCOMPARABLE_AVAILABILITY`, `_DEPENDENT_FIELDS`), then scalar or array diff. `_change_record()` (L424) writes the record.

**Step 3.1 Imports.** `from .claim_builders import CLAIM_FAMILIES, MEMBER_TO_FAMILY, explode_legacy_claims`.

**Step 3.2 Normalise both sides first.** At the start of `diff_claims()`:
```python
previous_claims = explode_legacy_claims(list(previous_claims or []), organisation_number)
current_claims = explode_legacy_claims(list(current_claims or []), organisation_number)
```

**Step 3.3 Split singletons from families.** Build `prev_by_field` / `curr_by_field` exactly as today **but exclude** any claim whose field is a family slot name or a member field (`field in CLAIM_FAMILIES or field in MEMBER_TO_FAMILY`). Then build, for each side:
```python
def _family_entries(claims):
    entries = {family: {"placeholder": None, "members": {}} for family in CLAIM_FAMILIES}
    for claim in claims:
        if not isinstance(claim, dict):
            continue
        field = claim.get("field")
        if field in CLAIM_FAMILIES:
            entries[field]["placeholder"] = claim
        elif field in MEMBER_TO_FAMILY and claim.get("claim_key"):
            entries[MEMBER_TO_FAMILY[field]]["members"][str(claim["claim_key"])] = claim
    return entries
```

**Step 3.4 Schema-evolution rule for singletons.** In the existing singleton loop, replace the "removed" and "added" branches with `continue` when the field is missing on one side. Reason: every pipeline envelope always carries every singleton slot (`_complete_claims`), so a missing field means the envelopes come from different schema versions (for example the four new registry fields of Phase 4), not a real-world change. Real appear/disappear transitions are still reported as availability changes (`not_available` with evidence to `available` and back). Add a comment saying exactly this.

**Step 3.5 Family diff.** After the singleton loop:
```python
def _family_state(entry):
    if entry["members"]:
        return "members"
    placeholder = entry["placeholder"]
    if placeholder is None:
        return "absent"
    availability = placeholder.get("availability")
    if availability in _UNCOMPARABLE_AVAILABILITY:
        return "uncomparable"
    if _is_placeholder(placeholder):
        return "unchecked"
    if availability == "not_available":
        return "confirmed_empty"
    if availability == "available":
        return "blob"          # shape not recognised by the builder (group fallback)
    return "uncomparable"      # not_applicable, ambiguous
```
For each family in `sorted(CLAIM_FAMILIES)`:
* `p, c = _family_state(prev), _family_state(curr)`.
* If both are `"blob"`: compare the two placeholder claims with the existing scalar logic (one `updated` record if value or availability differ).
* Else if both are in `{"members", "confirmed_empty"}`: for every key in `sorted(set(prev_members) | set(curr_members))`:
  * only in current: `added` (previous value None, previous availability None);
  * only in previous: `removed`;
  * in both: `updated` if `not _scalar_equal(prev.value, curr.value, field=member_field)` or availabilities differ; otherwise nothing.
  * Record `field` = the member field (for example `role`), plus `claim_key` = the key.
  * Evidence: `current_evidence_ids` from the current member; `previous_evidence` resolved from the previous member's `evidence_ids` through the existing `prev_ev_by_id` map; `previous_run_id` as today.
* Any other combination (`absent`, `unchecked`, `uncomparable`, or `blob` versus `members`): skip the whole family.

**Step 3.6 `_change_record()`.** Add keyword-only `claim_key: str | None = None`; when not `None`, set `record["claim_key"] = claim_key` (place it next to `array_item_key`). No other new key.

**Step 3.7 Validator check 17** (`check_changes_well_formed`, L313): the duplicate key tuple becomes `(field, change.get("array_item_key"), change.get("claim_key"), change_type)`.

**Preserve.** `_FROZEN_FIELDS`, `_UNCOMPARABLE_AVAILABILITY`, `_DEPENDENT_FIELDS`, placeholder skip, evidence-reference keys, determinism (sorted iteration), comparison on value and availability only (never on evidence IDs, spans, confidence, `retrieved_at`).

**Edge cases.**
* Previous envelope from `run-20261014-010747-274033` (blob `roles` with 6 roles) versus current members: the previous blob is exploded with the same builder, keys match, unchanged roles produce nothing.
* A board member leaves: one `removed` record for `role` with that `claim_key`.
* Previous `roles` failed, current members: family skipped (no mass `added`).
* Roles module confirmed empty now (`not_available` with evidence), members before: one `removed` per previous role. This is a real registry change.
* Financial metric restated: one `updated` on `financials.<metric>` with the same `claim_key`.
* New annual-account year: new keys appear as `added`; old years that drop out of the BRREG 3-record window appear as `removed`. Accept this (it mirrors the source; see Q4).
* `group` blob kept because the shape is unrecognised on both sides: compared as one value.

**Done when (static).** Both sides go through `explode_legacy_claims`; singletons exclude family fields; one-sided singleton fields are skipped; family diff implements the state table; `_change_record` supports `claim_key`; check 17 tuple updated.

---

### Phase 4: Emit member claims and the four registry claims (P1-1 output change)

**Precondition.** Phase 3 is present (`changes.py` imports `explode_legacy_claims`). If not, stop.

**Step 4.1 Registry claims.** Append to `REGISTRY_CLAIM_FIELDS` (L154-164), at the end, in this order:
```python
    ("industry", ("industry", "beskrivelse")),
    ("municipality_number", ("business_address", "kommunenummer")),
    ("postal_address", "postal_address"),
    ("registry_declared_website", "website"),
```
`CLAIM_FIELD_ORDER` is derived from this tuple, so the new slots appear automatically after `latest_submitted_accounts`. No other change is needed: the existing loop (L272-279) and the P0-3 logic produce `available` or `not_available` per field.

**Step 4.2 Module loop (L307-319).** Import `build_member_specs` from `.claim_builders`. Replace the loop body with:

```python
    for field in MODULE_CLAIM_FIELDS:
        record = records.get(field)
        if not isinstance(record, dict):
            continue
        availability = _availability(record, field=field)
        if availability is None:
            continue
        value = record.get("value") if availability in {"available", "not_available"} else None

        if field in CLAIM_FAMILIES and availability == "available":
            specs = build_member_specs(field, value, organisation_number)
            if specs:
                for spec in specs:
                    member_id = add_evidence(
                        f"{field}:{spec['claim_key']}", record,
                        claim_span=_claim_span(profile, record),   # Phase 5 replaces this with spec["span"]
                        availability="available",
                    )
                    claims.append(_claim(
                        spec["field"], spec["value"], "available",
                        _confidence("available", official=True), [member_id],
                        claim_key=spec["claim_key"],
                        reporting_period=spec.get("reporting_period"),
                        effective_at=spec.get("effective_at"),
                    ))
                continue
            if field != "group":
                # Module answered but no publishable member exists (for example only
                # deregistered roles): a confirmed-empty placeholder, evidence kept.
                evidence_id = add_evidence(field, record, claim_span=_claim_span(profile, record), availability="not_available")
                claims.append(_claim(field, None, "not_available", _confidence("not_available"), [evidence_id]))
                continue
            # group with an unrecognised shape: fall through and keep today's blob claim

        evidence_id = add_evidence(
            field, record, claim_span=_claim_span(profile, record),
            availability=availability,
        )
        claims.append(_claim(field, value, availability, _confidence(availability, official=True), [evidence_id]))
        _append_error(errors, field, record, availability)
```

Notes for the implementer:
* The `_claim()` guard turns a financial member with `value None` into `not_available`, `confidence None`, keeping `claim_key`, `reporting_period` and the evidence. That is intended (missing metric is not zero).
* `financial_history` and `accounting_obligation` are not families; they keep the existing path.
* Do not touch the website block (L281-305) or the observation loop (L321-348).
* `add_evidence` already deduplicates by its key; the member key `f"{field}:{claim_key}"` is unique per member.

**Step 4.3 Nothing else.** `build_output_contract.py`, `run_agent.py`, `run.py` need no change. `load_previous_envelopes()` works unchanged because legacy explosion happens inside `diff_claims()`.

**Edge cases.**
* Company with 21 roles (`810098252`): 21 `role` claims and 21 evidence items.
* Company without group (BRREG 404): `group` placeholder `not_available` as today.
* Financials record with all metrics present: 7 claims per record; 3 records give 21 claims.
* `_is_empty_checked()` (P0) still makes empty `roles`, `locations`, `financials` records `not_available` before this code runs; those keep the old placeholder path.

**Expected output for `810034882` (from the Oct-14 data):** 6 `role`, 3 `registered_workplace`, 21 `financials.*`, `group` placeholder `not_available`, plus the 4 new registry claims (`industry` = "Elektrisk installasjonsarbeid", `municipality_number` = "1108", `postal_address` per registry, `registry_declared_website` `not_available`).

**Done when (static).** `REGISTRY_CLAIM_FIELDS` has 13 entries; the module loop matches the code above; no website or observation code changed.

---

### Phase 5: Field-specific evidence spans (P1-2)

**Goal.** Every available claim's evidence `claim_span` names the source field and contains the value. Same `source_url`, `retrieved_at` and `content_sha256` as today; only more evidence items with precise spans.

**Step 5.1 `claim_builders.py`: span helpers.**
```python
REGISTRY_SOURCE_PATHS: dict[str, str] = {
    "legal_name": "navn",
    "legal_form": "organisasjonsform.kode",
    "employees": "antallAnsatte",
    "industry_code": "naeringskode1.kode",
    "municipality": "forretningsadresse.kommune",
    "business_address": "forretningsadresse",
    "bankrupt": "konkurs",
    "liquidating": "underAvvikling",
    "latest_submitted_accounts": "sisteInnsendteAarsregnskap",
    "industry": "naeringskode1.beskrivelse",
    "municipality_number": "forretningsadresse.kommunenummer",
    "postal_address": "postadresse",
    "registry_declared_website": "hjemmeside",
}


def render_span_value(value: Any) -> str:
    if value is None:
        return "(empty)"
    if isinstance(value, bool):
        return "true" if value else "false"
    if isinstance(value, dict):
        parts: list[str] = []
        lines = value.get("adresse")
        if isinstance(lines, list):
            parts.extend(_text(line) for line in lines if _text(line))
        place = " ".join(p for p in (_text(value.get("postnummer")), _text(value.get("poststed"))) if p)
        if place:
            parts.append(place)
        for key in ("kommune", "land"):
            if _text(value.get(key)):
                parts.append(_text(value.get(key)))
        if parts:
            return ", ".join(parts)
        return ", ".join(f"{k}={_text(v)}" for k, v in sorted(value.items()) if not isinstance(v, (dict, list)))
    if isinstance(value, list):
        return "; ".join(render_span_value(item) for item in value)
    return _text(value)


def registry_span(field: str, value: Any) -> str:
    return f"{REGISTRY_SOURCE_PATHS.get(field, field)}: {render_span_value(value)}"[:400]


def module_singleton_span(field: str, value: Any) -> str | None:
    """Span for the non-family module claims; None means keep the generic span."""
    if not isinstance(value, dict):
        return None
    if field == "financial_history":
        years = [str(year) for year in value.get("years") or []]
        return ("Filed annual-account years: " + (", ".join(years) if years else "(none)"))[:400]
    if field == "accounting_obligation":
        return (f"classification: {_text(value.get('classification'))}; "
                f"legal_form: {_text(value.get('legal_form'))}; "
                f"ruleset: {_text(value.get('ruleset_version'))}")[:400]
    return None
```
Every `REGISTRY_CLAIM_FIELDS` name must have an entry in `REGISTRY_SOURCE_PATHS`. If Phase 4 is reverted, the extra entries are harmless.

**Step 5.2 Registry loop (L262-279).** When `availability == "available"`: register one evidence per field instead of one shared item:
```python
            for field, path in REGISTRY_CLAIM_FIELDS:
                val = _nested_get(registry_value, path) if availability == "available" else None
                field_avail = "not_available" if availability == "available" and _is_missing_value(val) else availability
                if availability == "available":
                    field_evidence_id = add_evidence(
                        f"registry_live:{field}", registry,
                        claim_span=registry_span(field, val if field_avail == "available" else None),
                        availability=field_avail,
                    )
                else:
                    field_evidence_id = registry_id
                claims.append(_claim(
                    field, val if field_avail != "not_available" else None,
                    field_avail, _confidence(field_avail, official=True), [field_evidence_id],
                ))
```
* Keep the existing `registry_id = add_evidence("registry_live", ...)` call **only** for the non-available case. For the available case, move it inside an `else` so no unreferenced evidence item is created (validator check 05 rejects orphan evidence). Concretely: compute `registry_id` only when `availability != "available"`.
* Missing field span reads `"antallAnsatte: (empty)"`: the evidence proves absence at `retrieved_at`.

**Step 5.3 Member spans.** In the Phase 4 member block, replace `claim_span=_claim_span(profile, record)` with `claim_span=spec["span"][:400]`.

**Step 5.4 Module singletons.** In the module loop's final (non-member) path, use `module_singleton_span(field, value) or _claim_span(profile, record)` as `claim_span` when `availability == "available"`; keep `_claim_span(profile, record)` for every other availability (it carries the error note, for example `HTTP 404`).

**Preserve.** `PUBLIC_EVIDENCE_KEYS`; evidence ID pattern `ev-<org>-<n>` (generated by `add_evidence`); website and observation evidence untouched; `content_sha256` copied from the source record (BRREG records carry it, so check 06 still passes).

**Edge cases.** Registry `failed`/`blocked`/`not_applicable`: one shared evidence as today. `business_address` dict renders as `"Strandgata 21, 4307 SANDNES, SANDNES, Norge"`. `bankrupt false` renders `"konkurs: false"`. Spans are clipped to 400 characters.

**Done when (static).** Each available registry claim references its own evidence; no orphan registry evidence can be created; member evidence uses `spec["span"]`; `financial_history` and `accounting_obligation` use value spans.

---

## 7. File-by-file change map

| File | Phase | Function / section | Change |
|---|---|---|---|
| `src/norway_company_agent/claim_builders.py` (new) | 1, 2, 5 | constants; builders; `explode_legacy_claims`; span helpers | Create |
| `src/norway_company_agent/output_contract.py` | 1 | imports; `_claim()` L530; `_complete_claims()` L395; `assert_claim_schema_integrity()` L56 | Optional keys, families, schema rules 1b/1c |
| `scripts/validate_envelopes.py` | 1, 3 | imports; `check_claims()` L112; `check_changes_well_formed()` L313 | Family-aware check 02; check 17 key tuple |
| `src/norway_company_agent/changes.py` | 3 | imports; `diff_claims()` L210; `_change_record()` L424 | Legacy explosion, schema-evolution skip, family diff, `claim_key` |
| `src/norway_company_agent/output_contract.py` | 4 | `REGISTRY_CLAIM_FIELDS` L154; module loop L307-319 | 4 registry claims; member claims |
| `src/norway_company_agent/output_contract.py` | 5 | registry loop L262-279; member block; module singleton path | Per-field evidence and spans |

Must **not** change: `official.py`, `website.py`, `identity.py`, `discovery.py`, `crawl_events.py`, `page_signals.py`, `vocabulary.py`, `batch.py`, `evidence.py`, `scripts/run_competition_batch.py`, `scripts/run_agent.py`, `run.py`, `scripts/build_output_contract.py`, everything under `archive/`, `tests/`, `requirements.txt`, P0-1 artifacts.

---

## 8. Data-contract and compatibility considerations

* **Additive claim keys** (`claim_key`, `reporting_period`, `effective_at`) and **repeated member fields** are the only contract changes. Justification: the board-assessed competitor envelope uses repeated `role` / `registered_workplace` / `financials.*` fields and extra claim keys. Exact official field names are not published (Q1).
* **Removed blob claims** `financials`, `roles`, `locations`, `group` (when members exist). Any downstream reader of those blob values (none found in `src/` or `scripts/` besides the files in section 7; `_p6_verify.py` and `tests/` are off-path) will see placeholders or members instead.
* **Refresh across the upgrade.** The first refresh that compares a pre-P1 envelope with a post-P1 envelope produces no noise: blobs are exploded with the same builders and the four new registry fields are skipped as schema evolution.
* **Terminal status and modules** are unaffected (derived from module records, not claims).
* **Evidence volume** grows (roughly one item per claim). Validator checks 03, 05, 06 and 07 remain satisfiable by construction.
* **`SCHEMA_VERSION`** stays `"1.0"` because it is shared with fat envelopes and stage reports (`batch.py`, `run_competition_batch.py`), and the change is additive (Q2).

---

## 9. Static code-review checklist

Static inspection only. Do not run tests or any script.

**Phase 1**
- [ ] `claim_builders.py` has no `from .` or `import norway_company_agent` lines.
- [ ] `_claim()` keeps the P0-3 guard and only adds optional keys when not `None`.
- [ ] `_complete_claims()` drops a family placeholder when members exist and fills missing slots as before.
- [ ] `assert_claim_schema_integrity()` rejects unknown claim keys and member claims without `claim_key`.
- [ ] `check_claims()` matches section 6 Phase 1.

**Phase 2**
- [ ] Builder names and signatures match exactly; edge cases listed are handled.
- [ ] `FINANCIAL_METRICS` equals the keys in `official.normalize_financials()`.

**Phase 3**
- [ ] Both claim lists pass through `explode_legacy_claims()` first.
- [ ] Family slots and member fields are excluded from the singleton loop.
- [ ] One-sided singleton fields are skipped with the schema-evolution comment.
- [ ] Family state table implemented; mixed states skip the family.
- [ ] `_change_record()` accepts `claim_key`; check 17 tuple includes it.

**Phase 4**
- [ ] `REGISTRY_CLAIM_FIELDS` has the 4 new entries at the end.
- [ ] Member claims are emitted only for available family modules with non-empty specs.
- [ ] `group` with unknown shape keeps the blob claim.
- [ ] Website block and observation loop are byte-identical to before.

**Phase 5**
- [ ] Each available registry claim has its own evidence with `REGISTRY_SOURCE_PATHS` span.
- [ ] No unreferenced registry evidence is created in the available case.
- [ ] Member evidence uses `spec["span"]`.
- [ ] Non-available module claims still use `_claim_span()`.

**Global**
- [ ] Only files in section 7 changed. No new dependency. No test file created or edited. Nothing executed.

---

## 10. Explicit exclusions and deferred work

Excluded by the user: website discovery, website-derived claims, news, jobs, careers, social profiles, deep crawl.
Excluded by scope: OCR claims, P0-1 hygiene, 100/1000 company runs, performance, `operations` metrics, request-log URLs, stable cross-run evidence IDs, UI, tests.

Deferred observation (record only, do not implement): `prior_year_financials` observation claims are still one blob per company (OCR path, opt-in). If OCR is ever enabled for official runs, apply the same member pattern there.

---

## 11. Instructions for the future AI coding agent

1. Implement only the phase the user names, then stop and report.
2. Phase order is mandatory: 1, 2, 3, 4, 5. Never apply Phase 4 without Phase 3 in the code.
3. Re-read each target function before editing; if a described symbol is missing or different, stop and report instead of improvising.
4. Do not run tests, `pytest`, the pipeline, `run.py`, `run_agent.py`, any connector, any network call, or any script. Do not create or edit test files.
5. Do not modify files outside section 7 for that phase. Do not add dependencies. Do not rename existing functions; only add keyword-only parameters where this plan says so.
6. Do not touch anything website-related (section 1.3) or any P0-1 artifact.
7. Do not add claim keys other than `claim_key`, `reporting_period`, `effective_at`, and no change-record key other than `claim_key`.
8. After a phase, report the files and functions changed, any deviation and why, and tick that phase's section 9 items as statically reviewed. Never claim the code was tested or executed.
9. Never print, copy or commit values from `.env`.

---

## 12. Unresolved questions and assumptions

| # | Question | Assumption |
|---|---|---|
| Q1 | Exact official names for repeated claim fields and optional claim keys | Names in section 4 (aligned with the board-assessed competitor shape). Rename only via the constants in `claim_builders.py` if the contract specifies others. |
| Q2 | Should `SCHEMA_VERSION` change? | No; additive change, shared constant. |
| Q3 | Should deregistered roles be published as historical facts? | No; only active roles. |
| Q4 | BRREG returns at most 3 account records (`normalize_financials` slices `[:3]`); the oldest year drops out when a new year arrives, which refresh reports as `removed`. | Acceptable: mirrors the source. Preserving older years would need history storage (out of scope). |
| Q5 | Group structure beyond the observed `children` shape (for example a subsidiary queried from below) | Handled generically by the recursive node walk; unrecognised shapes keep the blob claim. |
| Q6 | Does the evaluator count member claims as individual facts? | Expected (the 30% claim-coverage component counts individual checked facts); not provable without the official contract or gold set. |
