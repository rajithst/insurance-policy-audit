---
name: insurance-analysis
description: >-
  Run the full insurance contract analysis workflow, market comparison
  benchmarking analysis, or the policy options catalog report. Use this skill
  when the user asks to analyze their insurance contract, run the insurance
  workflow, check their coverage, map their contract to policy booklets,
  compare their insurance against competitors, or explore all available policy
  options and riders. Triggers on: "run insurance analysis", "analyze my contract",
  "check my coverage", "run the workflow", "insurance reality check",
  "compare insurers", "market analysis", "find cheaper insurance",
  "benchmark my premium", "competitive quotes", "market comparison",
  "available options", "options report", "coverage catalog", "what options are available",
  "available riders", or any request to process the documents in the uploads folder.
---

# Insurance Contract Analysis Workflow

Execute the full insurance analysis pipeline as defined in the project's
[workflow.md](../../workflow.md) and [AGENTS.md](../../AGENTS.md).

Read both files completely before starting. They contain all the detailed instructions,
guardrails, and output formats you need to follow.

## Quick Summary

This workflow analyzes a Japanese auto insurance contract (保険証券) by:
1. Extracting structured data from the contract PDF
2. Matching to the correct versioned policy booklet and roadside terms by inception date
3. Tracing every coverage item to its legal definition, exclusions, and limits
4. Producing a human-readable report with gaps, benefits, and recommendations
5. *(Optional, on-demand)* Benchmarking the policy against 3 competitive insurers
6. *(Optional, on-demand)* Generating a comprehensive catalog of all available coverages, riders (特約 1–32), and roadside perks

## Execution Steps

### Step 1: Pre-Flight (Phase 0)

1. Verify `uploads/contracts/` has a PDF file.
2. Verify `uploads/policy-booklets/` has PDF files.
3. Verify `uploads/roadside-terms/` has PDF files.
4. Create `temp/` and `reports/` directories if they don't exist.
5. If any critical file is missing, STOP and tell the user what to add.

### Step 2: Run Agent 1 — The Contract Anchor (Phase 1)

1. Read the contract PDF from `uploads/contracts/`.
2. Follow all instructions for **Agent 1** in [AGENTS.md](../../AGENTS.md).
3. Extract inception date, vehicle details, coverages, endorsements, and policyholder info.
4. Write the output to `temp/baseline-contract.json`.
5. **Validate:** Confirm the JSON is well-formed and contains `policy.inception_date`.
6. Display any `extraction_warnings` to the user before proceeding.

### Step 3: Run Agent 2 — The Clause Tracer (Phase 2)

1. Read `temp/baseline-contract.json`.
2. Follow all instructions for **Agent 2** in [AGENTS.md](../../AGENTS.md).
3. **Date-match** the inception date against all booklet and roadside filenames:
   - Parse YYYYMMDD sequences from filenames.
   - Select the file whose date range contains the inception date.
   - Output the Document Selection Audit log.
4. Read the matched documents in full, tracking exact PDF pages, chapters, and articles.
5. Trace every coverage and endorsement to its legal definition, limits, and exclusions.
6. **Provider Official Web Research:**
   - Execute web searches on the underwriting insurer's official website (`<policy.insurer_domain>`).
   - Extract official links for coverage guides, roadside assistance, accident reporting, and My Page procedures.
7. Write the output to `temp/traced-clauses.md`.
8. **Validate:** Confirm every baseline coverage item appears with exact PDF page citations and official web links.

### Step 4: Run Agent 3 — The Gap & Benefit Synthesizer (Phase 3)

1. Read `temp/baseline-contract.json` and `temp/traced-clauses.md`.
2. Follow all instructions for **Agent 3** in [AGENTS.md](../../AGENTS.md).
3. Generate two comprehensive audit reports (one in English and one in Japanese), each containing:
   - **Granular Legal Citations:** Exact PDF filename, page number(s), chapter/section, and article number for every item.
   - **Official Provider Links:** Clickable, verified links to official guides on the insurer's website (`<policy.insurer_domain>`).
   - The 6 core sections:
      - Coverage Reality Table
      - Danger Zone (🔴🟡🟢)
      - Hidden Usable Benefits (including **Contract Entitlement Proof**, passenger coverage scope, and **4-column structured tables** for both 24/7 Roadside Assistance and Stranded Travel Protection with eligibility, coverage, and exact citations, plus the mandatory first-call reimbursement rule)
      - Powertrain-Specific Analysis (if applicable)
      - Recommendations
      - Quick Reference Card
4. Write the output to:
   - `reports/Policy_Coverage_Audit_Report_EN.md` (Full English audit report)
   - `reports/Policy_Coverage_Audit_Report_JA.md` (Full Japanese audit report)

### Step 5: Present Results (Phase 4)

1. Tell the user both reports are ready:
   - English: `reports/Policy_Coverage_Audit_Report_EN.md`
   - Japanese: `reports/Policy_Coverage_Audit_Report_JA.md`
2. Highlight the top 3 most critical findings (🔴 items).
3. List any extraction warnings or unmapped items.
4. Ask if they want deeper analysis on any area.

### Step 6: Run Agent 4 — The Market Analyst (Phase 5, Optional / On-Demand)

> **This step runs ONLY when the user explicitly requests market comparison.**
> Trigger phrases: "compare insurers", "market analysis", "find cheaper insurance",
> "benchmark my premium", "competitive quotes", "market comparison", "shopping around".

**Prerequisites:** `temp/baseline-contract.json` MUST exist. If it doesn't, run Steps 1–4 first.

1. Read `temp/baseline-contract.json` and (if available) `reports/Policy_Coverage_Audit_Report_EN.md`.
2. Follow all instructions for **Agent 4** in [AGENTS.md](../../AGENTS.md).
3. Build a standardized search profile and write to `temp/market-search-profile.json`.
4. Web-search Japanese auto insurance comparison portals and individual insurer sites.
5. Select the top 3 most competitive alternatives (dynamically excluding the policyholder's current provider).
6. Produce deep comparison analysis for each competitor (coverage parity, premium, roadside, switching logistics).
7. Write the output to:
   - `reports/Market_Comparison_Report_EN.md` (English market comparison report)
   - `reports/Market_Comparison_Report_JA.md` (Japanese market comparison report)
8. Present the reports to the user with:
   - The best-value pick and estimated savings range
   - Key differentiators between insurers
   - Direct quote links for each competitor
   - Switching action plan summary

### Step 7: Run Agent 5 — The Options & Catalog Analyst (Phase 6, Optional / On-Demand)

> **This step runs ONLY when the user explicitly requests an options availability report.**
> Trigger phrases: `/insurance-analysis options`, "options report", "what options are available",
> "list all available options", "coverage catalog", "available riders".

1. Confirm the policy booklet exists in `uploads/policy-booklets/`.
2. Follow all instructions for **Agent 5** in [AGENTS.md](../../AGENTS.md).
3. If `temp/baseline-contract.json` exists, load active coverages to compare against the full catalog.
4. Scan the master Special Provisions table (*特約一覧*) in the policy booklet to extract all 32 riders, core coverages, deductible structures, and driver restrictions.
5. Extract roadside assistance allowances and stranded travel support from the roadside terms.
6. Categorize options into:
   - Core Base Coverages (Liability, Injury, Hull General vs Limited)
   - Master Rider Catalog (All 32 Special Provisions: Liability, Injury, Hull, Family/Lifestyle, Driver Scope, Telematics, Payment)
   - Practical Recommendations ("Who Needs This?" personas: families, bicycles, pet owners, new car owners, EV/Hybrid owners)
   - Roadside Assistance & Travel Support Features
   - Deductible Configurations & Premium Discounts
7. Tag each item with:
   - `[CURRENTLY ACTIVE ✅]` (already attached to contract)
   - `[AVAILABLE TO ADD ➕]` (optional add-on)
   - `[AUTOMATICALLY INCLUDED 🔹]` (bundled mandatory clause)
8. Write the output to:
   - `reports/Available_Policy_Options_Report_EN.md` (English policy options catalog)
   - `reports/Available_Policy_Options_Report_JA.md` (Japanese policy options catalog)
9. Present a high-level summary to the user highlighting key add-ons they could consider.

## Error Handling

- **No contract PDF found** → STOP, ask user to add it to `uploads/contracts/`.
- **No matching booklet** → STOP, show the inception date and all available date ranges.
- **Inception date not extractable** → STOP, ask user to provide it manually.
- **Unmapped coverage items** → Continue but flag them prominently in the report.
- **Market analysis requested but no baseline** → STOP, run the standard audit pipeline first.
- **Insufficient competitor data** → Report as many competitors as viable, explain gaps.
- **Options catalog requested without booklet** → STOP, ask user to upload the policy booklet PDF.
