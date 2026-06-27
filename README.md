# Vanguard Breakout Station

An AI-augmented breakout & swing trading workstation. A Node/Express backend
screens the Nifty 50 (India) or US large-caps for high-conviction breakout
setups, enriches each with volatility-adjusted trade levels, news sentiment, and
a recommendation, and a React/Vite/Tailwind frontend renders an institutional
dark dashboard with interactive 4H charts, an AI trade dossier, and a live
portfolio tracker.

> **For educational and research purposes only. Not financial advice. Always
> verify liquidity before executing live market orders.**

---

## Quick start

You need **Node.js 18+** (the backend uses native `fetch` and ES modules). Run
the backend and frontend in two terminals.

### 1. Backend (port 3001)

```bash
cd backend
cp .env.example .env        # optional — sensible defaults already work
npm install
npm start                   # node server.js → http://localhost:3001
```

Health check: open `http://localhost:3001/api/health`.

### 2. Frontend (port 5173)

```bash
cd frontend
npm install
npm run dev                 # → http://localhost:5173
```

Vite proxies `/api/*` to `http://localhost:3001`, so no CORS or base-URL config
is needed in development. Open the printed localhost URL, pick a market, and
click **Run Market Scan**.

---

## How the screen works

A ticker must clear **all eight checks** to appear in the scanner. The checks
fold the institutional-grade filters from the brief directly into the pipeline.

1. **Liquidity & price floor** — minimum price and average traded value, so
   illiquid names are dropped before any heavier analysis.
2. **Daily uptrend** — latest daily close is above its 50-day SMA.
3. **Near highs** — price sits within ~10% of its ~52-week high (leadership).
4. **Valid 4H base** — a detectable consolidation over the prior 20×4H bars with
   a range no wider than ~15%.
5. **Breakout** — the latest 4H close prints above the consolidation
   resistance.
6. **Breakout-candle conviction** *(ATR volatility filter)* — the breakout
   candle body exceeds **1.5× the recent 4H ATR**, filtering normal noise.
7. **Relative volume** *(RVOL tiering)* — RVOL ≥ 1.5×; anything **> 3.0×** is
   tagged **"Institutional Accumulation."**
8. **Momentum health** — daily RSI between 50 and 80 with a rising 20-day SMA
   (momentum without being blown-off overbought).

Qualifying tickers are scored, the **top 50** are returned, and results are
**grouped by sector** (`quoteSummary` → `assetProfile`) so industry-wide
tailwinds are obvious at a glance.

### Enrichment per result

- **Pattern** — Flat Base, Ascending Triangle Close, Channel Breakout, or Range
  Breakout, classified from the consolidation geometry.
- **Trade setup** — strict **1:2 risk-to-reward** off the 14-period daily ATR:
  `entry = close`, `stop = close − 2·ATR`, `target = close + 4·ATR`.
- **News sentiment** — keyword-weighted **Bullish / Bearish / Neutral** over the
  latest headlines.
- **Recommendation** — **Strong Buy** (breakout + bullish news), **Speculative
  Buy** (breakout, neutral/no news), or **Hold/Watch** (breakout conflicting
  with bearish news).
- **Risk warning** — flagged when price is extended more than 10% above its
  20-day SMA.

### Weekly swing watchlist

`/api/swing-ideas` is deliberately strict and returns at most **5** names: a
20-day volatility squeeze (Bollinger width compressed vs its median), daily RSI
between 50–65, volume expanding over recent sessions, and heavily positive news.

---

## API

| Method | Endpoint | Body / Query | Returns |
| ------ | -------- | ------------ | ------- |
| `POST` | `/api/scan` | `{ market: 'india' \| 'usa' }` | `{ market, count, sectors, results[] }` |
| `POST` | `/api/chart` | `{ ticker }` | candles, consolidation band, entry/stop/target lines |
| `GET`/`POST` | `/api/swing-ideas` | `market` | `{ market, count, results[] }` |
| `GET` | `/api/portfolio` | — | trade array |
| `POST` | `/api/portfolio` | `{ ticker, market, type, executionPrice, shares, date }` | created trade |
| `DELETE` | `/api/portfolio/:id` | — | `{ ok, removed }` |
| `GET` | `/api/health` | — | service status |

Portfolio trades persist to `backend/portfolio.json`.

---

## Configuration (`backend/.env`)

| Variable | Default | Purpose |
| -------- | ------- | ------- |
| `PORT` | `3001` | API port |
| `SCAN_CONCURRENCY` | `5` | Parallel ticker fetches during a scan |
| `FINNHUB_API_KEY` | *(unset)* | Optional. News works without it. |

**News source.** Headlines come from the **keyless Google News RSS feed**, which
covers both Indian and US tickers out of the box — no key required. A Finnhub
key is supported as an optional enhancement but is not needed to run.

---

## Notes & caveats

- **Market data uses an unofficial Yahoo Finance endpoint** (`yahoo-finance2`).
  It is rate-limited and occasionally returns gaps; a scan touches dozens of
  tickers, so the first run can take a little time and a few names may be skipped
  if Yahoo throttles. Results are cached briefly to keep things responsive.
- **`lightweight-charts` is pinned to `4.2.3`.** The chart code uses the v4 API
  (`addCandlestickSeries`, `createPriceLine`, `LineStyle`). The v5 API renamed
  these, so do not bump the major version without updating `ChartPane.jsx`.
- **Tailwind v4** is wired through the `@tailwindcss/vite` plugin with
  `@import "tailwindcss";` in `src/index.css` — there is intentionally no
  `tailwind.config.js`.
- Portfolio P/L is summed in raw currency units; India (₹) and US ($) positions
  are **not** FX-converted.

---

## Project structure

```
vanguard-breakout-station/
├── backend/
│   ├── server.js              # Express app + routes
│   ├── lib/
│   │   ├── universe.js        # Nifty 50 / US large-cap symbol lists
│   │   ├── indicators.js      # SMA, RSI, ATR, RVOL, Bollinger, slope…
│   │   ├── patterns.js        # consolidation detection + classification
│   │   ├── sentiment.js       # Google News RSS + keyword scoring
│   │   ├── scanner.js         # 8-point screen, scoring, swing ideas, chart
│   │   └── cache.js           # in-memory TTL cache
│   ├── portfolio.json         # persisted trades
│   └── .env.example
└── frontend/
    ├── src/
    │   ├── App.jsx            # shell, market toggle, scan orchestration
    │   ├── api.js             # typed client over /api
    │   ├── format.js          # currency + badge class maps
    │   └── components/
    │       ├── Sidebar.jsx
    │       ├── StatusLog.jsx
    │       ├── ResultsTable.jsx
    │       ├── ChartPane.jsx        # lightweight-charts 4H view
    │       ├── AnalysisCard.jsx     # AI trade dossier
    │       ├── ScannerView.jsx      # split-screen scanner / swing
    │       └── PortfolioView.jsx    # FIFO ledger + live positions
    ├── vite.config.js         # react + tailwind plugins, /api proxy
    └── index.html
```

---

## Disclaimer

This software is provided for **educational and research purposes only**. It is
**not financial advice**, not a recommendation to buy or sell any security, and
makes no guarantee of data accuracy. Markets carry risk; always do your own
research and **verify liquidity before executing live market orders.**
