# /root/trading-advisory — concrete deployment (Aug 2026)

Working implementation of the trading-advisory-bot pattern on the home server.
Git repo (v2.9, clean git tree). Python venv at `.venv` (3.11), React+Vite+TS
in `web/`, systemd units in `systemd/`. **Public: https://advisory.lufria.tech**
(Cloudflare Tunnel → localhost:8001; UI statis + API diserve FastAPI di satu port).

## Ports & services

- FastAPI: `127.0.0.1:8001` (`trading-api.service`, uvicorn). Port 8000 is
  taken by Clipper-Apps — never reuse it.
- Vite dev: `127.0.0.1:5173`, proxy `/api` → `http://127.0.0.1:8001`.
- systemd timers active:
  * `trading-ingest.timer` (*:0/15) — Market data ingestion
  * `trading-signal.timer` (*:0/30) — General AI signal scan
  * `trading-monitor.timer` (*:0/5) — SL/TP monitor & alert
  * `trading-bpjs.timer` (Mon..Fri 08:30 WIB / 01:30 UTC) — Morning BPJS Briefing
  * `trading-bsjp.timer` (Mon..Fri 15:35 WIB / 08:35 UTC) — Afternoon BSJP Briefing
  * `trading-evaluate.timer` (Daily 00:00 UTC) — Accuracy & signal outcome evaluator

## Architecture evolution (v0.1 → v2.9)

- **v0.1**: Skeleton FastAPI + MySQL 8.4 + pandas manual indicators + DeepSeek + dual notifier (Telegram DM + Discord webhook) + React basic.
- **v0.2**: Market Overview 25 crypto + 25 stock watchlist di DB, yfinance provider (saham gratis tanpa key), candlestick chart (lightweight-charts), ticker tape horizontal loop, tabs Crypto / Saham AS / Sinyal / Portfolio / Riwayat. Hold signals normalized to None targets.
- **v0.3**: WebSocket real-time `/ws/market` (crypto 5s, stock 30s) + frontend `useMarketSocket` hook (auto-reconnect 1s→2s→5s + REST fallback). SignalCard hides Entry/SL/TP grid completely on `hold` signals.
- **v0.4**: Session-based auth (`app/auth.py`, HMAC signed cookie `session`, httpOnly, samesite=lax, 24h expiry = 1 day). Protected all `/api/*` and WebSocket handshake (reject 403 without cookie). Login page + AuthContext + Route guard + Logout button.
- **v0.5**: Saham Indonesia (IDX, 25 saham top via `asset_type='stock_id'`, yfinance `.JK` suffix, harga & form beli dalam IDR/Rp) + Live News Stream (`app/data/news.py`, Google News RSS zero-key, `/api/news` + `/berita` page + detail panel news) + integrasi sentimen berita ke Signal Engine Gemini prompt.
- **v0.6**: Full-scan 75 watchlist assets (`signal_max_symbols=35`), attach latest signals directly into market overview table rows (`Symbol | Harga Live | Chg 24h | Sinyal | Target TP | Potensi %`), and Sinyal Feed view modes ("Sinyal Aktif per Simbol" vs "Riwayat Log").
- **v0.7**: 218+ assets universe (89 IDX Indonesian stocks, 80 US stocks, 50 Crypto pairs). Fast batch quoting via `yf.download`. Instant live search bar with dynamic `+ Tambah ke Watchlist`. Instant cache invalidation (`invalidate_overview_cache`) on watchlist changes (zero delay). Table pagination (15, 25, 50, Semua). HOLD status displayed explicitly as "Wait & See" / "Netral".
- **v0.8**: Pinned Watchlist & VIP Priority Engine (`is_pinned` column in DB `watchlist`). Stockbit portfolio holdings (`JARR`, `UDNG`, `BRPT`, `KAEF`, `BLOG`, `CUAN`) pinned as priority #1 in scan engine. Pinned Dashboard Carousel widget on top of Market table + 1-click Pin toggle.
- **v0.9**: Executive Dashboard page (`/` & `/dashboard`) with Major Market Indices (`IHSG ^JKSE`, `LQ45 ^JKLQ45`, `S&P 500 ^GSPC`, `Nasdaq ^IXIC`, `Dow Jones ^DJI`, `Bitcoin BTCUSDT` via `GET /api/market/indices` with 60s cache). Real Stockbit Portfolio Tracking (6 holdings: `CUAN`, `BRPT`, `JARR`, `BLOG`, `KAEF`, `UDNG` totaling **Rp 411.299.690** modal).
- **v1.0**: Advanced Sorting & Filtering: Market page sorting (Top Gainers, Top Losers, Potensi Naik Tinggi/Rendah, Harga) + Portfolio sorting (Percentage P&L Besar/Kecil, Nominal P&L Rp Besar/Kecil, Modal Besar/Kecil) + category filters.
- **v1.1**: Portfolio Potential Recovery Gain Calculations (`potential_gain_pct`, `potential_gain_idr` per holding + Executive Recovery Banner showing `+Rp X` recovery potential if AI Target TP is achieved).
- **v1.2**: Persistent UI Session & Filter State across all pages using `usePersistedState<T>` with `localStorage`.
- **v1.3**: Multi-User Support: Full data isolation between `lufria` and `aura` (separate `positions.user_id` and personal `user_pins`), authenticated via `AUTH_USERS` in `.env`.
- **v1.4**: Portofolio Aura integration (3 holdings: `CDIA`, `COAL`, `JARR` totaling **Rp 45.283.036** modal) + Dual-sided Risk/Reward visualization (Downside Stop Loss % alongside Upside Take Profit %).
- **v1.5**: Engine Khusus **⚡ DAY TRADING (1 HARI SELESAI / INTRADAY)**: Target Take Profit dikalibrasi realistis untuk 1 sesi bursa (+1.5% s/d +3.5% / 2-5 fraksi harga) dengan Stop Loss ketat (-1.0% s/d -2.0%) dan label `⚡ Day Trade (1 Hari)`.
- **v1.6**: **Real-Time Intraday Percentage Fix**: Replaced delayed daily close calculations in `yfinance_provider` with multi-threaded `fast_info` (`last_price` vs `previous_close`), matching live broker apps (Stockbit) 100%. **Macro & Political Demo News Integration**: Injected live national macro events (demonstrations, political stability, Rupiah exchange rate) into Gemini's prompt for risk-aware Stop Loss adjustments.
- **v1.7**: **Engine Khusus BPJS & BSJP**: Automated daily pre-market & pre-close briefings to Discord `#advisory` and Telegram (`trading-bpjs.timer` at 08:30 WIB for Beli Pagi Jual Sore, `trading-bsjp.timer` at 15:35 WIB for Beli Sore Jual Pagi Besok) with concurrent AI analysis via `asyncio.Semaphore(6)`.
- **v1.8**: **Training Memory / Few-Shot Calibration Engine & News Catalyst Deep Dive**:
  - `app/llm/training_memory.py`: Injects historical winning and losing patterns (BPJS, BSJP, Overbought UMA trap) directly into prompt context.
  - DB table `signal_outcomes` and automated evaluator job `app/jobs/evaluate_signals.py` (`trading-evaluate.timer`).
  - Endpoint `GET /api/signals/accuracy` reporting historical win rates.
  - Categorized news catalysts (Commodities, Corporate actions, Index rebalancing, Macro politics).
- **v1.9**: **Live Price Anchor Injection & Target Surpassed Guard**:
  - Fixed inverted Target TP below live price bug on surging stocks (e.g. `UDNG` jumping +9.69% to Rp 1.585 while old TP was Rp 1.490).
  - Injected `fast_info` real-time price as anchor into `indicators["last_close"]` before LLM generation.
  - Validated `take_profit > live_price` and auto-converted surpassed targets to `HOLD` status ("Harga pasar live sudah melampaui area target resisten/TP sebelumnya") with potential calculated strictly against live real-time price.
- **v2.0**: **Saham AS (IBM) Multi-Currency P&L + Radar Makro Ekonomi + Konversi USD/IDR**:
  - Inserted IBM US stock position for `lufria` (Modal Rp 5.841.749, Entry $287.65, Qty 1.1237) totaling portfolio modal **Rp 417.141.439**.
  - Multi-currency portfolio engine: split P&L into Asset P&L (stock price change) vs Forex P&L (USD/IDR exchange rate change).
  - Endpoint `GET /api/market/macro`: Real-time USD/IDR rate, Gold XAU/USD ($/oz & Rp/gram), WTI Oil, US 10Y Yield, and Inflation news summary.
  - UI `MacroBar` + `UsdIdrConverter` widget on Dashboard.
- **v2.1**: **Broker App Portfolio Filtering**:
  - Dynamic portfolio filtering by application: `🇮🇩 Stockbit (Saham Indo: Rp 411,3 Jt)` vs `🇺🇸 Pluang (Saham AS: Rp 5,8 Jt)` with dynamic summary recalculations.
- **v2.2**: **Time-of-Day Strategy Auto-Assignment**:
  - Automatically tags signals as `BPJS` during morning/afternoon market hours (<14:00 WIB) and `BSJP` during late afternoon/weekend.
- **v2.3**: **Discord #advisory Auto-Purge Announcement Board**:
  - Performs automated bulk-delete of older messages before posting new briefings, keeping `#advisory` clean as a single live announcement board with 1 ping at the footer.
- **v2.4**: **Dual-Channel Discord Routing**:
  - `DISCORD_WEBHOOK_BPJS_URL` (`#bsjp-bpjs-advisory`, ID `1543114836769116200`): Dedicated single announcement bulletin board for daily BPJS (08:30 WIB) & BSJP (15:35 WIB) briefings with auto-purge before posting.
  - `DISCORD_WEBHOOK_URL` (`#advisory`, ID `1542883638289371146`): Dedicated stream for real-time 30-min market scan signals & emergency SL/TP alerts.
- **v2.5**: **Timezone Localization (GMT+7 / WIB) & 8-Pick Multi-Bubble Anti-Truncate Briefings**:
  - Converted all Google News RSS publication timestamps to GMT+7 (WIB) using `email.utils.parsedate_to_datetime` and Indonesian month abbreviations (`29 Agu 2026, 11:38 WIB`), and reduced cache TTL to 3 minutes for breaking news.
  - Expanded BPJS/BSJP daily recommendations to **8 top picks** (scanned from 50 top IDX stocks).
  - Implemented multi-bubble message chunking (max 4 picks per bubble, <1600 chars each) to prevent Discord's 2000-character truncation while maintaining a single mention ping on the final bubble.
- **v2.6**: **Newest-First Chronological News Sorting & Broader Indonesian National/Event Coverage**:
  - Strictly sorted all combined news feeds by timestamp descending (`Newest ➔ Oldest`).
  - Broadened Indonesian news queries to cover national breaking news, political events, legal/government policies, and social demonstrations.
- **v2.7**: **Direct Live Wire Breaking Feeds & Relative Time Ago Badges**:
  - Switched from search-based RSS to direct live news wire feeds (Antara News Terkini, CNBC Indonesia Market/News, Detikcom/Finance, CoinDesk, Decrypt) for true real-time breaking news published minutes ago.
  - Added relative time badges (`⚡ 5 menit lalu | 11:37 WIB`) and reduced cache TTL to 90 seconds.
- **v2.8**: **AI Summary & Market Potential Analysis on Every News Article**:
  - Added AI Summary (1-2 sentences), sentiment badges (🟢 Positif / 🔴 Negatif / ⚪ Netral), and potential sector impact on every news card with in-memory LLM summary cache.
- **v2.9**: **Executive All-in-One Market Digest & Ticker Linking**:
  - Endpoint `GET /api/news/digest` generating overall market sentiment score, top bullish catalysts, cautious stocks, and actionable strategy takeaway.
  - Rendered **All-in-One Executive Intelligence Panel** at the top of the news stream and tagged clickable `🏷️ Saham Terkait` badges on each article card.

## .env structure (gitignored)

```
# LLM Provider (OmniRouter self-hosted gateway on home server)
LLM_PROVIDER=omnirouter
LLM_BASE_URL=https://router.lufria.tech/v1
LLM_MODEL=auto/smart
LLM_API_KEY=sk-...

# Market Data
FINNHUB_API_KEY=                 # stocks primary (free tier 60 req/min)
TWELVEDATA_API_KEY=              # stocks fallback
SYMBOLS_CRYPTO=btcusdt,ethusdt,solusdt
SYMBOLS_STOCK=AAPL,TSLA,NVDA

# Database
DATABASE_URL=mysql+asyncmy://trading:trading_pass_2026@localhost:3306/trading_advisory

# Notifier
NOTIFY_TELEGRAM=true
TELEGRAM_BOT_TOKEN=              # reuse from /root/.hermes/.env
TELEGRAM_CHAT_ID=7864837597      # Hermes home channel / DM Lufria
NOTIFY_DISCORD=true
DISCORD_MENTION_ID=491905790966235136
DISCORD_WEBHOOK_URL=https://discord.com/api/webhooks/1542883917252792351/...
DISCORD_WEBHOOK_BPJS_URL=https://discord.com/api/webhooks/1543115212041756795/...

# App & Multi-User Auth
API_HOST=127.0.0.1
API_PORT=8001
AUTH_USERNAME=lufria
AUTH_PASSWORD=...
AUTH_USERS=lufria:pass1,aura:pass2
AUTH_SECRET=...
SESSION_HOURS=24
```
