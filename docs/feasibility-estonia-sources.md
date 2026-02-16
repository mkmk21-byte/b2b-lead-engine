# Feasibility & Strategy – Estonia B2B Outreach Lead Engine (Phase 0)

## Scope
This document defines a feasibility-first strategy for building a maintainable lead acquisition pipeline for Estonian B2B outreach (construction first).

## What Was Tested In This Environment

### Connectivity sanity checks
I attempted live source validation from this container using:

- `curl -I https://example.com -m 15`
- Python `urllib` requests against candidate domains:
  - `avaandmed.ariregister.rik.ee`
  - `inforegister.ee`
  - `1182.ee`
  - `ssb.ee`
  - `teatmik.ee`

### Result
Outbound HTTPS requests are blocked in this runtime (`CONNECT tunnel failed: 403 Forbidden`), so direct 5–10 company extraction tests could not be completed here.

Because of that, this report provides:
1. A pragmatic source-selection framework.
2. A concrete recommended architecture.
3. A minimal validation plan you can run immediately in a network-enabled environment.

---

## Source Comparison (Practical Engineering View)

## 1) Estonian Business Register Open Data (Äriregister / RIK open data)
**Role in pipeline:** Primary source of legal entities and classification metadata.

- **Technical difficulty:** Low–Medium (typically structured datasets/APIs).
- **Company-name extraction:** High confidence.
- **Website extraction:** Often partial or missing.
- **Decision-maker likelihood:** Medium for legal board/management identities, but role relevance to outreach persona can be limited.
- **Breakage risk:** Low (official source, stable schema compared to scraped HTML).
- **Scalability:** Very high.

**Verdict:** Best foundation for canonical company universe + registry identifiers.

## 2) Commercial/Directory Aggregators (SSB.ee, Inforegister, Teatmik, etc.)
**Role in pipeline:** Secondary enrichment (website/contact clues, cross-checks).

- **Technical difficulty:** Medium–High (many are SPA-heavy or anti-bot sensitive).
- **Company-name extraction:** Usually good, but DOM/JSON structures vary.
- **Website extraction:** Often available, but may be obfuscated or inconsistently placed.
- **Decision-maker likelihood:** Medium (sometimes includes representatives/board members).
- **Breakage risk:** High (front-end changes and anti-automation can break parsers).
- **Scalability:** Medium without frequent maintenance.

**Verdict:** Useful as enrichment only, not as the backbone.

## 3) Company Websites (direct crawl)
**Role in pipeline:** Primary source for real outreach contacts and role relevance.

- **Technical difficulty:** Medium (heterogeneous layouts, multilingual content).
- **Company-name extraction:** Not needed if starting from registry list.
- **Website extraction:** N/A (already targeted).
- **Decision-maker likelihood:** High on About/Team/Contact pages for SMB/mid-market.
- **Breakage risk:** Medium (site-specific variation, but robust heuristics can handle most).
- **Scalability:** High if you design a queue + polite crawler + extraction heuristics.

**Verdict:** Core enrichment layer for usable cold-email leads.

## 4) Professional Networks / Social Platforms
**Role in pipeline:** Optional decision-maker disambiguation.

- **Technical difficulty:** High (ToS restrictions, dynamic UI, anti-bot controls).
- **Decision-maker likelihood:** High data quality, but acquisition risk/compliance overhead high.
- **Breakage risk:** High.
- **Scalability:** Low–Medium unless using approved APIs/data providers.

**Verdict:** Avoid in phase 1 unless you have compliant API/provider access.

---

## Recommended Strategy (Clear Decision)

### Primary data source
**Estonian official open business data (Äriregister/RIK)** as the authoritative base table.

Why:
- Stable identifiers (registry code) reduce duplicates and merge errors.
- Easier to maintain than scraping SPA directories.
- Better long-term scalability and legal defensibility.

### Secondary enrichment sources
1. **Company websites** (contact/team/about/careers pages).
2. **One directory source at a time** (SSB or Inforegister or Teatmik) only for missing website/role fallback.
3. **MX/domain validation** as quality control before exporting to Smartlead.

### Sources to avoid (for now)
- Any source requiring login/paywall/API key you don’t yet have.
- Heavy SPA-first scraping as a core dependency.
- Any flow that depends on brittle browser clicks to collect base company lists.

---

## Suggested Tech Stack (Phase 1)

## Language/runtime
**Python** for phase 1.

Why:
- Fast iteration for parsing/enrichment/data cleaning.
- Strong ecosystem for scraping + ETL (`httpx`, `selectolax`/`beautifulsoup4`, `pandas`/`polars`, `pydantic`).

## Scraping/extraction tools
- **Default:** `httpx` + HTML parser (no browser).
- **Fallback:** `playwright` only for pages where static fetch fails.

Rule:
- Start with static HTTP extraction first; browser automation is an exception path.

## Storage
- **SQLite** for local reproducibility.
- Tables: `companies`, `websites`, `people`, `emails`, `provenance`, `runs`.

## Output
- Versioned CSV exports for Smartlead import.
- Deterministic schema:
  - `company_name`
  - `registry_code`
  - `website`
  - `decision_maker_name`
  - `role`
  - `email`
  - `source`

---

## Recommended Repository Structure

```text
b2b-lead-engine/
  README.md
  docs/
    feasibility-estonia-sources.md
  src/
    config.py
    models.py
    pipeline/
      ingest_registry.py
      enrich_websites.py
      extract_people_emails.py
      validate_export.py
    extractors/
      website_contact_extractor.py
      person_role_extractor.py
      email_pattern_inference.py
    integrations/
      registry_client.py
      directory_fallback_client.py
  data/
    raw/
    staging/
    curated/
    exports/
  tests/
    test_extractors.py
    test_dedup.py
```

---

## Minimal Validation Plan (Run This Before Scale)

1. Pull **50 construction companies** from primary registry source.
2. For first **10 companies**:
   - confirm website field presence/absence,
   - crawl contact/team pages,
   - extract people + roles + emails,
   - capture provenance URLs.
3. Compute phase-0 KPIs:
   - `% with website`,
   - `% with at least 1 decision-maker`,
   - `% with at least 1 plausible email`,
   - hard-bounce risk proxy (domain/MX checks).
4. Only then decide whether secondary directory enrichment is worth it.

---

## Scale Roadmap

## Phase 1 (Validation)
- 200–500 companies, construction only.
- Weekly runs.
- Manual QA sample (20–30 rows).

## Phase 2 (Stabilization)
- Add retries, backoff, and structured run logs.
- Add confidence scoring per email/person.
- Add one secondary enrichment source.

## Phase 3 (Automation)
- Scheduled runs (cron/GitHub Actions/server job).
- Delta updates only (new/changed companies).
- Push validated leads to Smartlead-ready CSV automatically.

## Phase 4 (Expansion)
- Add additional industries.
- Add multilingual extraction refinements (ET/EN/RU).
- Introduce human-in-the-loop review queue for low-confidence contacts.

---

## Practical Guidance
- Optimize for **data quality and reproducibility**, not maximum scrape volume.
- Keep a strict provenance trail (`source URL`, `extracted_at`, `method`).
- Treat browser automation as a targeted tool, not default architecture.
- Use one stable canonical ID (registry code) across the whole pipeline.
