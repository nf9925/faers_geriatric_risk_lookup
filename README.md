# Geriatric Medication Risk Lookup

> Identifying medications with the steepest age-driven rises in hospitalisation and death rates, using one quarter of FDA adverse event data.

---

## Motivation

Family caregivers of ageing parents face a quiet but urgent question. Which medications on the bedside pill organizer carry an outsized risk for an older body, and which are routine? Online searches return either alarmist blog posts or dense pharmacology papers, leaving the caregiver no closer to an answer before the next clinical appointment.

A common scenario illustrates the gap. Cancer patients and other older adults are routinely prescribed painkillers, and many widely used painkillers carry real risks of kidney damage that escalate with age. Caregivers reaching for a bottle to ease a parent's pain rarely know which painkiller is safer in that specific situation. Built after personal experience navigating exactly that kind of uncertainty, the project sets out to close part of the gap by surfacing the drugs most strongly associated with worse outcomes in older adults.

## The Problem

Adult children and informal caregivers of older patients lack a quick, evidence-based way to identify which prescribed medications carry the highest age-driven risk of hospitalization or death. Published literature is fragmented across single-drug case series, and general-purpose health sites do not stratify safety information by patient age. Caregivers walk into appointments underprepared to ask which medications deserve a closer look in a 75 or 85-year-old patient.

## Stakeholders

Three audiences benefit from the same analysis, ranked by how directly the project speaks to each:

1. **Family caregivers**, the primary audience the project is built around
2. **Geriatricians and primary care physicians**, who can use the same data to drive deprescribing conversations
3. **Hospital pharmacists** doing medication reconciliation at discharge

## Data Source

Data come from the U.S. Food and Drug Administration Adverse Event Reporting System (FAERS), now rebranded as the Adverse Event Monitoring System.

The scope of this analysis is 2026 Q1, a single quarterly extract containing 397,000 reports covering 2,424 distinct drugs after cleaning. Working from a single quarter carries real statistical limits, addressed in the limitations section below.

Four FAERS tables were used:

| Table | Contents |
|---|---|
| `DEMO26Q1` | Demographic information per report |
| `DRUG26Q1` | Drugs named in each report |
| `REAC26Q1` | Reactions reported (reserved for v2) |
| `OUTC26Q1` | Serious outcomes including hospitalization and death |

## Exploratory Data Analysis

Before running the analytical pipeline, a series of exploratory checks were run against the raw data to understand its shape and quirks. Row counts across the four FAERS tables came in at 397,224 for DEMO, 1,703,210 for DRUG, 1,330,675 for REAC, and 291,580 for OUTC. Reports carry multiple drugs, reactions, and outcomes, so the table sizes differ by design.

Several findings from this phase shape the interpretation of everything downstream. Age data is missing in 38 percent of reports, so the age-stratified analysis runs on roughly 62 percent of the raw pool. Among reports with valid age, the median lands at 59 years, which confirms that FAERS captures the middle-aged and older population that the project cares about. Sex distribution shows 49.9 percent female, 31.1 percent male, and 19 percent missing, with a sex by outcome cross tabulation revealing that male reports carry a substantially higher death share than female reports, roughly 11 percent versus 6 percent. Each of these findings feeds directly into the limitations section below.

Drug and reaction distributions also proved informative. Dupilumab sat at the top of the reported drug list with 32,991 reports, followed by tirzepatide and abaloparatide, meaning any leaderboard is inevitably shaped by which drugs draw the most attention. Reaction terms were led by off-label use, drug ineffectiveness, and nausea, a mix that reflects both patient experience and drug labelling gaps. Full outputs for every EDA check live inside the SQL script under the labelled EDA section.

## Method

Each report was filtered to primary suspect drug rows only, keeping only drugs suspected as the cause of the adverse event rather than concomitant medications. Age was cleaned to retain only reports recorded in years, with adults split into five brackets: 18 to 44, 45 to 64, 65 to 74, 75 to 84, and 85 plus. Outcome flags were built per report, marking whether the report ended in hospitalization, death, or both.

For each drug, the rate of each outcome was computed in the youngest and oldest brackets, then expressed as a ratio called the age penalty. A 95 percent confidence interval was constructed for every ratio using the log normal approximation, the standard formula taught in epidemiology textbooks for the confidence interval of a relative risk.

Drugs were ranked by the lower bound of the confidence interval rather than the point estimate. Ranking by the lower bound rewards drugs where the elevation can be defended with confidence, while still keeping uncertain drugs visible in a secondary section labelled "high signal, low confidence." No drug was dropped from the analysis based on small sample size alone, a deliberate methodological choice to honour the principle that statistical insignificance does not mean an effect is absent.

A sensitivity analysis was performed on the death leaderboard with common oncology drugs excluded. Cancer drugs concentrate at the top of any death analysis because cancer patients die more often, and removing them isolates the signal driven by drug pharmacology rather than the underlying disease.

## Key Findings

Hospitalization signals aligned closely with established geriatric concerns, surfacing clusters that map onto drug categories with decades of published warnings about elderly safety. The alignment lends construct validity to the method, since a credible signal detection approach should rediscover risks the field already takes seriously.

Death signals after oncology exclusion brought a different mix to the surface, foregrounding drug groups whose age-driven risk is less commonly highlighted in everyday caregiver-facing material.

Ranked leaderboards for hospitalization and death, the oncology sensitivity analysis, and full age trajectories for every drug are available as CSV files in the `exports` folder. Findings are reported as signals worth investigating, never as conclusions, and never as a recommendation to avoid any medication.

## Limitations

1. FAERS captures voluntary safety reports, not patient outcomes from a defined population. Numbers in this analysis are report rates, never true incidence in patients taking each drug. The story is always comparative.
2. Single-quarter limits statistical power. Adding older and newer quarters would tighten every confidence interval and enable a temporal trend analysis.
3. The oncology exclusion list was curated by hand and is not exhaustive. A more robust v2 would use a structured drug classification system such as the ATC code hierarchy.
4. The age penalty is computed as a two-point ratio between the youngest and oldest brackets. A regression across all five brackets would extract more information from the data.
5. Reporting bias varies by drug. Newer, heavily marketed drugs attract more reports per actual user than older drugs, so cross-drug comparisons should be read with that asymmetry in mind.
6. Age data is missing in 38 percent of the raw report pool, so the age-stratified analysis runs on roughly 62 percent of the data. Reports without age were excluded rather than imputed.
7. Male reports carry a roughly 11 percent death share against 6 percent for female reports. The pooled analysis does not separate the two sexes, so a v2 stratified by sex would isolate whether age penalties differ between men and women.
8. A small number of high-volume drugs, led by dupilumab at 32,991 reports, dominate the leaderboards. Report volume drives statistical confidence, so drugs with fewer reports may carry real signals that stay hidden under the CI ranking approach.

## Disclaimer

The project is intended to support better conversations with clinicians, not to replace medical advice. Findings should be brought to a qualified healthcare professional before any change to a medication regimen. The author is not a clinician, and no output from the project constitutes a clinical recommendation.

## Tech Stack

- Microsoft SQL Server 2022 (T-SQL)
- SQL Server Management Studio (SSMS)
- Power BI Desktop (caregiver dashboard in active development, including drug lookup slicer, age trajectory line chart, painkiller comparison visuals, and top drugs by death rate)

## How to Reproduce

1. Download the FAERS 2026 Q1 ASCII extract from the FDA portal at https://fis.fda.gov/extensions/FPD-QDE-FAERS/FPD-QDE-FAERS.html
2. Load `DEMO26Q1.txt`, `DRUG26Q1.txt`, `REAC26Q1.txt`, and `OUTC26Q1.txt` into a SQL Server database. Note that the field separator is the dollar sign and not the pipe, and that the row terminator differs by file
3. Run `sql/geriatric_drug_aging_gradient.sql` end-to-end
. The script uses temp tables so intermediate steps can be inspected without rerunning earlier sections
4. Export the leaderboards and trajectory data to CSV for the Power BI dashboard

## Repository Structure

```
.
├── README.md
├── sql/
│   └── geriatric_drug_aging_gradient.sql
├── exports/
│   ├── full age trajectory
│   ├── death leaderboard without oncology[death leaderboard without oncology.csv](https://github.com/user-attachments/files/29683483/death.leaderboard.without.oncology.csv)
[full age trajectory.csv](https://github.com/user-attachments/files/29683472/full.age.trajectory.csv)

└── powerbi/
    └── (Power BI dashboard, in progress)
```

## Future Work

- Brand name to active ingredient lookup, so caregivers can search "Ozempic" and the dashboard returns semaglutide automatically
- Multi-quarter analysis across 2021 through 2026 to test whether age penalties scaled as new drug classes entered widespread use
- Sex stratified analysis to isolate whether the male mortality skew observed in EDA reflects a biological difference or a reporting difference
- Companion dashboards for geriatricians (deprescribing conversation support) and hospital pharmacists (medication reconciliation)

## Acknowledgements

Thanks to my parents, whose care prompted this project.
