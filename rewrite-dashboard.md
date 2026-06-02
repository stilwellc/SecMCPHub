# Rewrite Security Dashboard

Rebuild the dashboard renderer from scratch. **Only run this when intentionally changing the UI** — adding a section, restyling, restructuring.

## Input

Read this file completely:
- `~/Downloads/security-posture-report.html` — the current dashboard. The data block (`<script id="report-data">` between `===REPORT_DATA_START===` and `===REPORT_DATA_END===` markers) must be preserved verbatim. The renderer below it is what you're rewriting.

## Task

Rewrite the file from scratch as ONE cohesive, production-ready dashboard. Do NOT patch — write it fresh as one intentional piece. Save back to `~/Downloads/security-posture-report.html` (overwrite).

## Architecture (the contract)

Single self-contained HTML file with two `<script>` blocks:
- `<script id="report-data">` — ALL data objects between `===REPORT_DATA_START===` and `===REPORT_DATA_END===` markers. The `/security-report` skill replaces this block weekly. **Preserve the existing data block exactly when rewriting.**
- `<script>` — ALL rendering functions. Never modified by the weekly skill.

External deps: Chart.js (CDN with SRI `integrity="<pinned-sha384-hash>"` + `crossorigin="anonymous"`) + a web-fonts CDN. Pin a specific Chart.js version and its SRI hash here when you ship.

## Design Philosophy

This is not a report with N equal sections. It's a **decision-making tool with a clear visual hierarchy**:

### Layer 1 — The Answer (no scroll needed)
The reader opens this and sees the answer immediately:
- **Risk grade ring** — THE hero element. Large, centered, unmissable. Grade + score + week-over-week delta.
- **"Things that changed this week"** — data-derived, not generic AI text. What got worse, what got better, what's new. Rendered as compact cards with icons.
- **KPI ticker** — single row of headline metrics. Every number has a ↑↓ delta from last week.

### Layer 2 — The Story (first scroll)
How an attacker exploits weaknesses, and what it costs:
- **Attack paths** — kill chains as visual flows. Each node shows real finding count. Cost-range estimates. This is what makes a security leader act.
- **Top risks table** — risks with finding evidence (corroborated risks). Residual score + finding count + confidence signal. Click to drill.

### Layer 3 — The Evidence (scroll deeper)
Supporting data organized as tabs/expandable sections, NOT a flat list:
- **Operations tab**: Triage funnel, MTTR/SLA with breach predictions, severity trends, CWE taxonomy
- **Teams tab**: Unified heatmap (vuln + GRC columns), benchmark, leaderboard, personnel risk, deep dive
- **Scan tab**: Tool coverage, blind spots with what-if toggles, shared deps
- **Risk tab**: Full risk register with heatmaps, filterable table, evidence map, gap proofs
- **Compliance tab**: Framework scorecards + countdown, contradicted controls prominently shown, evidence chains expandable on demand (NOT shown by default — too dense)
- **Strategy tab**: Maturity radar, recommendations, scenario simulator

### Layer 4 — The Details (on demand only)
- **Active findings table** — expandable rows with threat intel inline
- **Full risk table** — all risks with filters
- **Evidence chains** — expand per control

## Section Structure (reordered by importance)

### 1. Command Center (hero — no scroll)
The single most important view. Takes up the full viewport on load:
- **Giant risk score ring** (center) — grade letter large, score below, week-over-week delta
- **"This week" cards** (auto-generated from data — compare current vs history, find biggest changes):
  - 🔴 something that got worse
  - 🟢 something that got better
  - 🆕 something newly notable (e.g. a threshold crossing)
- **KPI strip** (bottom of hero): Total | Active Crits | Active Highs | Framework A % | Framework B % | Gaps | Triage Rate. Each with sparkline + ↑↓ delta
- **Composite-score factor breakdown** as small horizontal bars below the ring

### 2. Attack Paths (first scroll — the "why should I care")
N kill chains visualized as node→animated-arrow→node flows. Each path is data-driven from `ATTACK_PATHS`:
- Each node: label, detail, finding count badge (green = theoretical, red = corroborated by findings)
- Each path: links to the GRC risk it corroborates + cost-range estimate from `COST_EST`

Below: "Top corroborated risks" mini-table — risks where findings prove the risk is real.

### 3. Triage & SLA
- Triage funnel: total → active → risk accepted → verified → OOS → mitigated
- MTTR tiles: per-severity median days + SLA %
- SLA breach predictions: per active finding — age, window, remaining, predicted, status badge

### 4. Severity & CWE
- Severity donut + multi-month trend line
- CWE taxonomy horizontal bar (clickable)
- Combined in one section, two-column layout

### 5. Teams
Everything team-related in one place:
- Unified heatmap: Team | Crit | High | Med | Total | Δ | Risks | Unmit | Gaps
- Benchmark: two dropdowns + overlay chart
- Leaderboard: ranked by composite score, medals, badges
- Deep dive spotlight: auto for team with most active Highs
- Personnel risk: per-person cards with bus factor

### 6. Risk Trajectory
Tab toggle: Forecast (multiple scenarios) | Burndown (multi-week)

### 7. Scan Coverage & Blind Spots
- Tool volume bar + engagement donut
- Blind spots with what-if toggles
- Shared vulnerable dependencies

### 8. Active Findings
- Critical/High table with expandable rows
- Threat intel inline: EPSS, exploit availability, wild status
- Risk links on each finding
- Shared deps callout

### 9. Maturity
- Capability progress bars + radar chart (current vs target)

### 10. Recommendations
- Per-team P1/P2/P3 with ticket buttons (rebuilds on team filter)

### 11. Risk Register
- 5×5 heatmaps (inherent/residual toggle, click cells for modal)
- Summary cards: total, top-tier count, avg residual, unmitigated count
- Filterable table with expandable rows
- Category/treatment/owner charts
- Risk-to-finding evidence integrated: each risk shows finding count + confidence
- Corroborated gaps sub-table: gaps both sources confirm

### 12. Compliance
- Framework scorecards: progress bars, health grade
- Countdown timers: days to next audit + readiness %
- **Contradicted controls prominently shown** — these are the top-level view
- Control validation table: validated / contradicted / unverifiable counts — below the scorecards
- Evidence chains: per-framework tab toggle — **collapsed by default**, expand on click. Too dense to show inline.
- Domain distribution chart
- Unowned controls warning

## Interactive Features

- **Sticky sidebar TOC** with IntersectionObserver scroll spy. Groups: Command Center / Threat / Operations / Strategy / Risk & Compliance
- **Sticky KPI ticker** — appears on scroll past hero
- **Team filter** — filters ALL sections
- **Dark/light theme toggle**
- **Global search (Cmd+K)** — findings, risks, controls, teams, recs
- **Knowledge graph** — double-click any entity → modal with all connections
- **Cross-section linking** — interconnect banners on key sections
- **Drill-down modals** on KPIs, heatmap cells, chart bars
- **Expandable rows** on all tables
- **Animated counters** on scroll
- **Scenario simulator** — "fix N findings of class X" slider → projected score
- **"Create ticket" buttons** (configurable — wire to your ticketing system or simulate)
- **Copy one-pager** + **Markdown export** + **Print/PDF**
- **AI briefing** (optional — wire to your LLM provider with fallback)
- **Sparklines** on all KPI cards with ↑↓ deltas
- **@media print** styles

## Data Objects (read these from the existing `<script id="report-data">` block)

Every section is rendered from one or more of these objects. Copy them verbatim from the existing data block when rewriting the renderer:

```
META, TM, FD, TEAM_DATA, TD, RECS, VR, RFE_MAP, CV_MAP, GAP_PROOFS,
OWNER_MAP, ATTACK_PATHS, SLA_FINDINGS, BLIND_SPOTS, PERSONNEL_RISK,
THREAT_INTEL, EVIDENCE_CHAINS, SHARED_DEPS, HISTORY, COST_EST
```

## Critical Rules

- **Never show False Positive data**
- **Every entity links to related entities** via knowledge graph
- **Vuln + GRC + IR sources are woven together** — not separate silos
- **Typography**: a display serif, a UI sans, and a monospace (pick consistent web fonts)
- **Dark theme default** + full light theme via CSS vars
- **Compressed CSS** with short utility class names (`.K`, `.KG`, `.CC`, `.DT`, `.HM`, etc.) — keep file size manageable
- **Print styles**: light colors, hide interactive controls, no mid-section breaks
- **One file, zero build tools**
- **SRI on Chart.js CDN**
- **Hero section fills viewport** — grade visible without scrolling
- **Attack paths are section 2**, not buried at the bottom
- **Evidence chains collapsed by default** — expand on demand
- **Every number has a ↑↓ delta** where history data exists
- **All labels come from data, not hardcoded strings** — framework names, team names, risk IDs, etc. are properties of the data block so the renderer is environment-agnostic
