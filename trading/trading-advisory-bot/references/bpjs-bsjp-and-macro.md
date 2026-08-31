# BPJS, BSJP, and Macro News Intelligence System

Panduan detail operasional strategi BPJS/BSJP, jadwal briefing bursa, dan integrasi data makro.

## 1. Jadwal Briefing & Channel Discord
- **BPJS (Beli Pagi, Jual Sore)**:
  - Eksekusi: Senin - Jumat pukul 08:30 WIB (sebelum sesi Pre-Opening 08:45 WIB).
  - Target: Beli sesi 1 (09:00 - 09:15 WIB), pasang antrean Take Profit di sesi 2 (14:30 - 15:50 WIB).
  - Target TP: +1.5% s/d +3.5% (2-5 ticks IDX), Stop Loss ketat -1.0% s/d -2.0%.
  - Channel: `#bsjp-bpjs-advisory` (`1543114836769116200`).
  - Mekanisme: Purge pesan lama di channel -> kirim 8 rekomendasi dalam 2 bubble (Bubble 1 no ping, Bubble 2 + 1 ping mention di akhir).
- **BSJP (Beli Sore, Jual Pagi Besok)**:
  - Eksekusi: Senin - Jumat pukul 15:35 WIB (menjelang Pre-Closing 15:45 - 15:55 WIB).
  - Target: Beli di closing auction / sesi sore, pasang antrean jual di opening spike pagi besok (09:00 - 09:15 WIB).
  - Target TP: +2.0% s/d +4.0% (3-6 ticks), Stop Loss -1.5% s/d -2.0%.
  - Channel: `#bsjp-bpjs-advisory` (`1543114836769116200`).
  - Mekanisme: Sama dengan BPJS (8 rekomendasi, auto-purge).

## 2. Anti-Dividen-Trap Guard
- Jangan pernah merekomendasikan sinyal BUY berbasis dividen jika tanggal cum-date sudah lewat (sudah masuk masa ex-date / pembayaran).
- Filter berita: hanya beri tag `[Dividen]` jika artikel berusia < 3 hari dan secara eksplisit menyebut jadwal cum-date mendatang.
- Saham yang mendekati resistance dengan RSI > 70 pasca cum-date wajib diberi status HOLD untuk mencegah dividend trap.

## 3. Indikator Makro Ekonomi
- `USDIDR=X`: Kurs Dolar ke Rupiah live (sensitivitas emiten importir vs eksportir/komoditas).
- `GC=F`: Harga Emas Dunia XAU/USD & konversi Antam (katalis saham ANTM, PSAB, MDKA).
- `CL=F`: Minyak Mentah WTI (katalis saham MEDC, ELSA, ENRG).
- `^TNX`: Yield Obligasi US 10-Year & Inflasi US CPI / BI Rate.

## 4. Pitfalls Background CLI Execution
- Background CLI (Claude Code): Gunakan flag `--tools ""` dan `--strict-mcp-config` agar MCP tidak spawn di headless background.
- Systemd environment: Pastikan `HOME=/root` terdefinisi di `.service`.
- Timeout failover: `ClaudeCodeProvider` harus failover cepat (<=12s) ke OmniRoute Gemini agar briefing BPJS (08:30 WIB) dan BSJP (15:35 WIB) tidak terlambat diposting ke Discord.
