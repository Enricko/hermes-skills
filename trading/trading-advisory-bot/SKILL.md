---
name: trading-advisory-bot
description: Operate AI trading advisory bot (BPJS, BSJP, Forex PnL).
---

# Trading Advisory Bot (Home Server)

Petunjuk arsitektur, konfigurasi, dan pengoperasian sistem **Trading Advisory Bot** (Advisory only, spot tanpa leverage, eksekusi manual via Stockbit/Pluang).

## 1. Arsitektur Inti
- **Backend**: Python 3.11, FastAPI (port 8001), SQLAlchemy Async (asyncmy), MySQL 8.4 (`trading_advisory`).
- **Frontend**: React + Vite + TypeScript (`/root/trading-advisory/web`), diserve statis di port 8001 (single port untuk API + UI via Cloudflare Tunnel `advisory.lufria.tech`).
- **Database**:
  - `signals` (id, symbol, asset_type, action, strategy, timeframe, entry_price, stop_loss, take_profit, confidence, reasoning, created_at)
  - `positions` (id, user_id, signal_id, symbol, asset_type, amount_idr, entry_price, quantity, stop_loss, take_profit, status, opened_at, closed_at, exit_price)
  - `user_pins` (id, user_id, symbol, asset_type, created_at)
  - `signal_outcomes` (id, signal_id, symbol, strategy, entry_price, target_tp, stop_loss, outcome, max_profit_pct, max_loss_pct, evaluated_at, notes)
  - `price_history` (symbol, ts, open, high, low, close, volume)
  - `watchlist` (id, symbol, asset_type, enabled, added_at, is_pinned)
  - `alerts_log` (position_id, type, triggered_at, notified_at)

## 2. Multi-User & Isolasi Akun
- Akun didukung via `AUTH_USERS` di `.env` (format: `user1:pass1,user2:pass2`).
- User default:
  - `lufria`: Portofolio Stockbit 8 saham (CUAN, BRPT, JARR, BLOG, KAEF, UDNG, ADMR, CDIA) + Saham AS (IBM).
  - `aura`: Portofolio Stockbit (CDIA, COAL, JARR).
- Portofolio dan Pin Watchlist 100% terisolasi per `user_id`, sementara data harga pasar, berita makro, dan sinyal AI dibagi bersama.
- Filter portofolio di web otomatis memisahkan ringkasan total modal & P&L per aplikasi (Stockbit vs Pluang).

## 3. Strategi Trading Harian (BPJS & BSJP) & Notifier Discord
- **BPJS (Beli Pagi, Jual Sore)**:
  - Sesi beli: Pre-Opening (08:45 WIB) / Open Sesi 1 (09:00 - 09:15 WIB).
  - Target jual: Sesi 2 (14:30 - 15:50 WIB). Target TP: +1.5% s/d +3.5% (2-5 ticks). SL: -1.0% s/d -2.0%.
  - Timer: `trading-bpjs.timer` (Mon-Fri 08:30 WIB) ➔ Mengirim 8 rekomendasi ke channel Discord `#bsjp-bpjs-advisory` (`1543114836769116200`).
- **BSJP (Beli Sore, Jual Pagi Besok)**:
  - Sesi beli: Pre-Closing (15:40 - 15:55 WIB).
  - Target jual: Open Spike Pagi Besok (09:00 - 09:15 WIB). Target TP: +2.0% s/d +4.0%. SL: -1.5% s/d -2.0%.
  - Timer: `trading-bsjp.timer` (Mon-Fri 15:35 WIB) ➔ Mengirim 8 rekomendasi ke channel Discord `#bsjp-bpjs-advisory` (`1543114836769116200`).
- **Channel Sinyal Real-Time & Alert SL/TP**: Channel `#advisory` (`1542883638289371146`).

## 4. Multi-Currency P&L (Saham AS & Forex USD/IDR)
- Untuk saham AS (seperti IBM) dan crypto dalam USD:
  - `current_value_idr = current_price_usd * quantity * usd_rate`
  - `unrealized_pnl = current_value_idr - amount_idr`
  - `asset_pnl_idr = (current_price_usd - entry_price_usd) * quantity * usd_rate` (P&L pergerakan saham)
  - `forex_pnl_idr = unrealized_pnl - asset_pnl_idr` (P&L selisih kurs USD/IDR)

## 5. Indikator Makro Ekonomi & Stream Berita Terkurasi
- Endpoint `GET /api/market/macro`:
  - `USDIDR=X`: Kurs Dolar ke Rupiah live.
  - `GC=F`: Harga Emas Dunia XAU/USD & konversi Rupiah/gram Antam.
  - `CL=F`: Minyak Mentah WTI.
  - `^TNX`: US 10-Year Bond Yield.
  - Sentimen Inflasi (US CPI & BI Rate).
- Stream Berita Real-Time:
  - Mengagregasi Stockbit Snips, Antara, CNBC Indonesia, Detik Finance, IDX Channel, Bloomberg Technoz, CoinDesk.
  - Anti-Dividen-Trap: Hanya mendeteksi dividen yang memiliki jadwal cum-date mendatang, menolak dividen kedaluwarsa.
  - Ringkasan Eksekutif All-in-One (`GET /api/news/digest`) + Tag Saham Terkait yang dapat diklik langsung ke chart.

## 6. Training Memory & Kalibrasi Akurasi
- Modul `app/llm/training_memory.py`:
  - Menyuntikkan pola sukses/gagal historis bursa Indonesia (Few-Shot Training Setups) ke prompt Gemini.
  - Melacak akurasi sinyal di tabel `signal_outcomes` via job `trading-evaluate.timer`.
  - Endpoint `GET /api/signals/accuracy` mengembalikan statistik Win Rate.

## 7. LLM Provider & Systemd Service Execution Pitfalls
- **Claude Code CLI di Systemd**:
  - Eksekusi `claude -p` wajib menyertakan flag `--tools ""` dan `--strict-mcp-config` agar tidak memicu loading MCP servers (seperti chrome-devtools-mcp) yang menyebabkan timeout/hang di background.
  - File unit systemd (`trading-bpjs.service`, `trading-bsjp.service`) wajib memiliki `Environment=HOME=/root` dan `Environment=USER=root`.
  - `ClaudeCodeProvider` wajib memiliki timeout ketat (6–12 detik) dengan fallback otomatis instan ke OmniRoute (`get_llm_provider()`) agar batch watchlist (25+ saham) selesai dalam <30 detik sebelum jam bursa buka.

## 8. Real-Time IDX Market Data & Frontend Auto-Refresh
- **0-Delay Feed via Dual Direct Live Engine (Google Finance + TradingView Scanner)**:
  - Yahoo Finance (`.JK`) memiliki delay publik bawaan 10–15 menit untuk bursa Indonesia (IDX).
  - Gunakan `TradingViewProvider` (`app/data/tradingview_provider.py`) yang menggabungkan:
    1. Direct Google Finance scrape (`https://www.google.com/finance/quote/{SYM}:IDX` matching initial state array `["SYM","IDX"],...,"IDR",[price,chg,chg_pct]`) dengan `follow_redirects=True`.
    2. TradingView Scanner (`POST https://scanner.tradingview.com/indonesia/scan` dengan payload `{"symbols": {"tickers": ["IDX:ADMR", ...]}}`).
  - `YahooFinanceProvider` berfungsi sebagai fallback ketiga jika kedua provider tidak merespons.
  - In-memory cache TTL diatur sangat singkat (5 detik, `_LIVE_CACHE_TTL = 5`) agar harga 100% sinkron dengan papan bursa & Stockbit.
  - Frontend `Portfolio.tsx` dan `DashboardPage.tsx` wajib menyertakan auto-refresh interval 3–5 detik agar P&L dan harga saham bergerak live tanpa perlu refresh browser manual.

## 9. Optimasi Performa & Beban Server Signal Engine (30m Batch)
- **Single-Pass Macro Ingestion**:
  - Berita makro (inflasi, emas, kurs rupiah, The Fed) dan snapshot makro di-fetch **1x saja di awal job** `signal_engine.run()`, lalu di-share ke semua analisis simbol (mencegah 150+ request RSS duplikat per batch).
- **Fast Pre-Screener Filter**:
  - Untuk saham non-pinned yang berada dalam fase konsolidasi datar/netral (RSI 40–60, jauh dari support/resisten), status `HOLD` langsung disimpan dalam hitungan <0.01 ms tanpa memanggil LLM.
  - LLM hanya dipanggil untuk saham yang memiliki momentum aktif / breakout / oversold dip / saham yang di-pin oleh user.
- **Async Concurrency & Provider Bypass**:
  - Gunakan `asyncio.gather` dibatasi `asyncio.Semaphore(10)` agar 60+ simbol selesai dalam <60 detik.
  - Provider Finnhub / TwelveData dengan API key kosong langsung dilewati (bypass) tanpa melakukan HTTP request 401 Unauthorized yang membuang waktu.

## 10. Format Notifikasi Discord Mobile-First & Anti-Cutoff
- **Format Mobile-First 3-Baris per Rekomendasi**:
  - Hindari paragraf panjang yang melelahkan di layar HP. Setiap item BUY wajib diformat ringkas dalam 3 baris:
    1. ⚡ **Simbol** (Confidence Badge)
    2. • Entry: **Harga** ➔ TP: **Harga** (🎯 **+X.X%**) | SL: **Harga** (🛑 **-X.X%**)
    3. • 💡 *1 kalimat inti reasoning teknikal/katalis (maksimal 110 karakter via `_clean_reasoning`).*
- **Pagination Bubble Seimbang (Maksimal 4 Item per Bubble)**:
  - Jika $\le 4$ rekomendasi: satukan dalam 1 bubble tunggal (~600 karakter, pas 1 layar HP).
  - Jika 5–8 rekomendasi: pecah menjadi 2 bubble seimbang (Part 1: 1–4, Part 2: 5–8) dengan heading lanjutan yang jelas.
  - Mencegah pemotongan batas karakter Discord (2000 char) dan mengeliminasi scroll fatigue pada pengguna mobile.
  - Selalu panggil `purge_channel()` sebelum mem-post batch baru agar channel `#advisory` / `#bsjp-bpjs-advisory` selalu bersih seperti papan pengumuman live.
