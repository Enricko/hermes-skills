# Saweria Top-Up, Webhook & Real-Time Alert Integration for HematCuy

## Core Objective
Integrates the Saweria donation & supporter platform (`https://saweria.co/Lufria`) into HematCuy for server funding and live community interaction.

## Webhook & Real-Time Event Architecture
1. **Webhook URL:** `https://hematcuy.lufria.tech/api/v1/saweria/webhook` (configured in Saweria Creator Settings).
   - Exempt from HMAC middleware checks to accept raw Saweria notifications 24/7.
   - Extracts `donator_name`, `amount`, `message`, `saweria_id`.
   - Saves to MariaDB `donations` table.
   - Publishes JSON event payload to Redis channel `saweria:events`.
2. **Server-Sent Events (SSE):** `GET /api/v1/saweria/events`
   - Real-time event stream connected to all active website visitors.
   - Pushes `event: donation` whenever a new sawer notification is received.
3. **Frontend Alert Toast Queue (`SaweriaAlertToast.jsx`):**
   - Receives live donation events from the SSE stream.
   - Enqueues notifications and displays an alert toast for 6 seconds.
   - **Mandatory Interval / Jeda:** Leaves a 2-second pause before displaying the next queued alert so multiple incoming donations don't flood the UI.
4. **Leaderboard & Recent API:**
   - `GET /api/v1/saweria/leaderboard` — groups top supporters by `donator_name` (SUM `amount`), cached in Redis for 3 minutes.
   - `GET /api/v1/saweria/recent` — returns the last 20 donations.

## UI Principles (User Preference — Strict)
- **NO custom form inputs on the website:** Do not put custom text fields or payment inputs in `TopupModal.jsx`.
- **Direct Saweria Redirection:** Present clean, clickable tier cards with direct links opening `https://saweria.co/Lufria` in a new tab where users complete payment via QRIS, GoPay, OVO, DANA, ShopeePay, or LinkAja.
- **Embedded Leaderboard & History:** The modal features interactive tabs: `Paket Top Up`, `Leaderboard` (🥇 🥈 🥉), and `Terkini`.

## Supporter Tiers
- ☕ **Kopi Sachet (Rp 5.000):** Traktir kopi developer.
- 🔥 **Supporter Mart (Rp 15.000):** Dukung biaya server & proxy bypass.
- 💎 **HematCuy VIP (Rp 30.000):** Role VIP di Discord & prioritas request gerai baru.
- 👑 **Sultan Ritel (Rp 50.000+):** Dukungan penuh kelangsungan agregator harga.
