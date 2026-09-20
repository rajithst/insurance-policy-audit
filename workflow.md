# workflow.md

## Execution Plan: Insurance Coverage Reality Check

This workflow maps a high-level insurance contract to its underlying legal framework to expose exclusions, edge cases, and hidden benefits. The workflow is sequential — each phase depends on the output of the previous one.

---

### Phase 0: Pre-Flight Validation

Before any agent runs, verify the environment is ready.

**Checks:**

1. **Contract file exists:**
   - Scan `uploads/contracts/` — confirm at least one PDF is present.
   - If multiple PDFs are found, list them and ask the user which one to analyze.
   - If no PDF is found → **STOP.** Instruct the user to place their contract PDF in `uploads/contracts/`.

2. **Policy booklets exist:**
   - Scan `uploads/policy-booklets/` — confirm at least one PDF is present.
   - Log how many versions are available and their filename-encoded date ranges.
   - If the directory is empty → **STOP.** Instruct the user to add the relevant policy booklet(s).

3. **Roadside terms exist:**
   - Scan `uploads/roadside-terms/` — confirm at least one PDF is present.
   - Log how many versions are available and their filename-encoded date ranges.
   - If the directory is empty → Check if the policy booklet in `uploads/policy-booklets/` bundles roadside assistance terms. If not, **WARN** (roadside analysis will be skipped, but the rest can proceed).

4. **Output directories:**
   - Create `temp/` if it doesn't exist.
   - Create `reports/` if it doesn't exist.

**Gate:** All critical checks pass → proceed to Phase 1.

---

### Phase 1: Establish the Contract Baseline

**Agent:** `The Contract Anchor`  
**Input:** The PDF in `uploads/contracts/`  
**Output:** `temp/baseline-contract.json`

**Execution Steps:**

1. Trigger the Contract Anchor agent.
2. The agent reads the contract PDF and extracts all structured data.
3. The agent writes `temp/baseline-contract.json`.

**Validation Checkpoint:**
- [ ] `temp/baseline-contract.json` exists and is valid JSON.
- [ ] `policy.inception_date` is present and in `YYYY-MM-DD` format.
- [ ] `policy.expiration_date` is present.
- [ ] `coverages` array has at least one entry.
- [ ] `vehicle.powertrain_type` is set (not empty).
- [ ] If `extraction_warnings` is non-empty, display them to the user before proceeding.

**Error Path:**
- If inception date is missing → **STOP.** The agent must report what it found and what went wrong. The user may need to provide the date manually or upload a clearer scan.
- If no coverages are extracted → **WARN.** Proceed but flag that downstream analysis will be limited.

---

### Phase 2: Date-Based Document Selection & Forensic Clause Tracing

**Agent:** `The Clause Tracer`  
**Input:** `temp/baseline-contract.json` + all PDFs in `uploads/policy-booklets/` and `uploads/roadside-terms/`  
**Output:** `temp/traced-clauses.md`

**Execution Steps:**

1. Trigger the Clause Tracer agent.
2. The agent reads the baseline JSON to get the inception date and insurer domain (`policy.insurer_domain`).
3. **Date Matching (Critical Sub-step):**
   - The agent scans all filenames in `uploads/policy-booklets/`.
   - For each filename, it extracts the validity date range using the parsing rules documented in `AGENTS.md` → Document Inventory section.
   - It selects the **one** booklet whose date range contains the inception date.
   - It repeats this for `uploads/roadside-terms/` (or verifies if roadside terms are integrated within the policy booklet).
   - It outputs a **Document Selection Audit** log showing all candidates, their parsed ranges, and which one was selected (or why none matched).
4. The agent reads the selected documents in full, tracking exact page numbers, chapters, and articles.
5. For every coverage item and endorsement in the baseline, the agent traces it to the exact chapter/article and page in the booklet.
6. **Provider Official Web Research & Link Enrichment:**
   - Using web search and URL reading tools, the agent queries the underwriting provider's official domain (`policy.insurer_domain`).
   - Gathers verified URLs for compensation guides, roadside services, accident reporting, and My Page account management.
7. The agent writes `temp/traced-clauses.md`.

**Validation Checkpoint:**
- [ ] `temp/traced-clauses.md` exists and is non-empty.
- [ ] The Document Selection Audit is present at the top of the file.
- [ ] Exactly one policy booklet was selected (filename logged).
- [ ] Exactly one roadside terms document was selected (filename logged), OR the policy booklet was confirmed to include integrated roadside assistance terms.
- [ ] Every coverage item includes exact PDF filenames, page numbers, and article numbers.
- [ ] Official provider links from the insurer's domain are verified and integrated.
- [ ] Exclusions (免責事項) sections are populated with exact article and paragraph citations.

**Error Paths:**

| Scenario | Action |
|---|---|
| No matching policy booklet for the inception date | **STOP.** Report the inception date and all available booklet date ranges. Ask the user to upload the correct booklet version. |
| No matching roadside terms for the inception date | **WARN.** Skip roadside analysis. Note this gap in the output. Continue with booklet analysis only. |
| Multiple booklets match (overlapping ranges) | Select the one with the **narrowest** date range. Log the decision. |
| A coverage item has no corresponding chapter in the booklet | Do NOT skip it. List it under "Unmapped Items" with a flag. |
| The booklet appears to be for a different product/insurer | **STOP.** Likely a date mismatch or wrong document. Re-verify. |

---

### Phase 3: Gap & Benefit Synthesis

**Agent:** `The Gap & Benefit Synthesizer`  
**Input:** `temp/baseline-contract.json` and `temp/traced-clauses.md`  
**Output:** `reports/Policy_Coverage_Audit_Report_EN.md` and `reports/Policy_Coverage_Audit_Report_JA.md`

**Execution Steps:**

1. Trigger the Gap & Benefit Synthesizer agent.
2. The agent reads both input files.
3. The agent generates two comprehensive, professional reports (one in English and one in Japanese) structured into:
   - Coverage Reality Table (with PDF filename, page number, chapter/article, and official website links)
   - Danger Zone (risk-severity classified, with exact exclusion article citations and impact)
   - Hidden Usable Benefits (with contract entitlement proof, passenger coverage scope, and **4-column structured tables** for both 24/7 Roadside Assistance and Stranded Travel Protection, official contact numbers, and mandatory first-call reimbursement procedures)
   - Powertrain-Specific Analysis (for Hybrid/EV/e-POWER vehicles)
   - Recommendations (actionable next steps and optimization)
   - Quick Reference Card (emergency numbers, checklist, and key constraints)
4. The agent writes:
   - `reports/Policy_Coverage_Audit_Report_EN.md`
   - `reports/Policy_Coverage_Audit_Report_JA.md`

**Validation Checkpoint:**
- [ ] `reports/Policy_Coverage_Audit_Report_EN.md` exists and is non-empty.
- [ ] `reports/Policy_Coverage_Audit_Report_JA.md` exists and is non-empty.
- [ ] All six sections are present in both reports with granular citations (PDF filename, page, chapter, article).
- [ ] Clickable, verified official provider URLs are embedded in each relevant section.
- [ ] Every coverage item from the baseline appears in both reports.
- [ ] Any "Unmapped Items" from the traced clauses are flagged prominently.
- [ ] Any `extraction_warnings` from the baseline are acknowledged.
- [ ] Both reports contain no invented coverage or exclusion that isn't backed by the traced clauses.

**Error Path:**
- If traced clauses contain many unmapped items → The report must have a prominent warning section explaining that the analysis is incomplete.

---

### Phase 4: Human Review & Iteration

**This phase is manual.** After the report is generated:

1. **Present the reports** (`reports/Policy_Coverage_Audit_Report_EN.md` and `reports/Policy_Coverage_Audit_Report_JA.md`) to the user.
2. **Highlight key areas for review:**
   - Any extraction warnings from Phase 1.
   - Any unmapped items from Phase 2.
   - The most critical exclusions (🔴 items) from the Danger Zone.
3. **Solicit feedback:**
   - Does the coverage mapping look accurate?
   - Are there coverage items the user expected but that aren't in the report?
   - Does the user want deeper analysis on any specific area?
4. **If revisions are needed:**
   - If the issue is in the contract extraction → re-run from Phase 1.
   - If the issue is in the clause tracing → re-run from Phase 2.
   - If the issue is only in the report formatting/emphasis → re-run Phase 3 only.

---

### Phase 5: Market Benchmarking (Optional / On-Demand)

> **This phase does NOT run automatically.** It is triggered independently when the user explicitly requests a market comparison, competitive analysis, or premium benchmarking.

**Agent:** `The Market Analyst`  
**Input:** `temp/baseline-contract.json` + `reports/Policy_Coverage_Audit_Report_EN.md`  
**Output:** `temp/market-search-profile.json`, `reports/Market_Comparison_Report_EN.md` and `reports/Market_Comparison_Report_JA.md`

**Prerequisites:**
- `temp/baseline-contract.json` MUST exist. If it doesn't, instruct the user to run the standard audit pipeline first (Phases 0–3).
- `reports/Policy_Coverage_Audit_Report_EN.md` SHOULD exist for context enrichment. If missing, proceed with baseline data only.

**Execution Steps:**

1. Trigger the Market Analyst agent.
2. The agent reads the baseline contract JSON and builds a standardized search profile.
3. The agent writes `temp/market-search-profile.json`.
4. **Web Research:**
   - The agent searches Japanese auto insurance comparison portals (kakaku.com, hoken-square, nttif, Yahoo! Insurance).
   - The agent visits individual direct-sales insurer portals (Mitsui Direct, SBI, Sony, AXA Direct, Zurich, Saison, e-Design, Rakuten), dynamically excluding the policyholder's current insurer.
   - For each candidate insurer, gathers: estimated premiums, coverage limits, roadside scope, unique benefits, customer ratings, and quote page URLs.
5. The agent selects the **top 3 most competitive alternatives** based on value, coverage parity, and service differentiation.
6. The agent produces a deep comparison analysis for each competitor (coverage parity, premium analysis, roadside comparison, unique advantages/disadvantages, switching logistics).
7. The agent writes:
   - `reports/Market_Comparison_Report_EN.md`
   - `reports/Market_Comparison_Report_JA.md`

**Validation Checkpoint:**
- [ ] `temp/market-search-profile.json` exists and matches the baseline contract profile.
- [ ] Exactly 3 competitors are selected and analyzed.
- [ ] All premiums are labeled as **estimates** with the disclaimer present.
- [ ] Master comparison table includes all 4 insurers side-by-side.
- [ ] Each competitor has: coverage parity table, premium analysis, roadside comparison, advantages, disadvantages, and official links.
- [ ] Switching action plan includes NCD portability, mid-term cancellation warning, and step-by-step process.
- [ ] Both EN and JA reports are generated.

**Error Paths:**

| Scenario | Action |
|---|---|
| `temp/baseline-contract.json` does not exist | **STOP.** Instruct the user to run the standard audit pipeline first. |
| Web search returns no pricing data for an insurer | Replace with the next-best candidate. State the data gap clearly. |
| Fewer than 3 viable competitors found | Report as many as available. Explain why others were excluded. |
| Comparison portal data conflicts with insurer portal data | Prefer insurer official data. Note the discrepancy. |

---

## Workflow Diagram

```
Phase 0                Phase 1                    Phase 2                         Phase 3              Phase 4
┌──────────┐    ┌─────────────────┐    ┌──────────────────────────────┐    ┌──────────────┐    ┌──────────────┐
│Pre-Flight │───▶│Contract Anchor  │───▶│  Date Match → Clause Tracer  │───▶│  Synthesizer  │───▶│ Human Review  │
│Validation │    │                 │    │                              │    │              │    │              │
│           │    │ uploads/        │    │ baseline.json                │    │ baseline +   │    │ Coverage     │
│ Check all │    │ contracts/*.pdf │    │     ↓                        │    │ traced.md    │    │ Reality      │
│ files     │    │     ↓           │    │ Scan booklets → date match   │    │     ↓        │    │ Check.md     │
│ exist     │    │ baseline-       │    │ Scan roadside → date match   │    │ Coverage     │    │     ↓        │
│           │    │ contract.json   │    │     ↓                        │    │ Reality      │    │ Feedback     │
│           │    │                 │    │ Read matched docs            │    │ Check.md     │    │ Loop         │
│           │    │                 │    │     ↓                        │    │              │    │              │
│           │    │                 │    │ traced-clauses.md            │    │              │    │              │
└──────────┘    └─────────────────┘    └──────────────────────────────┘    └──────────────┘    └──────────────┘
     │                  │                          │                             │
   STOP if            STOP if                    STOP if                       WARN if
   files missing      no inception date          no booklet matches            items unmapped


Phase 5 (Optional / On-Demand — triggered independently)
┌──────────────────────────────────────────────────────────────────────────────────┐
│  Market Analyst                                                                  │
│                                                                                  │
│  baseline-contract.json + audit report                                           │
│      ↓                                                                           │
│  Build search profile → market-search-profile.json                               │
│      ↓                                                                           │
│  Web search: comparison portals + insurer sites                                  │
│      ↓                                                                           │
│  Select top 3 competitors                                                        │
│      ↓                                                                           │
│  Deep comparison analysis (coverage, premium, roadside, switching)               │
│      ↓                                                                           │
│  Market_Comparison_Report_EN.md + Market_Comparison_Report_JA.md                 │
└──────────────────────────────────────────────────────────────────────────────────┘
     │
   STOP if
   baseline missing
```

---

## File Flow Summary

| File | Created By | Consumed By | Location |
|---|---|---|---|
| Contract PDF | User (pre-uploaded) | Agent 1 | `uploads/contracts/` |
| Policy Booklet PDFs | User (pre-uploaded) | Agent 2 | `uploads/policy-booklets/` |
| Roadside Terms PDFs | User (pre-uploaded) | Agent 2 | `uploads/roadside-terms/` |
| `baseline-contract.json` | Agent 1 | Agent 2, Agent 3, Agent 4 | `temp/` |
| `traced-clauses.md` | Agent 2 | Agent 3 | `temp/` |
| `market-search-profile.json` | Agent 4 | (internal reference) | `temp/` |
| `Policy_Coverage_Audit_Report_EN.md` | Agent 3 | User, Agent 4 | `reports/` |
| `Policy_Coverage_Audit_Report_JA.md` | Agent 3 | User | `reports/` |
| `Market_Comparison_Report_EN.md` | Agent 4 | User | `reports/` |
| `Market_Comparison_Report_JA.md` | Agent 4 | User | `reports/` |