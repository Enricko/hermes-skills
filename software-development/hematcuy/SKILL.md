---
name: hematcuy
description: Use when developing or maintaining the HematCuy aggregator.
version: 1.0.0
author: Hermes session learnings
license: MIT
platforms: [linux]
metadata:
  hermes:
    tags: [hematcuy, retail, ingestion, scrape, deploy]
---

# HematCuy — Universal Price & Deal Aggregator

## When to Use
Any work on `/root/hematcuy` — backend FastAPI (:8002), frontend React/Vite, MariaDB `hematcuy`, Redis (:6379), systemd `hematcuy.service`, Cloudflare Tunnel `hematcuy.lufria.tech`. Covers ingestion workers, redirect logic, cross-store price comparison, and the data-source quirks learned the hard way.

## Key Data Sources (Indonesian retail)
- **hemat.id — authoritative e-katalog / flyer digitizer.**
  - Weekly flyers: `https://www.hemat.id/katalog/indomaret/`, `https://www.hemat.id/katalog/alfamart/`. 200 OK, no WAF, structured HTML: `img alt="Promo Harga ..."` + sibling text with `Rp` prices. Parse with BeautifulSoup.
  - Per-item detail pages: `https://www.hemat.id/harga/{slug}/` — contain per-retailer price tables. Section heading pattern: `Harga <item> di <Retailer> Selengkapnya`, followed by a `<table>` with rows labeled `Harga hari ini / terendah / tertinggi`. Scrape these on demand for cross-store comparison (see `backend/app/api/v1/comparison.py`).
  - No `X-Frame-Options` → embeddable in iframe, BUT always ship an escape hatch ("Buka di tab baru") because a JS frame-buster can leave a blank box.
- **KlikIndomaret online** — 403 on datacenter IPs; long search keywords can trigger the **21+ age-verification modal** (heated-tobacco products like TEREA pollute broad results). Fix: use SHORT clean keywords (e.g. `Tissue Indomaret`, `Minyak Bimoli 2L`).
- **Alfagift** — geofenced quick-commerce: stock tied to nearest store, so "Produk tidak tersedia di area Anda" is normal. Online prices ≠ flyer prices (flyer = cashier promo, online = regular shelf price).

## Price Semantics: Katalog vs Online (user expectation — do not skip)
- **Katalog** (`promo_type='instore_catalog'`, amber tag) = flyer/catalog price, valid at the physical cashier. NOT the online price.
- **Online** (`promo_type='online'`, emerald tag) = price at the online store, buyable directly.
- Users WILL compare HematCuy price against online stores and complain about mismatch. Always tag clearly and redirect to the place where the SHOWN price actually applies:
  - Katalog item WITH per-item hemat.id detail URL → redirect to that detail page (price visible there).
  - Katalog without detail → store e-katalog page `https://www.hemat.id/katalog/{store}/`.
  - Online → official store product page.
- Card price is ONE retailer's price, not the market's cheapest → "Bandingkan Semua Mart" button opens the comparison modal.

## Redirect Logic & Mobile App Deep-Linking (`backend/app/api/v1/redirect.py` & `frontend/src/utils/appLinks.js`)
- `_get_clean_redirect_url(listing, target=None)`
- **App Deep-Linking (`target='app'` / `getAppDeepLink()` in `frontend/src/utils/appLinks.js`):**
  - **Klik Indomaret:** Official Android package is **`edts.klik.android`** (verified via `www.klikindomaret.com/.well-known/assetlinks.json`). Android Intent scheme:
    `intent://www.klikindomaret.com/search/?key={query}#Intent;scheme=https;package=edts.klik.android;S.browser_fallback_url={encoded_web_search};end;`
    *Chrome Android Quirk:* Chrome blocks `intent://` navigation if delivered via HTTP 307 server redirects. Always fire the intent directly from the frontend `<a>` tag or client-side click handler via `getAppDeepLink()`.
  - **Alfagift (Alfamart):** Universal Link / Android Intent: `intent://alfagift.id/find/{query}#Intent;scheme=https;package=com.alfamart.alfagift;S.browser_fallback_url=https%3A%2F%2Falfagift.id%2Ffind%2F{query};end;` (intercepted by Alfagift app on Android/iOS).
  - **Steam Mobile:** `steam://store/{external_id}`.
  - **Shopee & Tokopedia:** Direct search queries `https://shopee.co.id/search?keyword={query}` and `https://tokopedia.com/search?st=product&q={query}`.
- `target='gomart'` → `build_gomart_search_url(title)` (strips retailer prefix words, keeps first 4 words, url-encoded).
- indomaret / klikindomaret → `affiliate_url or product_url` or fallback `https://www.hemat.id/katalog/indomaret/`.
- alfagift → `affiliate_url or product_url` (real alfagift.id link) or fallback `https://www.hemat.id/katalog/alfamart/`.
- other stores (Steam, Epic, etc.) → `affiliate_url or product_url`.

## Retail Pricing Mechanics: Webstore vs Mobile App vs Cashier
- **Why webstore prices differ from mobile apps & flyers:**
  - Desktop webstores (e.g. `klikindomaret.com`) default to standard central warehouse shelf prices unless the user logs in and manually selects a local store branch.
  - Mobile apps (Klik Indomaret & Alfagift) use smartphone GPS to automatically detect the nearest physical branch, unlocking in-store flyer promo prices, member Poinku discounts, and flash sales.
  - HematCuy catalog prices reflect in-store flyer and mobile app promotions (tagged `Katalog`, amber badge). Price comparison modals feature a quick `[App 📱]` button so mobile users can jump directly into the retailer's app for checkout.

## Price Comparison Endpoint & Pitfalls
- `GET /api/v1/products/{id}/comparison` — scrapes the hemat.id item page on demand, parses per-retailer tables → `stores[]` with today/low/high + `price_available`. GoMart/Shopee/Tokopedia are **link-only** (`price=None`) — never invent prices, they render client-side. Redis TTL 1800s.
- Response also carries `online_available` + `online_url`: set when an official webstore link (klikindomaret.com / alfagift.id) is found on the item page ("Beli Online" button or bare anchor). Currently most hemat.id item pages have NO webstore link → flag is honestly false; do not fabricate it.
- **DOM-Crossing Link Trap (Solved):** NEVER use `heading.find_next("a")` when extracting retailer catalog links. When a retailer table has no link, `find_next("a")` jumps across the DOM into unrelated recommendation carousels or articles (e.g. attaching `https://www.hemat.id/harga/minyak-goreng/` to a milk/tissue product). Always search only within `heading.find("a")` or `table.find("a")`. If absent, fallback cleanly to `STORE_CATALOG_FALLBACK.get(slug)` (`https://www.hemat.id/katalog/{slug}/`) or `source_url`.
- **Carton & Multi-pack Outlier Filtering:** In hemat.id historical tables, bulk carton prices (e.g. Rp 99.000 for a 40-pcs carton) are occasionally logged in single-unit item tables (e.g. 115ml milk Rp 2.650). Filter out any price `> 4 * base_price` to prevent misleading "-97% discount" claims and corrupted ranges where `low > high`.
- **Product Photo vs Banner Distinction:** `serverless-image-op-0` is the authentic product photo CDN on hemat.id and must be kept. Do NOT replace it with `soup.find("img")` which grabs the WhatsApp community banner (`/artikel/`, `WAG-Komunitas-Sahabat-Hemat-1.png`).
- Frontend modal: `frontend/src/components/PriceComparisonModal.jsx` — tab "Harga per Mart" (table with per-row "Beli" button when `online_url` exists, "Katalog" link otherwise) + tab "E-Katalog" (iframe embed). **E-Katalog tab defaults to ITEM DETAIL (`katalogView='item'` → `source_url`), NOT the weekly flyer** — flyer is only a fallback when no detail page exists.

## Image Pipeline & CDN Hotlink Protection (audited & fixed 2026-08-30)
- **Root causes of blank photos (Solved):**
  1. *Missing URL gap:* the cross-store staples worker reads hemat.id price TABLES whose rows carry NO thumbnail — only a link to the item detail page. Solved via `backfill_images.py` matching `product_url LIKE '%hemat.id/%'` to extract `og:image`.
  2. *Hotlink Protection (403 Forbidden):* upstream grocery CDNs (`img.hemat.id`, `nos.wjv-1.neo.id`, `nos.jkt-1.neo.id`) return `403 Forbidden` if request has no `Referer` header. Solved by injecting `Referer: https://www.hemat.id/`.
  3. *Ephemeral S3 hashes (`serverless-image-op-0`):* temporary cache hashes on Biznet/Neo S3 expire after ~1-2 weeks returning 403 Access Denied. Solved by resolving permanent `img.hemat.id` URLs and `og:image` from live product pages.
  4. *Steam thumbnail subfolder hashes:* Steam search HTML capsule thumbnails use dynamic hash paths (`.../apps/{appid}/{hash}/capsule_231x87.jpg`). Naive replacement of `capsule_231x87` to `header.jpg` yields a 404 because `header.jpg` does not exist under that subfolder hash. Use the exact `img src` from the search row or fetch `og:image` (`capsule_616x353`) directly from the store page.
- **Proxy Architecture (`backend/app/api/v1/images.py`):**
  - Route `@router.api_route("/proxy", methods=["GET", "HEAD"])` supports both `GET` and `HEAD` requests.
  - Dynamically injects `Referer` based on upstream host (`https://www.hemat.id/` for `hemat.id` and `neo.id`, `https://store.steampowered.com/` for Steam, `https://store.epicgames.com/` for Epic).
  - Follows upstream 301 redirects safely with SSRF verification (`is_safe_image_url()`), 4MB payload cap, and binary Redis cache `img:{sha256}` TTL 7d.
  - Frontend routes all external product photos through `proxiedImage(url)` (`frontend/src/utils/images.js`).

## Advanced Search & Filter (Steam-Workshop Sidebar & Mobile Drawer)
- **Backend `search.py` params:** `q`, `categories`/`category` (comma-separated multi), `stores`/`store` (comma-separated multi), `promo_type` (`instore_catalog`|`online`), `is_free`, `min_discount`, `min_price`/`max_price` (bounds Product.current_lowest_price; when store(s) pinned, also bounds those listings' current_price), `sort_by` (relevance|price_asc|price_desc|discount_desc|newest), pagination. Cache key includes all filters.
- **Granular 12-Vertical Taxonomy:** Products are classified into 12 granular categories: `beras-biji` (rice_bowl), `minyak-bumbu` (oil_barrel), `susu-dairy` (water_drop), `mie-instan` (ramen_dining), `daging-segar` (set_meal), `minuman-kopi-teh` (local_cafe), `snack-biskuit` (cookie), `kebersihan-rumah` (cleaning_services), `perawatan-tubuh` (soap), `ibu-bayi` (child_care), `kesehatan-farmasi` (medication), `gaming` (sports_esports), `sembako` (shopping_basket).
- **Taxonomy endpoints & A-Z Sorting:** `GET /api/v1/categories` & `/api/v1/stores` back the filter dropdowns (only non-empty taxonomy, Redis TTL 600). Backend queries MUST sort alphabetically (`Category.name.asc()` and `Store.name.asc()`) and frontend components (`FilterSidebar.jsx` & `FilterBar.jsx`) MUST apply `.sort((a, b) => a.name.localeCompare(b.name, 'id'))` so categories and stores are cleanly ordered A-Z.
- **Frontend Architecture (`FilterSidebar.jsx`):** Sticky `w-64` left sidebar on desktop, slide-in drawer (`w-84 max-w-[88vw]`) with backdrop blur on mobile. Sections:
  1. Lokasi Belanja & Wilayah (modal trigger).
  2. Kategori Barang (multi-select chip grid with icons and live product counts).
  3. Jaringan Gerai & Toko (multi-select store badges with live listing counts).
  4. Jenis Promo & Diskon (100% Gratis, Diskon 50%+, Diskon 30%+, Kasir & App, Online).
  5. Rentang Harga (Min/Max inputs + quick presets `< 25rb`, `25rb – 50rb`, `50rb – 100rb`, `> 100rb`).
  6. Mobile Sticky Action Button: `Terapkan Filter (N Produk)` at the bottom of the drawer.
- **REAL-TIME filter pitfall:** filters must apply on EVERY tab. Route through `/api/v1/search` with empty `q` whenever any filter is active (`activeFilterCount > 0`), not just when `searchTerm` is present — otherwise the "all" tab ignores filters and feels broken. Effect deps include the whole `filters` object.
- **Ad placement frequency:** in-feed ads render every 8 cards in grid (`(index+1) % 8 === 0`) and every 10 rows in table (`(index+1) % 10 === 0`) — user wants recurring ads, not a single one.
- **Community Support & Live Saweria Integration (`backend/app/api/v1/saweria.py` & `frontend/src/components/`):**
  - **Webhook URL:** `https://hematcuy.lufria.tech/api/v1/saweria/webhook` (exempt from HMAC checks; saves to MariaDB `donations` and publishes to Redis `saweria:events`).
  - **Saweria Test Webhook Idempotency Bypass:** The Saweria admin "Munculkan Notifikasi" test button always sends fixed dummy ID `00000000-0000-0000-0000-000000000000`. Detect this dummy ID and generate dynamic test IDs (`test_{timestamp}`) so test notifications trigger DB inserts, leaderboard updates, and real-time SSE broadcasts without being blocked as duplicate.
  - **Real-Time SSE Alert (`SaweriaAlertToast.jsx`):** Listens to `/api/v1/saweria/events`, enqueues incoming donations, and displays a toast for 6s with a mandatory **2-second interval jeda** between queued alerts.
  - **Leaderboard & Recent APIs:** `/api/v1/saweria/leaderboard` (top supporters) and `/api/v1/saweria/recent` (latest 20 donations).
  - **UI Convention (Strict):** NO form inputs on the website. `TopupModal.jsx` displays clean tier cards and directly opens `https://saweria.co/Lufria` in Saweria where payment is processed via QRIS/E-Wallets. See `references/saweria-topup-integration.md`.

## Ingestion Workers & Scaled Catalog Ingestion
- **Full Store Category Ingestion (`ingest_full_stores.py`):** Ingests all 42 product categories from `https://www.hemat.id/harga/` (Beras, Susu UHT, Minyak Goreng, Deterjen, Shampoo, Roti, Kopi, Sabun Mandi, dll.). Scrapes each item detail page (`/harga/{slug}/`) and extracts per-retailer price tables across 16 national chains, scaling the catalog to 1,400+ products and 3,200+ store listings.
  - *Database Unique Key Safety:* Store listings enforce unique `(store_id, external_id)`. Generate `external_id = f"{store_slug}_{canonical_id[:16]}"` and always query `StoreListing.store_id == store.id, StoreListing.external_id == external_id` before insert, wrapped with transactional `db.commit()` and `db.rollback()` per item.
  - *Zero-Price Bug Prevention (Threshold > Rp 500):* When scraping per-retailer price tables from hemat.id, blank cells or placeholder dashes (`"—"`) can accidentally parse as `0`, corrupting `current_lowest_price` to `Rp 0`. Always enforce a positive threshold (`val > 500 IDR`) for grocery retail rows. Only genuine digital game freebies (`is_free=True`) are allowed to have `price == 0`. Orphaned products with 0 listings must be purged (`~Product.listings.any()`).
- **Steam Franchise Deep-Scanning & Dynamic CDN Paths (`steam_worker.py`):**
  - Scrapes 35+ pages of Steam Specials search and explicitly scans top 50 famous franchises (Alan Wake, Cyberpunk, Witcher, GTA, Resident Evil, etc.) to capture publisher sale discounts up to -90% that are missed by default relevance sort.
  - *Dynamic Fastly Hashes:* Modern Steam uses subfolder hashes on `shared.fastly.steamstatic.com`. Always extract `img.src` from `row.find('img')` on the search row; do not hardcode static `header.jpg` paths which return 404 on newer Steam titles.
- **DLC & Add-on Content Tagging (`schemas/deal.py`, `DealCard.jsx`, `DealRow.jsx`):**
  - Comprehensive regex pattern (`DLC_REGEX` / `DLC_CLIENT_REGEX`): `\b(dlc|expansion|soundtrack|soundtracks|season pass|battle pass|expansion pack|add-on|addon|addons|content pack|bonus pack|character pack|skin pack|upgrade pack|artbook|ost|deluxe upgrade|weapon pack|costume|suite|themes?|music|audio|sounds?|packs?|bundles?|pass|presets?)\b`.
  - Catches standalone theme packs, music suites (e.g. `Microsoft Flight Simulator Suite: Themes Reimagined`), soundtrack OSTs, and expansion DLCs.
  - Automatically flags `is_dlc = True` in Pydantic models and renders a prominent purple `[ 🎮 DLC ]` / `[ Add-on ]` badge on cards and list rows on both backend and frontend layers.
- **Graceful Image Proxy Fallback (`api/v1/images.py`):**
  - When upstream third-party CDNs fail with 404/403/502, `proxy_image` returns an inline SVG vector placeholder (`FALLBACK_SVG`) instead of throwing 502, preventing broken image icons on the frontend.
- **Supermarket & Health/Beauty Flyers (`supermarket_flyers_worker.py`):** Ingests weekly flyer promotions across 9 national supermarket & drugstore chains: `superindo`, `hypermart`, `alfamidi`, `lottemart`, `tiptop`, `yogya`, `watsons`, `guardian`, `harihari`. Ingestion maps personal care (`shampoo`, `sabun`, `skincare`) to `perawatan`, cleaning supplies to `kebersihan`, and staples to `sembako`.
- **Epic Games Freebies vs Discounts (`epic_worker.py`):** In Epic's promo API, regular percentage discounts (e.g. 50% off) carry `discountSetting: { discountType: 'PERCENTAGE', discountPercentage: 50 }`. A true 100% freebie MUST have `current_price == 0` AND `discountPercentage == 0` with `originalPrice > 0`. DO NOT treat all `discountType == 'PERCENTAGE'` as freebies.
- **Pagination & Load More (`App.jsx` & `search.py`):** Backend `/api/v1/search` provides unified pagination (`page`, `page_size`, `total`, `total_pages`). Frontend uses batch size 24 (`PAGE_SIZE = 24`) and appends newly loaded items (`setDeals(prev => [...prev, ...newItems])`) with a dedicated "Muat Lebih Banyak" button showing active and total counts (`{deals.length} dari {totalCount} produk`).
- **Image Backfill (`backfill_images.py`):** Scans all products with empty image URLs against `product_url LIKE '%hemat.id/%'` (both `/katalog/` and `/harga/` routes) to extract `og:image`, ensuring 100% thumbnail coverage across all supermarket listings.

## Mobile UX & Responsive Layout Standards (User-Tested)
- **Grid Layout on Mobile:** Always use 2 columns (`grid-cols-2 gap-2 sm:gap-3 sm:grid-cols-2 md:grid-cols-3 lg:grid-cols-4`) with compact image aspect ratio (`aspect-[4/3]`) and tight typography (`text-xs sm:text-sm`). Single-column mobile grids produce gigantic cards that take up the whole screen and ruin browsing.
- **Card CTA Hierarchy & Universal Button Alignment:** 
  - Every card across ALL categories MUST follow the exact same two-button layout for visual harmony:
    `[ Kiri: Ikon Tool Pendamping ]  [ Kanan: Tombol Utama Hijau ]`
  - **Sembako / Ritel:** Left is the compact secondary button (`[📖]` `menu_book` flyer, `p-2 bg-slate-100 text-slate-700`), and Right is the primary large CTA **"Bandingkan Harga"** (`flex-1 bg-emerald-600 text-white font-bold`).
  - **Game & Digital Deals:** Left is `[📊]` `monitoring` price history icon button, and Right is the primary large CTA **"Beli" / "Klaim"** (`flex-1 bg-emerald-600 text-white font-bold`).
- **Fixed Height & Symmetrical Grid Alignment:** Always give product titles a fixed 2-line clamped height (`h-8 sm:h-9 line-clamp-2 leading-snug`). This guarantees the price row and action buttons align 100% straight across all grid rows, eliminating jagged bottoms.
- **URL State Sync Pattern (Zero Reload & No SessionStorage):** Every filter change (search `?q=`, category `?cat=`, store `?store=`, tab `?tab=`, sort `?sort=`, price `?min=&max=`, page `?p=`) MUST synchronize directly to the browser URL via `window.history.replaceState` WITHOUT reloading the page. On mount and on browser `popstate` (Back/Forward), parse `window.location.search` so filters persist on reload/reopen and remain shareable. DO NOT store filter state in `sessionStorage`.
- **Image Thumbnail Cleanliness (Anti-Clutter):** NEVER put bulky store badges (e.g. wide white "Steam Store" pills) over product artwork/thumbnails — it covers the game/product titles. Keep thumbnail images clean with only minimal corner badges (`-95%` / `GRATIS`); place store pills and tags in the metadata row below the image.
- **List / Table Mode on Mobile (Zero Horizontal Scroll):** Never render a raw `<table>` with `overflow-x-auto` on mobile — it pushes the "Harga Termurah" and "Aksi" columns offscreen to the right, forcing users to swipe horizontally. Use responsive flex rows (`divide-y divide-slate-100`) with left thumbnail (56×56px), middle title + store badge, and right-aligned bold price + action buttons that fit 100% within the mobile viewport.
- **Settings Modal Design:** Do NOT duplicate store filter checkboxes in the settings modal when the Advanced Filter Sidebar/Drawer already handles store and category selection. Keep Settings Modal focused on 18+ safe search/blur toggles, default view mode (grid vs list), system info, and Saweria donation support.
- **Viewport & Scroll Area:** Keep header/navbar compact on mobile (sleek search bar, compact category tabs). Avoid intrusive sticky mobile overlays that consume vertical height. Always give `<main>` ample bottom padding (`pb-28 md:pb-12`) so list items scroll comfortably above the mobile `BottomNav`.

## Deploy / Build / Verify
1. `cd /root/hematcuy/frontend && /root/.hermes/node/bin/npm run build`
2. `/root/hematcuy/backend/.venv/bin/pytest backend/tests -v` (expect 28: API + grocery routing/comparison)
3. `redis-cli flushall` (DB writes must invalidate `search:*`/`deals:*` cache)
4. `systemctl restart hematcuy.service`
5. Verify: `curl -sI https://hematcuy.lufria.tech/` → 200; `curl -sI "/api/v1/redirect?listing_id=N"` → 307 + correct `location:`.

## Monetization & AdSense Setup
- **Publisher ID (CURRENT):** `ca-pub-7799061692166115` — updated 2026-08-29 (old one was `ca-pub-2151403616648672`, do not reuse).
- **`ads.txt`:** store in `/root/hematcuy/frontend/public/ads.txt` (Vite copies `public/` → `dist/` root; FastAPI SPA static routing serves `/ads.txt`). Content: `google.com, pub-7799061692166115, DIRECT, f08c47fec0942fa0`. Sync the same line to `/root/lufria-profile/public/ads.txt` (apex `https://lufria.tech/ads.txt`) — both domains verified.
- **index.html head (BOTH projects):** `<meta name="google-adsense-account" content="ca-pub-7799061692166115">` + async `<script src="https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js?client=ca-pub-7799061692166115" crossorigin="anonymous"></script>`.
- **Ad units (created in dashboard, wired in `frontend/src/components/AdSlot.jsx` `DEFAULT_SLOTS`):**
  - `in-feed` → slot `5237542431`, `data-ad-format="fluid"`, layout key `-6j+cv+2x-18+2v` (inside grid every 8 cards / table every 10 rows).
  - `under-chart` → slot `9176787446`, `data-ad-format="auto"` + `data-full-width-responsive="true"` (below charts in modals).
  - `sticky-mobile` → anchor slot, not created yet (slot id empty).
- **AdSlot.jsx pattern:** `DEFAULT_SLOTS` map per slotType + `activeSlot = slotId || DEFAULT_SLOTS[slotType]`; only `(window.adsbygoogle || []).push({})` when a slot is active (avoids console errors before units are approved). Placeholder promo card renders when no slot id.
- **AdSense account quirks:** account created via YouTube Partner hides "Sites" in sidebar — use `https://adsense.google.com/adsense/u/0/sites`. Google consent message (GDPR): choose the 3-option CMP ("Izinkan, Jangan izinkan, Kelola opsi"). Each subdomain must be added as its own site; "Sepertinya Anda sudah menambahkan situs ini" is harmless — ads.txt/meta detection covers it.
- **Ad slot IDs come from the user's dashboard** — whenever they paste a unit snippet, extract `data-ad-slot` and the in-feed `data-ad-layout-key`, update `DEFAULT_SLOTS` + `IN_FEED_LAYOUT_KEY`, rebuild, restart.

## Pitfalls
- **Claude Code often edits files between Hermes turns.** Re-read before writing; the patch/write tools warn "modified since last read". If you already overwrote a Claude Code version, recover it from the Claude Code session JSONL — see `references/claude-code-jsonl-recovery.md`. The recovered version is usually smarter than a naive rewrite (per-item detail priority, affiliate priority, `target=` param), and the routing tests import its functions directly.
- **Claude Code invocation on this project:** write the brief to `/tmp/*.md`, run `claude -p "$(cat /tmp/brief.md)" --allowedTools "Read,Edit,Write,Bash,Grep,Glob" --fallback-model haiku --max-turns 80` in the **background with notify_on_complete** (long runs exceed foreground timeouts). Exit code 0 + `Error: Reached max turns (80)` = PARTIAL work: audit file mtimes for deliverables (e.g. `backend/backfill_images.py`, `frontend/src/components/FilterSidebar.jsx`) and finish the rest yourself, then re-run pytest.
- Worker sync: `backend/sync_deals.py`; scheduler = APScheduler (daily ~06:00 WIB). `ingestion_runs` / `catalog_editions` tables track audit.
- Indomaret/Alfamart DB rows may carry `product_url` = either generic katalog URL or a per-item hemat.id detail URL — redirect logic must respect the difference (don't hardcode everything to the generic katalog).
