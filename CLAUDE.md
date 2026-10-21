# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

SEVORA — a front-end-only "AI trading platform" marketing/demo site (gold/XAUUSD focus, plus forex, crypto, indices). React 18 + Vite 5 + react-router-dom 6, plain JSX (no TypeScript). There is no backend, no test suite, and no linter configured. Despite the parent folder name, this is a JavaScript project.

## Commands

```bash
npm install
npm run dev       # Vite dev server (launch.json uses port 5173)
npm run build     # production build into dist/
npm run preview   # serve dist/ (launch.json uses port 4173)
```

`.claude/launch.json` defines `sevora-dev` and `sevora-preview`, which invoke `node_modules/vite/bin/vite.js` directly via `C:\Program Files\nodejs\node.exe`.

Deployment: `deploy.bat` builds, zips `dist/` into `sevora-dist.zip`, and opens Netlify Drop for a manual drag-and-drop upload. There is no CI.

## Build quirks (vite.config.js)

The build is deliberately shaped so `dist/index.html` works when opened from any path / static host:
- `base: './'` (relative asset URLs) and `HashRouter` in `src/main.jsx` — routes are `#/gold`, etc. Don't switch to `BrowserRouter`.
- Rollup output format is `iife` (not ESM), and a custom `fix-script-tags` plugin rewrites `<script type="module">` to `<script defer>` and strips `crossorigin` at build time only.

`dist/` and `sevora-dist.zip` are gitignored, though `dist/index.html` and a root-level `assets/index-*.js` are tracked from an earlier build — they are build output, not source.

## Architecture

- `src/main.jsx` → `HashRouter` → `LanguageProvider` → `App`. `App.jsx` wraps everything in `LiveMarketProvider`, shows a one-per-session `SplashScreen` (sessionStorage `sevora_splash_shown`), and declares routes. `/login` is standalone; every other page renders inside `components/Layout.jsx` (Sidebar, Topbar, PriceTicker, MobileNav, CommandPalette, FloatingCopilot, ParticleBackground) via `<Outlet>`.
- **Live market data** — `src/context/LiveMarketContext.jsx` is the single source of truth for prices. It polls the public TradingView scanner endpoint every 10s for the symbols in `TV_MAP`; those symbols must stay in sync with the ones the embedded TradingView chart widgets use (`components/TradingViewWidget.jsx`, which injects TradingView's external embed scripts). It also pulls gold headlines from Yahoo RSS via rss2json every 10 min and computes trading-session state. It exposes both `instruments` (keyed by symbol) and a backward-compat `market` object (`market.gold`, `market.btc`, …) that many components still read.
- **Derived "AI" levels** — `src/hooks/useLiveLevels.js` computes support/resistance/targets/bias/confidence and the AI daily plan purely from the live gold price with fixed formulas. When there's no live price it returns `null`s so UI shows `—` rather than stale numbers. `useMarketStatus.js` computes market open/closed and session from UTC time.
- **Mock data** — all "AI" features (mentor chat, copilot, bot dashboard, journal, marketplace, calendar, etc.) are simulated from `src/data/mockData.js` and hardcoded logic; there are no real AI/API calls beyond the market/news fetches above.
- **i18n** — `src/context/LanguageContext.jsx` provides `t('dot.path.key')` with `en` and `sr` (Serbian) dictionaries in `src/translations/`. Language persists in localStorage `sevora_lang`; a missing key returns the key string itself. Any user-visible string must be added to **both** `en.js` and `sr.js` with matching structure — recent work has been translating remaining hardcoded strings. Watch for local variables named `t` shadowing the translation function.
- **Styling** — a single global `src/index.css` with CSS custom properties on `:root` (note: `--gold*` variables are actually purple; `--price` is the amber price color) plus utility/component classes, combined with heavy inline `style={{}}` objects in components. Responsive breakpoints are `@media` blocks at 1280/1200/1024/768px in `index.css`; mobile layout is a recurring concern. Static images and the Geist font live in `public/`.
- Pages in `src/pages/` are large self-contained files (e.g. `Gold.jsx` ~1600 lines) with sub-components defined inline rather than split out.
