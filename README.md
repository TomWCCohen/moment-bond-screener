# Moment Bond Screener

A single-page fixed income trading platform built on the [Moment API](https://moment-api.com), submitted as part of the Moment internship application.

## Live Demo

> Vercel Link: https://tcohen-moment-bond-screener.vercel.app

---

## What We Built

The Moment Bond Screener is a client-facing prototype of a fixed income trading workflow, built on the Moment paper API. The goal was to go beyond a basic data viewer and implement a realistic end-to-end flow: screen bonds, build and compare portfolio scenarios, analyze them with AI, and submit orders — all from a single self-contained file with no backend and no build step.

The application was built iteratively across several sessions, starting from a raw API integration and progressively adding layers of functionality: live pricing, AI analysis, multi-scenario management, bulk order submission, a persistent order blotter, and scenario comparison.

---

## Features

### Bond Screener
The left panel displays the full bond universe available in the Moment paper environment (~100 bonds). Every column is sortable. The table updates instantly as filters change, with no page reload.

- **Search** by ISIN, CUSIP, issuer name, or ticker
- **Filter** by sector, S&P rating, and bond status (Outstanding / All)
- **Sort** by any column (ISIN, issuer, coupon, maturity, rating, price, YTW, duration, sector)
- **Bulk select** via checkboxes — select multiple bonds and add them all to a scenario at once
- **Smart Screens** — one-click pre-built filters:
  - Investment Grade (BBB- and above)
  - High Yield (BB+ and below)
  - Short Duration (< 5 years)
  - Best YTW (sorted by yield-to-worst descending)
  - Callable bonds
  - Utilities sector
  - Financials sector
- **Hover tooltips** on each ISIN showing the bond's short description

### Bond Detail Panel
Clicking any bond opens a full detail view in the right panel. The panel is designed to give a trader everything they need to make a decision without leaving the screen.

**Header**
- Issuer name, ISIN, CUSIP, bond type, currency
- Short description of the bond
- Live price (fetched from the Moment marks endpoint on selection) with a "LIVE" indicator
- Yield-to-worst alongside the price
- Quick "Add to Scenario" button with a dropdown to choose or create a scenario

**Key Metrics Grid** (9 metrics)
- Coupon rate and coupon type
- Maturity date
- Modified duration
- S&P rating (color-coded badge: green → AAA, yellow → BBB, red → HY, grey → NR)
- Moody's rating
- Callable status with next call date if applicable
- Settlement convention
- Bond status

**Calculated Metrics Bar** (5 derived metrics)
- Current yield (annual coupon / price)
- Risk-adjusted yield (YTW per year of duration — a quick proxy for return per unit of rate risk)
- vs. Par (trading premium or discount, color-coded)
- Liquidity score (5-dot visual scale derived from Moment's institutional and retail liquidity aggregates)
- Time to maturity with exact date

**AI Analysis** (collapsible section)
Calls Claude Haiku directly from the browser and structures the response into 3 labeled cards:
- Issuer Profile — who the issuer is, business model, credit context
- Investment Case — why this bond is or isn't attractive at current levels
- Key Risk — the primary risk factor a buyer should monitor

The section is collapsible (chevron toggle) and has a Refresh button to regenerate without reopening the bond.

**Press & Research Links**
Quick-access links to search for the issuer across: Google News, Bloomberg, Reuters, Financial Times, Wall Street Journal, SEC EDGAR.

**Price History Chart**
Interactive Chart.js line chart with 7D / 30D / 90D tabs. Hover tooltip shows price, absolute change, and percentage change vs. the previous day. Chart color is green if the bond is up over the period, red if down.

**Order Form**
- Buy / Sell toggle
- Amount in par value ($) with a $10,000 minimum
- Limit price (auto-filled from the live mark price on selection)
- Submit button that calls `POST /v1/trading/order/` and immediately starts polling the order status every 3 seconds

### Scenario Builder
The Scenario Builder is the core of the application — a modal workspace for constructing and managing bond portfolio scenarios before sending them to market.

**Multi-scenario tabs**
- Create as many named scenarios as needed (Scenario 1, 2, 3…)
- Switch between them with tabs inside the modal
- Delete any scenario except the last one
- All scenarios persist in localStorage across sessions and page reloads

**Building a scenario**
- Add individual bonds via the "+ Scenario" dropdown in the detail panel
- Add multiple bonds at once via the bulk-select action bar in the screener table
- Bonds already in a scenario are highlighted in purple in the screener table
- Remove any bond from the scenario with the × button

**Per-bond controls**
- BUY / SELL toggle for each position
- Amount field (par value, $) with live allocation percentage recalculation

**Portfolio metrics panel** (right sidebar, live)
- Number of bonds
- Weighted average YTW
- Weighted average duration
- DV01 (dollar value of a 1 basis point rate move — dollar loss per 1bp rise in rates)
- % Investment Grade
- Total portfolio value
- Donut allocation chart (interactive, hover to see par amounts)

**AI scenario analysis**
Sends the full scenario (all bonds, weights, ratings, yields, duration) to Claude Haiku and returns a 3-point structured analysis: overall quality and credit risk profile, key risk metrics assessment, and one specific actionable improvement recommendation.

**Submission**
- "Submit All Orders" sends the full scenario to `POST /v1/trading/order/bulk/` in a single request
- Each order in the bulk submission is then polled individually for status updates
- The submission is recorded in the scenario's history immediately

**Submission history bar**
Every time a scenario is submitted, a new row appears in a history bar at the bottom of the modal. Each row shows the date, time, and number of bonds. Hovering a row reveals a detailed tooltip listing every bond submitted: issuer name, ISIN, side, and amount. The history is fully persistent across sessions — you can reload the page and see every past submission.

### Scenario Comparison
A dedicated modal that compares all non-empty scenarios side by side across 12 metrics:

- Number of bonds
- Total amount ($)
- Weighted average YTW
- Weighted average duration
- DV01
- Risk-adjusted yield
- % Investment Grade
- % High Yield
- % Not Rated
- Top sector by allocation
- Max single-name concentration
- Average maturity

Best values are highlighted in green, worst in red. Each numeric metric includes a mini bar chart for visual proportion. An AI comparison button sends all scenario summaries to Claude Haiku for a decisive recommendation on which scenario wins and why.

### Order Blotter
A modal order history accessible from the "Orders" button in the header (with a live badge count).

- Displays all orders placed in the current and past sessions
- **Persistent** — orders are saved to localStorage and restored on page reload
- **Blue dot indicator** — a small blue dot marks orders placed in the current session, making it easy to distinguish from historical orders
- **Status polling** — after any order is submitted, the app polls `GET /v1/trading/order/{id}/` every 3 seconds until the order reaches a terminal state (filled, rejected, cancelled). Status updates are persisted immediately so they survive a reload.
- **Auto-refresh** — while the blotter is open, it refreshes from the Moment API every 10 seconds (LIVE indicator with animated dot)
- **Rejection reasons** — if an order is rejected, the reason from the API response is displayed in red below the status badge
- Columns: session indicator, order ID, bond/issuer with ISIN, side, amount, price, status, date, time

### Settings
A settings modal (gear icon, top right) allows overriding the Anthropic API key (for AI features) and the Moment API key. Both are stored in localStorage only — never sent anywhere except the respective APIs.

---

## Architecture

Single-file HTML application — no build step, no dependencies beyond Chart.js (CDN).

```
bond-screener-v4.html
├── CSS           (~480 lines)  design system, layout, components
├── HTML          (~410 lines)  structure, modals, forms
└── JavaScript    (~1030 lines)
    ├── Config & state
    ├── API layer             → Moment paper API
    ├── Claude AI layer       → claude-haiku-4-5-20251001
    ├── Bond screener         → filter, sort, render
    ├── Bond detail           → metrics, chart, order form
    ├── Scenario system       → multi-scenario, comparison
    ├── Order blotter         → submit, poll, persist
    └── Helpers & init
```

---

## API Integration

**Moment paper API** (`https://paper.moment-api.com`)

| Endpoint | Usage |
|---|---|
| `GET /v1/data/instrument/` | Load bond universe |
| `GET /v1/data/instrument/{isin}/price/` | Price history chart |
| `GET /v1/data/marks/` | Live mark prices on bond selection |
| `POST /v1/trading/order/` | Single order from detail panel |
| `POST /v1/trading/order/bulk/` | Scenario bulk submission |
| `GET /v1/trading/order/{id}/` | Order status polling (every 3s) |
| `GET /v1/trading/order/` | List all orders for blotter sync |

> **Note on order status:** All orders are rejected by the Moment paper environment's Risk Management System. This is expected — the paper API key provided grants access to market data endpoints only. The full order flow (payload construction, bulk submission, individual status polling, rejection reason capture) is implemented per the Moment API specification.

**Claude API** (`claude-haiku-4-5-20251001`)
Called directly from the browser using Anthropic's supported client-side access pattern (`anthropic-dangerous-direct-browser-access`). Used for bond-level analysis, scenario analysis, and scenario comparison. The API key is user-provided via Settings and stored in localStorage only — never sent to any intermediary server. In a production context this would be proxied through a backend.

---

## Setup

Open `bond-screener-v4.html` in any modern browser. No installation, no server, no build step required.

To enable AI analysis, add your Anthropic API key in Settings (⚙️ top right).

The Moment API key is pre-configured for the paper environment.

---

## Known Limitations & Production Considerations

- **Order execution:** The paper API key grants market data access only — all orders are rejected by the RMS. The trading flow is fully implemented and would function with a trading-enabled account.
- **Claude API client-side:** For simplicity, AI calls are made directly from the browser. A production deployment would proxy these through a backend to avoid exposing the API key.
- **Paper API key in source:** The default Moment paper key is bundled for ease of demo. It is read-only and scoped to the paper environment. A production setup would inject this at deploy time via an environment variable.
- **No authentication:** The app has no user login — it is designed as a single-user prototype.
- **~100 bonds:** The paper environment exposes a limited bond universe. The screener is built to scale to the full 1M+ bond universe available in the live environment.

---

## Local Storage

| Key | Contents |
|---|---|
| `moment_scenarios_v3` | All scenarios, bond allocations, and full submission history |
| `moment_orders_v1` | Persistent order blotter history |
| `claude_api_key` | Anthropic API key (user-provided) |
| `moment_api_key` | Moment API key override |
