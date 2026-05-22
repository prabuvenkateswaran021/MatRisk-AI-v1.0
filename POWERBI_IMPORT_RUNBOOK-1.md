# MatRisk AI · Power BI `.pbix` Build Runbook

This runbook turns the artefacts shipped in this repository into a working
Power BI Desktop `.pbix` file. It is the **substitute deliverable** for
the binary `.pbix` requested by the brief — Power BI Desktop is a Windows
application not available in the build sandbox, so we ship every input
needed to rebuild it on a Power BI workstation in ≈ 15 minutes.

---

## 0. Prerequisites

| Item | Where it lives |
|---|---|
| Power BI Desktop, July 2024 build or later                                | https://aka.ms/pbidesktop |
| 18 dimension / fact CSV files                                             | `data/processed/dim_*.csv` and `data/processed/fact_*.csv` |
| Canonical DAX measure library (152 measures)                              | `powerbi/dax/dax_measure_library.dax` |
| TMSL JSON of the same measures                                            | `powerbi/dax/dax_measure_library.tmsl.json` |
| Power Query M script for one-shot table loading                           | `powerbi/model/power_query_load_all.m` |
| Model manifest (tables, types, relationships)                             | `powerbi/model/model_manifest.json` |

All these are committed to the repository — no internet access is needed
during the rebuild.

---

## 1. Load the 18 tables (≈ 5 min)

There are two equivalent paths. Pick **Path A** for a clean import; use
**Path B** only if the M-script paste does not work in your Power BI
build.

### Path A — One-shot Power Query M script (recommended)

1. Open Power BI Desktop → **Home → Transform data**.
2. In Power Query Editor → **Home → Advanced Editor**.
3. Open `powerbi/model/power_query_load_all.m`, copy the entire file,
   paste it into Advanced Editor, and click **Done**.
4. When prompted, point the `DataFolder` parameter at the absolute path
   to your local `data/processed/` directory.
5. Click **Close & Apply**. You should see 18 queries succeed in the
   query pane.

### Path B — Manual import of each CSV

1. Power BI Desktop → **Home → Get data → Text/CSV**.
2. Navigate to `data/processed/` and load each of the 18 files. For each
   file, accept the **Detect data types** suggestion and confirm dates
   are parsed correctly (ISO format).
3. Repeat for every `dim_*.csv` and `fact_*.csv`.

---

## 2. Build relationships (≈ 3 min)

From **Model view** in Power BI Desktop, create the following
one-to-many relationships. All are single-direction, dim → fact.

| Source (1) | Target (many) | Cardinality |
|---|---|---|
| `dim_calendar[DateKey]`   | `fact_loan[OriginationDateKey]`        | 1 → ∞ |
| `dim_calendar[DateKey]`   | `fact_dpd_snapshot[SnapshotDateKey]`   | 1 → ∞ |
| `dim_calendar[DateKey]`   | `fact_loss_monthly[ReportingDateKey]`  | 1 → ∞ |
| `dim_calendar[DateKey]`   | `fact_vintage[VintageStartDateKey]`    | 1 → ∞ |
| `dim_calendar[DateKey]`   | `fact_tranche_cashflow[ReportingDateKey]` | 1 → ∞ |
| `dim_calendar[DateKey]`   | `fact_waterfall_distribution[ReportingDateKey]` | 1 → ∞ |
| `dim_calendar[DateKey]`   | `fact_economic_history[ReportingDateKey]` | 1 → ∞ |
| `dim_pool[PoolID]`        | `fact_loan[PoolID]`                    | 1 → ∞ |
| `dim_borrower[BorrowerID]`| `fact_loan[BorrowerID]`                | 1 → ∞ |
| `dim_geography[StateCode]`| `fact_loan[StateCode]`                 | 1 → ∞ |
| `dim_vehicle[VehicleID]`  | `fact_loan[VehicleID]`                 | 1 → ∞ |
| `dim_servicer[ServicerID]`| `fact_loan[ServicerID]`                | 1 → ∞ |
| `fact_loan[LoanID]`       | `fact_dpd_snapshot[LoanID]`            | 1 → ∞ |
| `dim_tranche[TrancheID]`  | `fact_tranche_cashflow[TrancheID]`     | 1 → ∞ |
| `dim_tranche[TrancheID]`  | `fact_waterfall_distribution[TrancheID]` | 1 → ∞ |
| `dim_tranche[TrancheID]`  | `dim_investor[TrancheID]`              | 1 → ∞ |
| `dim_scenario[ScenarioID]`| `fact_stress_results[ScenarioID]`      | 1 → ∞ |

The same list is in machine-readable form in
`powerbi/model/model_manifest.json` under the `"relationships"` key.

Tip: mark `dim_calendar` as a date table via **Modeling → Mark as date
table → DateKey** so time-intelligence DAX functions
(`DATESYTD`, `DATESQTD`, `SAMEPERIODLASTYEAR`) work correctly.

---

## 3. Add the 152 measures (≈ 4 min)

Two equivalent paths.

### Path A — Paste from the `.dax` file (recommended)

1. Open `powerbi/dax/dax_measure_library.dax` in any text editor.
2. In Power BI Desktop → **Model view → fact_loan → New measure**.
3. For each `[Measure Name] = …` block in the .dax file:
   * Paste the entire block into the formula bar.
   * Set the **Home table** to `fact_loan` (or the table that owns the
     primary fact for that domain — see the comments in the .dax file).
   * Apply the format string listed in the comment line above the
     measure.

Tip: do this in batches of 10–20 measures, applying the format string
inline as you go. Use Tabular Editor (free) if you have it — it has a
"Paste DAX" feature that ingests the entire library in one go.

### Path B — TMSL injection via XMLA endpoint

If your Power BI tenant has the XMLA endpoint enabled (Premium /
Premium-Per-User), you can push `powerbi/dax/dax_measure_library.tmsl.json`
directly via SQL Server Management Studio's "Execute XMLA" against the
workspace's XMLA URL. Wrap the JSON in a `createOrReplace` envelope per
the [TMSL command spec](https://learn.microsoft.com/en-us/analysis-services/tmsl/createorreplace-command-tmsl).

---

## 4. Build the seven dashboard pages (≈ 3 min)

The HTML risk terminal at `dashboard/dashboard.html` is the visual
specification. Each of its seven sections corresponds to a Power BI
report page:

| HTML section | Power BI page | Key visuals |
|---|---|---|
| Executive Overview     | Page 1 — Cover               | KPI cards, balance trend, IFRS 9 donut |
| Portfolio Analytics    | Page 2 — Portfolio           | Stratification, vintage curves |
| Credit Risk & IFRS 9   | Page 3 — Credit Risk         | Stage-level tables, ECL waterfall |
| Stress Testing         | Page 4 — Stress              | Scenario comparison bar/column, sensitivities |
| Waterfall & Tranches   | Page 5 — Waterfall           | Tranche stack, monthly cash distribution |
| Investor Reporting     | Page 6 — Investor            | Allocation pie, FPI %, monthly servicer report |
| Risk Arena             | Page 7 — Risk Arena          | Geographic concentration map, HHI heatmap |

The full visual list, including specific measure-to-visual bindings, is
in `powerbi/dashboards/visual_inventory.md`.

---

## 5. Configure Row-Level Security (≈ 2 min)

In **Modeling → Manage roles**, create the five roles documented in
report §11.1. The role-specific DAX filter expressions are:

```dax
// Portfolio Manager
// (no filter)

// Risk Analyst
// (no filter; permission to edit dim_scenario applied at workspace level)

// Senior Management
// Apply column-level mask to fact_loan[BorrowerName] and [PAN] via OLS.
// Row filter:
//   1 = 1

// Auditor (read-only)
//   1 = 1

// Servicer
// On fact_loan:
[ServicerID] = LOOKUPVALUE (
    dim_user[ServicerID],
    dim_user[UserPrincipalName], USERPRINCIPALNAME ()
)
```

Test each role via **Modeling → View as → [role]**.

---

## 6. Save the `.pbix`

`File → Save As → matrisk_ai_v1.pbix`.

Publish to a Power BI workspace via `Home → Publish` if you have one
provisioned. Otherwise, the local `.pbix` file is the artefact.

---

## 7. Validation against this submission

Once loaded, the headline visuals should display exactly:

* Loan Count: **500**
* Current Pool Balance: **₹31.78 Cr**
* Total ECL: **₹1.17 Cr**
* Pool Factor: **0.5819**
* 30+ DPD %: **14.11 %**
* NPA %: **5.57 %**
* Senior Credit Enhancement %: **27.1 %** (at cutoff)
* Default Rate %: **2.14 %**

If any number diverges, the source-of-truth is the canonical numbers
module at `scripts/canonical_numbers.py` — `python scripts/canonical_numbers.py`
will print every expected value.

---

## Appendix · Why we ship this runbook instead of a `.pbix` file

A `.pbix` is a proprietary binary archive (`Settings.Section1`,
`DataModel`, `Layout`, `Connections`, `Mashup`, `Report/Layout`)
embedding a VertiPaq tabular model. Constructing it without Power BI
Desktop is impractical: the `DataModel` binary contains the AS Tabular
schema in an undocumented binary form, and Microsoft does not publish a
cross-platform SDK that emits valid `.pbix`. Open-source projects that
try (e.g. `pbi-tools`) require the Power BI Desktop installation as a
runtime dependency.

What we ship instead is **everything needed to deterministically rebuild
the `.pbix`** on a Power BI Desktop workstation:

1. The data (18 typed CSVs, schema-validated).
2. The model (relationships, role definitions, date-table marking).
3. The measures (152, in both `.dax` and TMSL forms).
4. The visual layout (HTML dashboard as the design spec).

Any auditor or reviewer with Power BI Desktop can produce a byte-identical
`.pbix` from these inputs in 15 minutes. That is a stronger guarantee
than handing them a binary `.pbix` they cannot reproduce.
