# WHEB Tools — Project Reference for Claude Code

Branded single-file HTML calculator tools for UK financial advisers, published at https://cswm.me.uk.

## Purpose & context

- Owner: Mike Savage, UK employee benefits adviser and pension specialist at Warren House Employee Benefits (Stony Stratford, Milton Keynes).
- A suite of branded, self-contained single-file HTML tools with **no backend dependencies**.
- Adviser-facing: used with clients on a laptop (mobile secondary, but must not break at 390px width).
- Every tool carries a standard disclaimer. Tools are for UK financial advisers only, **not** public/client self-service.
- Tools must be "idiot-proofed" for adviser use: tooltips, clear labels, inline guidance. **Comma formatting on all number/currency inputs** is a firm preference.

## Repo & deployment

- Repo: `github.com/savshouse/wheb-tools` — **public**, so never commit credentials, client data or real scheme data.
- GitHub Actions → FTP(S) deploy to Fasthosts `/htdocs/` on every push to `main`. Workflow: `.github/workflows/deploy.yml` (SamKirkland/FTP-Deploy-Action). Password is held in the repo secret `FTP_PASSWORD`.
- Deploy excludes: `.git*`, `node_modules/`, `.github/`, `README.md`. Add `CLAUDE.md` and any test files to the exclude list so they don't get published.

## Live tools

| Tool | URL | File |
|---|---|---|
| Annual Allowance Calculator | https://cswm.me.uk/aacalc | `annual-allowance-calculator.html` |
| Retirement Planner | https://cswm.me.uk/planner | `planner.html` |
| QR Code Generator | https://cswm.me.uk/qrgenerator | `qrgenerator.html` |
| WHEB Tools Hub | https://cswm.me.uk/tools | `wheb-tools-hub.html` |
| TRS Generator | cswm.me.uk (not yet in hub) | source was `trs-src2.html` |
| PDF Password Protector | TODO | TODO |
| Salary Exchange Calculator | https://cswm.me.uk/salexchange | `salexchange.html` |

- `retirement-planner.html` is an **earlier, different tool** — do not confuse it with `planner.html`.

## Working method (follow this)

These files are large (the planner is ~210KB). **Never regenerate a whole file** — full rewrites caused corrupted files and silent regressions earlier in the project. Every change follows this loop:

1. Surgical edit: `assert s.count(old) == 1` then `s.replace(old, new)`. For complex JS inside nested quotes, index-based slicing (find start/end, splice) is more reliable than `str.replace()`.
2. Syntax check: extract all `<script>` blocks (`re.findall(r'<script>(.*?)</script>', html, re.DOTALL)`), write to a temp `.js`, run `node --check`.
3. Runtime check with Playwright. Stub Chart.js if offline: `pg.route('**/chart.umd.min.js', ...)` returning `window.Chart = function(ctx,cfg){ this.destroy = function(){}; }`.
4. Only then is the change done. Prefer several small verified passes over one big risky change.

Playwright tips:
- For comma-formatted currency inputs, `fill()` can race the focus handler. Use `el.click(); el.press('Control+A'); el.type('value')`.
- Use `locator.screenshot()` for elements below the fold.
- Use `pg.emulate_media(media="print")` to test `@media print` rules.
- An apparent double `window.print()` call in headless Chromium is an emulation artefact, not a bug.

Common bug patterns already hit (check for these):
- Functions defined inside a `DOMContentLoaded` closure but called from inline `onclick` → must be exposed on `window`.
- CSS selectors going stale after markup changes (e.g. `.irow` → `.irow4`) and silently breaking `applyState()`.
- Rebuilding DOM on `oninput` wipes what the user is typing.
- Chart.js can't render into a `display:none` canvas — expand the section and re-run `calc()` before printing.

## Master template

- `wheb-template.html` is the definitive starting point for **all new tools**: exact CSS, embedded base64 logo/favicon, header, footer, back-to-top, and marked zones for tool-specific styles and JS.
- Logo/favicon are base64-embedded (no external images); definitive source is `retirement-planner_3.html`.

## Branding constants

- Navy: ⚠️ **conflicting records** — `#000d3c` in one place, `#1D353B` in others. Check `wheb-template.html` and use whatever it contains; don't change it in existing files without asking.
- Gold: `#c9a84c`, Gold-light: `#e8d08a`, Off-white: `#f7f6f2`, Border: `#e2e2de`
- Font: system UI stack, no webfont. Page `max-width: 920px`.
- Header: sticky navy bar, logo left, nav links, phone (`01908 326586`) and email (`whteam@warren-house.co.uk`) right.
- `⊛ Tools` gold nav link is the **first** header-nav item on every tool → https://cswm.me.uk/tools
- Gold gradient divider: `linear-gradient(90deg, navy, gold, gold-light, navy)`
- Hero: navy background, white h1, gold-light subtitle.
- Sections: `.sec` + `.sec-hd` + `.sec-icon` (navy square, gold SVG) + `.stitle`
- Footer: 3 columns (logo + address | contact | tools links). Registered: Warren House Employee Benefits Limited, England & Wales No. 15133550.
- Floating back-to-top button, bottom-right, fades in on scroll.

---

## Tool notes

### Retirement Planner (`planner.html`)

**Chrome:** nav links "Employee Benefits" → warren-house.co.uk and "Personal Financial Planning" → Mike's True Potential page. Footer has no gold divider above it; logo 64px.

**Accumulation inputs:** DOB (auto-calcs age, state pension age from DWP transitional bands, ONS life expectancy — each auto-fill stops once the user edits that field); multiple pension pots (auto-totalled); salary, growth, inflation, salary growth; retirement age stepper buttons (not native spinners); employee/employer contributions each toggle % of salary vs £ pcm, with future-age tiers; one-off top-ups; separate ISA/GIA savings pots. Growth rate and retirement age sit in an assumptions bar above results.

**Decumulation** (collapsible "Plan retirement income", collapsed by default): TFC % (default 25%); annuity as a £ amount, remainder to drawdown; initial drawdown £ pcm with its own inflation; drawdown stages; one-off withdrawals; editable state pension amount/age with CPI toggle; guaranteed income sources each with own inflation; life expectancy plus "model to age".

**Joint mode:** Single/Joint toggle (default Single) reveals a full parallel partner input set sharing the same engine. Household results are **calendar-year aligned** (partners differ in age). Combined pot is a simple labelled sum at each person's own retirement age. Independent decumulation per partner, no shared pot, no survivor modelling.

**Engine:** shared pure functions `computeAccumulation` / `computeDecumulation` for single and joint. Regression-test against baseline numbers after any refactor. Income tax uses 2024/25 bands (PA £12,570, 20/40/45%), no taper, no Scottish rates. Monte Carlo: 500 runs, ±4%/yr accumulation, ±3%/yr drawdown, 10/25/50/75/90th percentiles and % exhausted.

**Results:** per-year pot chart; 7-year projection table (Age / Pot / 25% PCLS / annuity income); drawdown chart turning red past exhaustion; stacked income-by-source chart with running total in tooltip; household income chart; breakdown bars; tax cards; methodology section.

**Save/Load:** `gatherState()` / `applyState()`, `version: 2`; Save JSON (lossless, includes partner), Export CSV (includes PARTNER section), Load. Round-trip tested with a real reload.

**Print:** two buttons — Export accumulation / Export retirement income — using `print-mode-accum` / `print-mode-decum` classes, dynamic letterhead title.

**Out of scope (deliberate):** death/survivor benefits, salary sacrifice, annual allowance checking — separate tools.

### Annual Allowance Calculator (`annual-allowance-calculator.html`)

**Rules** (verified against HMRC PTM057100 / Royal London / Quilter):
- AA: 2008/09–2013/14 £50k; 2014/15 £40k; 2015/16 pre-alignment £80k; 2015/16 post-alignment £0; 2016/17–2022/23 £40k; 2023/24+ £60k.
- Carry forward: 2015/16 treated as a single year, effective £40k.
- Taper 2016/17–2019/20: thresh £110k, adj £150k, min £10k, baseAA £40k.
- Taper 2020/21–2022/23: thresh £200k, adj **£240k**, min £4k, baseAA £40k (was wrongly £260k for 2021/22–2022/23; fixed).
- Taper 2023/24+: thresh £200k, adj £260k, min £10k, baseAA £60k.
- Threshold income: `netIncome − RAS − deathBen + salSac`
- Adjusted income: `netIncome + netPay + empDC + (dbPIA − dbEmpEe) − deathBen`
- `deathBen` comes off **both**. RAS is in threshold income only, **not** adjusted.
- MPAA: £10k → £4k (2020–2023) → £10k (2023+). DC cap strictly enforced; no CF against DC; alternative AA for DB with CF.

**Implementation:**
- Join year selector (defaults to current tax year); pre-join years hidden.
- Taper: manual entry or 2-stage threshold → adjusted calculator with tooltips; compact audit-trail print view.
- Historical inputs table: DC/DB split when MPAA active; tabindex order; MPAA/TAPERED badges; cumulative AA available; taxable excess.
- CF: pre-computed `cfYearUnusedMap`; `getPrior3CFYears()` looks up exactly Y-1, Y-2, Y-3 from `CF_ORDER` — **never** `slice(-3)`.
- Results: 6 MPAA-aware metric cards, CF breakdown, alerts (AA charge, 2023/24 CF expiry, success).
- Save/Load: JSON + CSV, filename `aa_clientname_date`.
- Print: `preparePrint()` on `beforeprint`; page 1 = header, client strip, MPAA notice, results, disclaimer; page 2+ = MPAA detail, taper, steps 3–4. Use `#po-page1` / `#po-detail` ids — **not** `.p1` / `.p2`.

### TRS Generator (Total Reward Statement)

- Two-page A4 printable statement.
- Scheme configuration JSON editor with save/load (configs stored in SharePoint).
- Benefit categories: base scheme plus per-category overrides.
- AE contribution bases: qualifying earnings, Sets 1/2/3, custom.
- Salary sacrifice: simple vs SMART with exact NI band calculations.
- NMW checks: age bands, apprentice status, contracted hours, **52-week annual hours convention** (paid leave counts as worked hours).
- Costing: per-mille rates or flat costs for group risk (DIS, GIP, CIC); these costs count in the **employer package**.
- Holiday value shown only above the statutory minimum. Protection cover excluded from the package total.
- Structured sections for EAP, Cycle to Work, EV salary sacrifice; section activation tickboxes collapse cards.
- Floating tab navigation in sticky header; Tools link; back-to-top.
- Deterrent-level password gate only (not real security).
- **Data protection:** no online storage of completed statements. Pseudonymised data is still personal data under UK GDPR. Keep processing client-side or inside the employer's own M365 tenant. Statements contain salary, so PDFs should be password-protected or distributed via the employer's HR system.
- Testing: JS engine extracted to a Node module; suites `full.js`, `cats.js`, `cost.js` (74 checks: NI band straddling, AE sets, category inheritance, costing, NMW breaches). Run before every delivery.
- Compliance wording: statement of fact not advice; scheme rules prevail; illustrative only; flag taxable benefits.

### QR Code Generator (`qrgenerator.html`)

- 12 QR types including vCard (main use: Outlook email signatures).
- Engine: `qrcode-generator` from jsdelivr (switched from `qrcodejs`, which didn't render).
- Style/Frame tabs live in a permanently visible section below the type-specific inputs; `wTab` is scoped to its parent `.sec` so tab groups don't interfere. Labels are set in the Frame tab only.
- Apple Wallet `.pkpass` is **not possible** client-side (needs server-side Apple signing).

### Salary Exchange Calculator (`salexchange.html`)

- Built from the Scottish Widows adviser salary exchange spreadsheet (`salary-exchange-calculator-25-26.xlsm`) — its `Parameters`/`Calculations` sheets are the source of truth for the 2025/26 UK & Scottish tax bands and NI thresholds used here; the multi-member/bulk-employer sheets in that workbook were deliberately **not** ported (individual-employee tool only, by design).
- **Dynamic scenario builder:** one shared "Employee & salary details" block (employer name, UK/Scottish, above-SPA, salary, bonus) plus any number of independently-configured "structure" cards, each with its own method (Salary Sacrifice / Net Pay / Relief at Source), salary basis (Basic salary / Qualifying earnings / Total earnings) and amount type (%/£pm/£pyr) — any combination of any of these can be compared side by side, plus a fixed "No contribution" reference column.
- **Engine (`computeScenario`):** Salary Sacrifice reduces salary before both tax and NI (employer NI saving optionally reinvested 0/25/50/75/100%); Net Pay reduces taxable income only (no NI saving); Relief at Source is calculated on the **full, unreduced salary** for take-home pay (net contribution = gross × 0.8 paid from net pay) — any higher/additional-rate top-up relief beyond the mechanical 20% is computed but shown as a **separate, clearly-caveated line** ("may be claimable via Self Assessment/tax code adjustment"), never folded into take-home pay, since it isn't automatic. Verified against hand-calculated reference figures in a Node test harness before shipping (band-walk tax, NI thresholds, QE banding, above-SPA, bonus sacrifice, RAS-vs-Net-Pay equivalence).
- 2025/26 only (no year selector) — UK PA taper (£100k/£12,570), UK bands (£37,700/£74,870 widths, 20/40/45%), Scottish bands (starter/basic/intermediate/higher/advanced/top, 19/20/21/42/45/48%), employee NI (£12,570 threshold, 8%/2%), employer NI (£5,000 threshold, 15% — the post-April-2025 rate).
- Soft guardrail alerts (not hard blocks): sacrifice >80% of salary; resulting pay below the NLW-based `£22,308` floor (flagged to verify against the current published rate); total pension contribution above the standard £60k Annual Allowance (links conceptually to the Annual Allowance Calculator).
- Comparison table (grouped Pay/Pension/Employer rows, one column per structure) plus a Chart.js stacked bar (take-home / employee pension / employer pension per structure). "Pension £ per £1 take-home given up" is the headline value-for-money metric.
- Not modelled (deliberate): impact of reduced salary on other salary-linked benefits (life cover, mortgage affordability, statutory pay, means-tested benefits, student loan), multi-employee/bulk employer estimator, prior tax years.

### WHEB Tools Hub (`wheb-tools-hub.html`)

- `TOOLS` array: add a tool by copying an object `{title, url, description, tags[], icon (SVG path d attr), badge}`.
- Real-time search plus tag filter auto-built from tags.

### PDF Password Protector

- TODO: file name, URL, library used, and how it works.

---

## Toolchain & resources

- Python, Node.js (`--check`), Playwright for editing and verification.
- Microsoft 365 / SharePoint team environment; M365 Copilot licences available.
- Authoritative AA/taper sources: HMRC PTM057100, Royal London, Quilter.
- Scottish Widows is the most-used pension provider.

## Roadmap

- Add TRS Generator (and PDF Password Protector, if not already there) to the `TOOLS` array in `wheb-tools-hub.html`.
- Verify the live `planner.html` matches the feature list above.
- Mortgage suitability letter automation (separate track): M365 Power Automate / Power Apps with existing Copilot licences; client data must stay inside the firm's M365 tenant; v1 deliberately narrow, starting with the EOR branch; Phase 2 (tagged paragraph library from past letters) deferred.
- More tools to follow using the same template and deployment pattern.
