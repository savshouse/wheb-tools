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
| PDF Password Protector | https://cswm.me.uk/pdfprotector | `pdfprotector.html` |
| Salary Exchange Calculator | https://cswm.me.uk/salexchange | `salexchange.html` |
| Salary Exchange Calculator — Multiple Employees | https://cswm.me.uk/salexchange-bulk.html (no clean-URL redirect set up yet) | `salexchange-bulk.html` |

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
- `⊛ Tools` gold nav link is the **first** header-nav item on every tool → https://cswm.me.uk/tools. Exact markup (copy verbatim, don't retype the entity): `<a class="hdr-link" href="https://cswm.me.uk/tools" target="_blank" rel="noopener" style="color:var(--gold-light);font-weight:700">&#9783; Tools</a>`. This was missing from `wheb-template.html` itself (fixed 2026-09-28, after it shipped without one on `salexchange.html`, and `pdfprotector.html` had a mismatched "&#8592; WHEB Tools" variant) — the template now includes it, so new tools built from it get it for free. If you ever find a tool without it, that tool predates the fix; add the link above rather than reinventing it.
- **Segmented control / pill buttons (`.mode-toggle .mode-btn`, incl. `.seg-full` for 3+ option rows):** all buttons in a group must render at the same height regardless of whether their label wraps to one or two lines — never let a 2-line label ("Salary Sacrifice", "Qualifying earnings") stretch taller than a 1-line neighbour ("% of salary") in an adjacent row. Fix: `display:flex;align-items:center;justify-content:center` on `.mode-btn` plus a `min-height` sized to fit the longest (2-line) label at that font-size (e.g. `min-height:40px` at `font-size:11px`), not just padding. Check this whenever a segmented control is added or its label text changes. First got this wrong on `salexchange.html`'s scenario cards (fixed 2026-09-28).
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

- Originally built from the Scottish Widows adviser salary exchange spreadsheet (`salary-exchange-calculator-25-26.xlsm`, 2025/26 figures) — the multi-member/bulk-employer sheets in that workbook were deliberately **not** ported (individual-employee tool only, by design). Rolled forward to **2026/27** on 2026-09-28 (see below) — check the figures again before assuming any future year matches either.
- **Dynamic scenario builder:** one shared "Employee & salary details" block (employer name, UK/Scottish, above-SPA, salary, bonus) plus any number of independently-configured "structure" cards, each with its own method (Salary Sacrifice / Net Pay / Relief at Source), salary basis (Basic salary / Qualifying earnings / Total earnings) and amount type (%/£pm/£pyr) — any combination of any of these can be compared side by side, plus a fixed "No contribution" reference column.
- **Engine (`computeScenario`):** Salary Sacrifice reduces salary before both tax and NI (employer NI saving optionally reinvested, entered as a free-typed % — default 0, no preset options, for maximum flexibility); Net Pay reduces taxable income only (no NI saving); Relief at Source is calculated on the **full, unreduced salary** for take-home pay (net contribution = gross × 0.8 paid from net pay) — any higher/additional-rate top-up relief beyond the mechanical 20% is computed but shown as a **separate, clearly-caveated line** ("may be claimable via Self Assessment/tax code adjustment"), never folded into take-home pay, since it isn't automatic. Verified against hand-calculated reference figures in a Node test harness before shipping (band-walk tax, NI thresholds, QE banding, above-SPA, bonus sacrifice, RAS-vs-Net-Pay equivalence).
- **2026/27 only** (no year selector) — UK PA taper (£100k/£12,570), UK bands (£37,700/£74,870 widths, 20/40/45%, all frozen since 2021/22 and confirmed frozen to 2030/31 per Autumn Budget 2025), employee NI (£12,570 threshold, 8%/2%, unchanged), employer NI (£5,000 threshold, 15%, unchanged since April 2025, confirmed staying to at least 2030/31), AE qualifying earnings £6,240/£50,270 (DWP-confirmed unchanged for 2026/27), standard Annual Allowance £60,000 (confirmed unchanged post-Budget).
  - **Scottish bands did move for 2026/27** (rates unchanged at 19/20/21/42/45/48%, but Starter/Basic/Intermediate band widths increased ~7.4% per the Scottish Budget; Higher/Advanced/Top thresholds — £43,662/£75,000/£125,140 — unchanged, which is why those two bottom bands' *widths* are unchanged too): widths are now `[3967, 12989, 14136, 31338, 50140]` (was `[2306, 11685, 17101, 31338, 50140]` for 2025/26). **Don't assume UK-frozen ⇒ Scottish-frozen** — Scotland sets its own bands annually and has changed them almost every year; re-verify this specific number before trusting any later tax year.
  - NLW-based pay floor (`MIN_ANNUAL_INCOME`) updated to `£24,785` (was £22,308, itself already stale even for 2025/26) — from the 2026/27 rate of £12.71/hr × 37.5hrs × 52wks confirmed effective 1 April 2026. This rises every April — re-check it every tax year, it will never stay valid on its own.
- Soft guardrail alerts (not hard blocks): sacrifice >80% of salary; resulting pay below the NLW-based floor above; total pension contribution above the standard £60k Annual Allowance (links conceptually to the Annual Allowance Calculator).
- Comparison table (grouped Pay/Pension/Employer rows, one column per structure) plus a Chart.js stacked bar (take-home / employee pension / employer pension per structure). "Pension £ per £1 take-home given up" is the headline value-for-money metric (a ratio, so it's never rescaled by the period toggle below).
- **Annual/Monthly toggle** (`periodToggle`, top-right of the Comparison section header): a display-only transform — `computeScenario` and every stored figure stay annual; `periodValue()`/`fmtP()` divide by 12 for on-screen table rows, the chart datasets and the RAS extra-relief note only when Monthly is selected. Deliberately **not** applied to the NLW/AA guardrail alerts or the £-per-£1 ratio row, since those are inherently annual/ratio figures and dividing them would misstate the comparison. The print/PDF output just prints the live page, so it automatically reflects whichever period is selected.
- **Contribution amount inputs** (`sc-ee-amt`/`sc-er-amt`, and the NI-reinvestment field) show a live `£` prefix or `%` suffix depending on the row's Amount type, via `.amt-affix`/`mode-pct`/`mode-currency` classes toggled in `refreshScenarioCardVisibility` — purely cosmetic, doesn't affect the stored value.
- **Mobile (≤640px):** the comparison table's first (label) column is capped to `90px` and allowed to wrap (`white-space:normal`) instead of auto-sizing to its longest label — unconstrained, it was eating ~75% of the visible width on a 390px screen. Desktop is untouched (rule is behind the media query). If the table gains more feature columns and mobile gets cramped again, revisit this width rather than reverting it.
- **Tax code input** (`tax-code`, defaults to `1257L`): `parseTaxCode()` supports standard L/M/N/T allowance codes (allowance = numeric×10), flat-rate codes BR/D0/D1/NT, `0T` (zero allowance, no taper), and K-codes (adds numeric×10 to taxable income, no allowance). A leading S/C prefix is stripped and ignored — the existing Tax jurisdiction toggle remains the sole control for UK vs Scottish bands, so a tax code's own country prefix never overrides it; flat-rate codes always use rUK rates regardless of the toggle. `taperedPA()` and `incomeTax()` take a `taxCodeInfo` object instead of always assuming the standard personal allowance. Unrecognised codes fall back to 1257L and surface a warning in `#alerts`.
- Not modelled (deliberate): impact of reduced salary on other salary-linked benefits (life cover, mortgage affordability, statutory pay, means-tested benefits, student loan), prior tax years. Multi-employee/bulk salary sacrifice is a **separate tool** — see below.

### Salary Exchange Calculator — Multiple Employees (`salexchange-bulk.html`)

- Built 2026-09-28 in response to a request to model groups of employees at once, referencing the original Scottish Widows spreadsheet's `Inputs_multi-member`/`Calcs_MM` sheets as a pattern (shared scheme-wide settings + one row per employee) — but with a different, more detailed column set the user specified directly (matches their payroll bureau's contribution schedule, not the spreadsheet's simpler annual comparison table).
- **Salary Sacrifice only, by design** — confirmed with the user before building, since the requested columns (NI Rebate, Employer Single/bonus sacrifice) are Salary-Sacrifice-specific concepts with no Net Pay/RAS equivalent (no employer NI saving to rebate, no pre-tax bonus mechanism). Does **not** compute income tax, employee NI or take-home pay at all — only pension contribution amounts — since none of the requested output columns needed them.
- **Per-employee model, monthly (PCM) cadence**, not annual like the single-person tool: `Pensionable Earnings = Period Earnings (PCM) × Alteration Factor` (the 0–1 proportion of the month worked, e.g. for starters/leavers), with **no qualifying-earnings banding** — contribution rates apply directly to Pensionable Earnings. `Annual Salary` is reference-only and never feeds a calculation.
- **NI Rebate is scheme-wide, not per-employee** (`ni-rebate-pct`, single input, confirmed with the user) — matches the original spreadsheet's "% of Employer NI Saving to Pension" pattern. The tool calculates each employee's own employer NI saving from their own sacrificed amount and shows the £ result per employee in both the table and the CSV export.
- **Employer NI saving approximation:** `employerNIMonthly()` uses the annual secondary threshold (£5,000) ÷ 12 and the 15% rate, applied to that employee's own Pensionable Earnings before/after their sacrifice — a documented simplification of real payroll's period-threshold/cumulative NI calculation; called out in the assumptions section.
- **State Pension age flag (`SPA` badge):** `statePensionDate()` reuses the same simplified DWP transitional-band bands as `planner.html`'s `statePensionAgeYears()`, but returns an actual date (not a rounded age) so it can be compared directly against today. **Informational only** — does not change NI or any contribution figure; the adviser is expected to confirm NI treatment in payroll.
- **Minimum wage flag (`NMW` badge):** compares post-sacrifice pay this period ÷ hours actually worked this period (`Hours Per Week × 52/12 × Alteration Factor`) against 2026/27 age-banded rates verified via WebSearch (21+: £12.71/hr; 18–20: £10.85/hr; under 18: £8.00/hr, confirmed effective 1 April 2026). Age is derived from DOB. **Apprentice status is not captured by the requested columns, so the apprentice sub-rate (£8.00/hr) is not modelled** — apprentices need checking individually. Re-verify all three rates every tax year (NMW/NLW rises every April, same caveat as the single-person tool's `MIN_ANNUAL_INCOME`).
- **CSV import/export**, exact columns per the user's spec: `Employee Number, Date of Birth, Hours Per Week, Annual Salary, Period Earnings (PCM), Alteration Factor, Pensionable Earnings, Employee Contribution Rate %, Employer Contribution Rate %, Employee Contribution Amount £, Employer Contribution Amount, NI Rebate, Employer Single, Total Contribution Amount`. The first 6 plus EE/ER rate % and Employer Single are inputs; Pensionable Earnings, both contribution amounts, NI Rebate and Total are always recalculated on import/export, never trusted from the file. Hand-rolled RFC4180-ish CSV parser/writer (no library) — handles quoted fields and embedded commas. "Download CSV template" gives headers + one worked example row.
- **Table performance:** hundreds-of-rows scale (confirmed with the user). Full `renderTable()` re-render only on import/add/remove; a keystroke in any row updates just that row's calculated `<td>`s and the summary cards via event delegation on `#emp-tbody`, never rebuilding the DOM under the user's cursor (see the `oninput` DOM-rebuild bug pattern at the top of this file).
- No clean URL configured on Fasthosts yet — linked directly as `salexchange-bulk.html` from the Tools Hub until Mike sets one up (see Live Tools table above).
- Not modelled (deliberate, same reasoning as the single-person tool plus): Net Pay/Relief at Source for groups, qualifying-earnings banding, income tax/take-home pay, Annual Allowance tapering/MPAA, apprentice NMW rates.

### WHEB Tools Hub (`tools.html`)

- `TOOLS` array: add a tool by copying an object `{title, url, description, tags[], icon (SVG path d attr), version, lastUpdated}`.
- `version` (e.g. `'v1.0'`) renders as a chip on the card; `lastUpdated` (`'YYYY-MM-DD'`) renders as a "Last updated: …" line at the bottom of the card. Both are set by hand — bump `version` and `lastUpdated` together whenever a tool gets a meaningful update. There is no "new"/"updated" badge any more (it never expired on its own, so it was replaced with this).
- Real-time search plus tag filter auto-built from tags.

### PDF Password Protector (`pdfprotector.html`)

- Mode toggle: **Add password** / **Remove password**. Two entirely separate engines, deliberately not sharing code, wired via `mode` + a branch at the top of the single `#protectForm` submit handler.
- **Add password** (unchanged since first shipped): `@cantoo/pdf-lib` (jsdelivr CDN) loads the uploaded PDF and calls `pdfDoc.encrypt({ userPassword, ownerPassword })` (same password for both), then downloads the result.
- **Remove password**: `@cantoo/pdf-lib`'s own decrypt-then-resave was verified (2026-09-26) to be broken in the published `2.11.1` dist — `PDFDocument.load(bytes, {password})` authenticates correctly and clears `isEncrypted`, but `.save()` still re-embeds a stale `/Encrypt` entry, producing output that opens with **neither** the original password **nor** no password (genuinely corrupted). Confirmed directly against the raw library, independent of our UI. If revisiting that library for this: check for a release past 2.11.1 and re-run the same round trip (create → encrypt → load-with-password → save → try loading with and without the original password) before trusting it again.
  - Instead, removal uses **QPDF** (the real `qpdf` CLI, industry-standard for exactly this) compiled to WebAssembly via the `qpdf.js` npm package (jsdelivr CDN, `qpdf.js@2.0.0/src/`), run in a dedicated Web Worker: `qpdf.execute(['--decrypt', '--password=...', '--', 'in.pdf', 'out.pdf'])` against its virtual filesystem.
  - **Worker loading quirk (important if editing this):** a classic `Worker` must be constructed from a same-origin script — no CORS header can override this, confirmed by direct testing. So the CDN's `qpdf-worker.js` can't be passed straight to `new Worker(url)`. The code fetches its source (`fetch()` is CORS-fine), replaces the one line that derives its own asset base path from `self.location.href` (which would resolve to a useless `blob:` URL otherwise) with the real hardcoded CDN base path, then constructs the worker from a `Blob` of the patched source. Everything after that — `importScripts()` for the emscripten glue, `locateFile()` fetching the `.wasm` — is a normal cross-origin resource load, which browsers allow without restriction. This exact sequence (including a true cross-origin round trip on two different local origins) was verified working before shipping.
  - A cheap pre-check (`PDFDocument.load(bytes, {ignoreEncryption:true})` then `.isEncrypted`, using the same already-loaded pdf-lib — read-only, doesn't touch the encrypt path) gives a friendly "this PDF isn't password protected" message instead of a silent no-op, since qpdf's own `--decrypt` succeeds without complaint on an already-unencrypted file.
  - qpdf's own error for a wrong password is the string `"<file>: invalid password"` — matched case-insensitively for the friendly on-screen message.
- Now listed in the Tools Hub (`tools.html`).

---

## Toolchain & resources

- Python, Node.js (`--check`), Playwright for editing and verification.
- Microsoft 365 / SharePoint team environment; M365 Copilot licences available.
- Authoritative AA/taper sources: HMRC PTM057100, Royal London, Quilter.
- Scottish Widows is the most-used pension provider.

## Roadmap

- Add TRS Generator to the `TOOLS` array in `tools.html` (PDF Password Protector done).
- Verify the live `planner.html` matches the feature list above.
- Mortgage suitability letter automation (separate track): M365 Power Automate / Power Apps with existing Copilot licences; client data must stay inside the firm's M365 tenant; v1 deliberately narrow, starting with the EOR branch; Phase 2 (tagged paragraph library from past letters) deferred.
- More tools to follow using the same template and deployment pattern.
