# Weekly Security Intelligence Report

Pull fresh data from your configured MCPs (vulnerability management, GRC, IR), compute derived metrics, and inject a single data block into the dashboard template.

## Prerequisites

- Dashboard template at `~/Downloads/security-posture-report.html` (with the `===REPORT_DATA_START===` / `===REPORT_DATA_END===` markers intact)
- Historical KPIs at `~/.claude/data/security-metrics-history.json`
- CWE mapping at `~/.claude/data/cwe-risk-mapping.json`
- IR mapping at `~/.claude/data/ir-risk-mapping.json`

## Template contract

The dashboard template has two `<script>` blocks:
- `<script id="report-data">` — data between `===REPORT_DATA_START===` and `===REPORT_DATA_END===` markers. **This is the only part you replace.**
- `<script>` — rendering functions. **Never touch this.**

Do NOT create a new HTML file. Do NOT run `/rewrite-dashboard`. Just read the template, replace the data block, save it back.

## Source assumptions

The skill is written against three categories of source, each accessed via an MCP server. The reference implementation uses common tools, but the pattern works with anything that exposes equivalent data:

| Category | What it provides | Reference impl example |
|---|---|---|
| Vuln management | findings (severity, status, age, mitigation, team, product), teams, products | DefectDojo |
| GRC / risk register | risks, controls, frameworks, test results | Vanta |
| Incident / IR ticketing | IR projects, incident outcomes | Linear |

When porting to your environment, replace the `MCP_VULN`, `MCP_GRC`, `MCP_IR` prefixes below with the real MCP tool names and adjust field shapes to match.

## STEP 1: Collect all data

**Error handling**: If any MCP call fails, retry once. If still fails, warn user and use most recent value from `security-metrics-history.json`. Flag the week as stale rather than rendering zeros.

### Vulnerability source — run in parallel batches

**Batch 1 — org structure + current 90d counts per severity:**
```
MCP_VULN:get_teams(limit: 50)
MCP_VULN:get_products(limit: 100)
MCP_VULN:get_findings(date_range: "90d", duplicate: false, limit: 1, severity: "Critical")
MCP_VULN:get_findings(date_range: "90d", duplicate: false, limit: 1, severity: "High")
MCP_VULN:get_findings(date_range: "90d", duplicate: false, limit: 1, severity: "Medium")
MCP_VULN:get_findings(date_range: "90d", duplicate: false, limit: 1, severity: "Low")
MCP_VULN:get_findings(date_range: "90d", duplicate: false, limit: 1, severity: "Info")
MCP_VULN:get_findings(date_range: "90d", duplicate: false, limit: 1)
```

**Batch 2 — year totals + active + mitigated + status:**
```
MCP_VULN:get_findings(date_range: "1y", duplicate: false, limit: 1, severity: "Critical")
MCP_VULN:get_findings(date_range: "1y", duplicate: false, limit: 1, severity: "High")
MCP_VULN:get_findings(date_range: "1y", duplicate: false, limit: 1, severity: "Medium")
MCP_VULN:get_findings(active: true, duplicate: false, limit: 1, severity: "Critical")
MCP_VULN:get_findings(active: true, duplicate: false, limit: 1, severity: "High")
MCP_VULN:get_findings(active: true, duplicate: false, limit: 1, severity: "Medium")
MCP_VULN:get_findings(active: true, duplicate: false, limit: 100, severity: "Critical")  # full list for sample modal
MCP_VULN:get_findings(active: true, duplicate: false, limit: 100, severity: "High")
MCP_VULN:get_findings(date_range: "90d", duplicate: false, is_mitigated: true, limit: 30, severity: "Critical")  # for MTTR
MCP_VULN:get_findings(date_range: "90d", duplicate: false, is_mitigated: true, limit: 30, severity: "High")
MCP_VULN:get_findings(duplicate: false, limit: 1, status: "Risk Accepted")
MCP_VULN:get_findings(duplicate: false, limit: 1, status: "Verified")
MCP_VULN:get_findings(duplicate: false, limit: 1, status: "Out of Scope")
```

**Batch 3 — per-team (one call per `team_id × severity`):**

Configure your tracked team IDs in the variable below. For each team, fetch active counts by severity:
```
TEAM_IDS = [<team-id-1>, <team-id-2>, ...]   # populate from your vuln source

for team_id in TEAM_IDS:
    MCP_VULN:get_findings(active: true, duplicate: false, limit: 1, severity: "Critical", team_id: <team_id>)
    MCP_VULN:get_findings(active: true, duplicate: false, limit: 1, severity: "High",     team_id: <team_id>)
    MCP_VULN:get_findings(active: true, duplicate: false, limit: 1, severity: "Medium",   team_id: <team_id>)
```

### GRC source — run in parallel
```
MCP_GRC:risks(pageSize: 100)           — paginate until done
MCP_GRC:controls(pageSize: 100)        — paginate until done
MCP_GRC:frameworks(pageSize: 10)
MCP_GRC:list_framework_controls(frameworkId: "<framework-a-id>", pageSize: 100)
MCP_GRC:list_framework_controls(frameworkId: "<framework-b-id>", pageSize: 100)
```

Configure your tracked framework IDs (e.g. SOC 2, ISO 27001, HIPAA, PCI) according to what your GRC tool exposes.

### IR source
```
MCP_IR:list_projects(team: "<security-ir-team>")
```

Or read from the local `ir-risk-mapping.json` if your IR data is stored offline.

## STEP 2: Compute metrics

1. **Trend deltas** — `prior_90d = year - current_90d`, `delta = ((current - prior) / prior) × 100`
2. **MTTR** — average days from `date` to `mitigated` per severity
3. **SLA compliance** — % within Critical ≤ 7d, High ≤ 30d, Medium ≤ 90d (adjust thresholds to your policy)
4. **Triage rate** — `(total_crit - active_crit) / total_crit × 100`
5. **Risk score** (8-factor weighted composite):
   ```
   crit_triage_rate    × 0.20
   sca_coverage        × 0.10    (target floor, e.g. 95%)
   sast_coverage       × 0.10    (target floor, e.g. 85%)
   high_sla_compliance × 0.15
   container_coverage  × 0.08
   active_crit_penalty × 0.12    (100 - 5 × active_crits)
   ctrl_test_pass_avg  × 0.15
   risk_pressure       × 0.10    (100 - crit_tier × 8 - unmitigated × 3)
   ```
   Grade: 90+ A, 80–89 B+, 70–79 B, 60–69 C+, <60 C. Tune weights to your environment.
6. **GRC derived** — per-risk inherent / residual / tier / reduction, framework completion / test / doc %
7. **Cross-source** — auto-map findings → risks via CWE (read `cwe-risk-mapping.json`), compute confidence per risk, identify contradicted controls, build gap proofs
8. **Attack paths** — chain findings by CWE adjacency into kill chains; estimate cost range per path
9. **SLA predictions** — per active finding, compute age vs SLA window vs predicted MTTR
10. **Deep dive** — auto-select team with most active Highs for the spotlight section
11. **Historical deltas** — read `security-metrics-history.json`, compute week-over-week changes for sparklines

## STEP 3: Build the report-data block

Construct the full `<script id="report-data">` content containing ALL data objects:

| Object | Purpose |
|---|---|
| `META` | dates, KPI counts, scores, framework stats, cross-source counts |
| `TM` | benchmark data per team |
| `FD` | sample findings for modals (per severity) |
| `TEAM_DATA` | per-team scores, MTTR, SLA |
| `TD` | per-team chart data (donut, trends, burndown, CWE, tools, funnel, radar) |
| `RECS` | per-team recommendations (all + each team) |
| `VR` | full risk array with computed fields |
| `RFE_MAP` | risk-to-finding evidence (auto-derived from CWE mapping) |
| `CV_MAP` | control validation mappings |
| `GAP_PROOFS` | corroborated compliance gaps |
| `OWNER_MAP` | team → risk owner mapping |
| `ATTACK_PATHS` | kill chains from finding data |
| `SLA_FINDINGS` | active findings with SLA projections |
| `BLIND_SPOTS` | areas with no scan coverage |
| `PERSONNEL_RISK` | per-person risk concentration |
| `THREAT_INTEL` | CVE enrichment (EPSS, exploit status) |
| `EVIDENCE_CHAINS` | per-framework full chains |
| `SHARED_DEPS` | cross-product vulnerable packages |
| `HISTORY` | sparkline data from metrics history |
| `COST_EST` | per-attack-path financial impact ranges |

## STEP 4: Inject into template

1. Read `~/Downloads/security-posture-report.html`
2. **One `Edit`**: replace everything between `===REPORT_DATA_START===` and `===REPORT_DATA_END===` with the generated data block
3. Save back to the same file

## STEP 5: Validate

Before publishing:
1. File exists and is above a minimum size threshold (e.g. > 50 KB)
2. `META.kpi.total > 0`
3. `VR.length > 0`
4. Both `<script>` blocks pass `node --check` (extract and test)
5. If any check fails, warn user, do NOT publish

## STEP 6: Persist metrics

Append this week's key metrics to `~/.claude/data/security-metrics-history.json`. Keep the rolling buffer bounded (e.g. 26 weeks).

## Critical rules

- **Never show False Positive data**
- **One `Edit` to inject data** — replace between markers only
- **All data from live MCP calls** — flag stale if a call fails
- **Validate before saving**
- **Persist metrics for trending**
- **Sections are consolidated by the renderer** — produce one data object per section, not one per data point
