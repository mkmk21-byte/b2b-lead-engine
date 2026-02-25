# Äriregister / RIK Open Data Analysis (Active Legal Entities)

## Scope
This note answers the implementation-specific questions for using official Estonian Business Register (RIK / Äriregister) open data as the base for active-company lead generation.

## Environment limitation
Direct live calls from this container to RIK endpoints are blocked (`CONNECT tunnel failed: 403 Forbidden`), so I cannot execute remote schema introspection here.

To keep this useful, this document includes:
1. The official dataset family to target.
2. The field mapping strategy you should apply.
3. Copy-paste commands/scripts to verify exact dataset names + exact field names locally.
4. A precise 50-company validation test for EMTAK section F (construction).

---

## 1) Official dataset(s) to use for active legal entities

Use the **RIK open data (Äriregistri avaandmed)** dataset family and specifically the **legal entity/company records** feed (not beneficial owner, not filings, not annual reports).

In the RIK portal, this is typically exposed as one or both of:

- **Bulk dataset resource** for legal entities (CSV/JSON dump; periodic refresh).
- **REST API endpoint(s)** for company/entity records.

Because endpoint/resource naming can change by language/version, treat the exact resource title as a runtime discovery step (commands below), then lock the discovered resource ID in config.

### Selection criteria for the correct dataset resource
Pick the resource that contains all of:
- registry identifier (company register code),
- legal name,
- status (to filter active entities),
- EMTAK/NACE activity code(s),
- optional contact fields (including website/domain when present).

---

## 2) Access method

Recommended order:

1. **Bulk export (CSV/JSON)** for base ingestion and reproducibility.
2. **API** for incremental refresh/delta checks and per-entity enrichment.

Why:
- Bulk is better for deterministic snapshots and backfills.
- API is better for ongoing sync.

---

## 3) Schema structure and exact-field discovery

Because this environment cannot query RIK directly, use the discovery workflow below to retrieve **exact field names** from the live schema metadata in your local environment.

## 3.1 Field groups you need
- Identity: registry code, company/legal name.
- Classification: EMTAK code (primary and/or list).
- Status: active/inactive marker.
- Contact: website/domain (if available).

## 3.2 Known naming patterns (for quick orientation)
In RIK/open-data resources, the above commonly appear under names similar to:
- registry code: `ariregistri_kood` / `reg_kood` / `registry_code`
- company name: `nimi` / `arik_nimi` / `company_name`
- EMTAK: `emtak` / `emtak_kood` / `tegevusala_emtak`
- website: `veebileht` / `www` / `internet_aadress` / `website`

Do **not** hardcode these guesses; confirm exact names with schema calls first.

---

## 4) Exact commands to fetch dataset/resource names and field schema locally

> Run these on a machine with outbound internet access.

## 4.1 Discover dataset package/resources
```bash
curl -sS "https://avaandmed.ariregister.rik.ee" > /tmp/rik_home.html
# Then inspect links to API docs, data catalog, or download resources.
```

If the portal exposes CKAN-style metadata/API, use:
```bash
# Example pattern only; replace with actual discovered API base
curl -sS "<DISCOVERED_API_BASE>/package_list" | jq .
curl -sS "<DISCOVERED_API_BASE>/package_show?id=<DATASET_ID>" | jq .
```

## 4.2 Extract exact schema field names from the chosen company resource
For CSV resource:
```bash
python - <<'PY'
import pandas as pd
p = "company_resource.csv"   # downloaded from RIK
# read header only
cols = list(pd.read_csv(p, nrows=0).columns)
print("\n".join(cols))
PY
```

For JSON resource:
```bash
python - <<'PY'
import json
p = "company_resource.json"  # downloaded from RIK
with open(p, "r", encoding="utf-8") as f:
    data = json.load(f)
row = data[0] if isinstance(data, list) else data.get("results", [])[0]
print("\n".join(row.keys()))
PY
```

## 4.3 Confirm exact required mappings
Once you have field names, create this mapping config:

```yaml
registry_code: <EXACT_FIELD>
company_name: <EXACT_FIELD>
emtak_code: <EXACT_FIELD>
website: <EXACT_FIELD_OR_NULL_IF_MISSING>
status: <EXACT_FIELD>
```

---

## 5) Does a website/domain field exist?

In RIK legal-entity open data, a website/contact URL field is commonly present in at least some resources, but population completeness is variable.

**Your operational assumption should be:**
- field may exist,
- coverage is partial,
- missing values are expected and require secondary website discovery/enrichment.

Use the validation script below to confirm both existence and fill rate in your chosen resource.

---

## 6) 50-company EMTAK F validation test (construction)

If direct sampling is unavailable (as in this environment), run this exact local test.

### 6.1 Test objective
Estimate `% of active construction companies (EMTAK F) with non-empty website`.

### 6.2 Script
Save as `scripts/validate_rik_construction_website_rate.py` and run after downloading the chosen legal-entity dataset.

```python
import pandas as pd
from pathlib import Path

# --- configure these after schema discovery ---
CSV_PATH = Path("company_resource.csv")
REGISTRY_FIELD = "<EXACT_REGISTRY_CODE_FIELD>"
NAME_FIELD = "<EXACT_COMPANY_NAME_FIELD>"
EMTAK_FIELD = "<EXACT_EMTAK_FIELD>"
STATUS_FIELD = "<EXACT_STATUS_FIELD>"
WEBSITE_FIELD = "<EXACT_WEBSITE_FIELD_OR_LEAVE_EMPTY_IF_NONE>"
ACTIVE_VALUE = "active"  # replace with exact value used in dataset

# ----------------------------------------------

def is_construction(code: str) -> bool:
    # EMTAK/NACE section F => codes that start with "F" OR numeric 41/42/43 families
    if code is None:
        return False
    s = str(code).strip().upper()
    return s.startswith("F") or s.startswith("41") or s.startswith("42") or s.startswith("43")


def has_website(v) -> bool:
    if v is None:
        return False
    s = str(v).strip()
    return s != "" and s.lower() not in {"nan", "none", "null", "-"}


def main():
    df = pd.read_csv(CSV_PATH, dtype=str, keep_default_na=False)

    # active companies only
    active = df[df[STATUS_FIELD].str.strip().str.lower() == ACTIVE_VALUE.lower()].copy()

    # construction companies only
    c = active[active[EMTAK_FIELD].apply(is_construction)].copy()

    # deterministic sample of 50
    sample = c.sort_values([REGISTRY_FIELD]).head(50).copy()

    if WEBSITE_FIELD and WEBSITE_FIELD in sample.columns:
        sample["website_present"] = sample[WEBSITE_FIELD].apply(has_website)
        pct = (sample["website_present"].mean() * 100.0) if len(sample) else 0.0
    else:
        sample["website_present"] = False
        pct = 0.0

    print(f"sample_size={len(sample)}")
    print(f"website_field_exists={WEBSITE_FIELD in sample.columns if WEBSITE_FIELD else False}")
    print(f"website_populated_pct={pct:.2f}")

    cols = [REGISTRY_FIELD, NAME_FIELD, EMTAK_FIELD]
    if WEBSITE_FIELD and WEBSITE_FIELD in sample.columns:
        cols.append(WEBSITE_FIELD)
    cols.append("website_present")

    out = Path("construction_f50_website_check.csv")
    sample[cols].to_csv(out, index=False)
    print(f"wrote={out}")


if __name__ == "__main__":
    main()
```

### 6.3 Run
```bash
python scripts/validate_rik_construction_website_rate.py
```

### 6.4 Output you should record
- sample size (expect 50),
- whether website field exists,
- `% website_populated_pct`,
- CSV evidence file for manual QA.

---

## 7) Delivery checklist (what to lock into your pipeline config)
After local run, store these exact values in `config`:

- dataset/resource ID,
- access method URL (bulk/API),
- exact field names for:
  - `registry_code`,
  - `company_name`,
  - `emtak_code`,
  - `website` (or explicit `null` if absent),
- active-status filter value,
- measured `EMTAK F website_populated_pct` from the 50-company sample.

This gives you an auditable and repeatable base ingestion layer before any enrichment scraping.
