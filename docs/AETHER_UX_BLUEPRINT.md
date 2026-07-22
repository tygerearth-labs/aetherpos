# AETHER UX BLUEPRINT v1.0

> **Status**: DRAFT — for review before POS pilot redesign
> **Owner**: Product (Z.ai) + Founder (Ahtjong)
> **Predecessor**: `docs/CHECKPOINT-PHASE-0.5.md` (platform declared stable)
> **Successor**: POS Pilot Redesign (only after this blueprint is approved)
> **Mandate**: Make Aether easy for humans to understand. **No new audits.**
> **Hard constraint**: Do NOT touch core, sync, FEFO, HPP, or consumption engine.

---

## 0. Reading Order

This document follows the agreed sequence:

1. Business Mode
2. User Role
3. User Intent
4. First-Time Journey
5. Daily Operational Journey
6. Navigation
7. Page Guidance
8. System Feedback
9. (Pilot) POS Redesign Principles

Each section ends with **"Design Implications"** — concrete rules the redesign must obey.

---

## 1. Business Mode

### 1.1 What Aether actually is (one paragraph)

Aether is a **point-of-sale and inventory platform** for Indonesian small-to-medium businesses. It is **single-tenant per outlet** at the Free tier and **multi-outlet per group** at Pro/Enterprise. The unit of work is **one outlet** doing four things every day: **sell, restock, count, and review**. Every feature in the app must trace back to one of those four verbs.

### 1.2 The four operational verbs

| Verb | What it means | Where it lives in the app |
|------|---------------|---------------------------|
| **Jual (Sell)** | Take money from a customer in exchange for goods | POS → Transaction |
| **Beli (Restock)** | Acquire goods from a supplier, increasing inventory | Purchase → InventoryBatch |
| **Hitung (Count)** | Verify what is actually on the shelf vs what the system thinks | Stock Opname → InventoryBatch adjustment |
| **Lihat (Review)** | Understand what happened and decide what to do next | Dashboard, Reports, Audit Log, Insights |

> Every menu item, every modal, every empty state must answer: **which verb is this helping with?** If none — cut it.

### 1.3 Business modes (industry presets — NOT yet implemented; proposed)

Today Aether is industry-agnostic at the schema level: every outlet has the same fields. But the *vocabulary* an F&B coffee shop uses ("menu", "resep", "expiring soon") differs from retail ("barcode", "SKU", "restock") which differs from services ("layan", "terapis", "slot"). The blueprint proposes **industry presets** as a UX-only layer — same data model, different vocabulary and feature emphasis.

| Mode | Who it's for | Vocabulary shift | Featured modules | Suppressed modules |
|------|--------------|------------------|------------------|--------------------|
| **F&B / Kopi / Resto** | Coffee shops, milk bars, warung makan, bakeries | "Produk" → "Menu", "Stok" → "Bahan", "Pelanggan" → "Tetap" | POS (table mode optional), Pembelian (bahan baku), Freshness/FEFO, Waste Report | Transfer (rare), Stock Opname (monthly) |
| **Retail / Minimarket** | Minimarkets, toko sembako, butik, toko elektronik | "Produk" stays, "Barcode" first-class, "SKU" first-class | Products (barcode print), Stock Opname (weekly), Transfer (between branches) | Freshness/FEFO (off by default), Waste Report |
| **Jasa / Service** | Barbershop, salon, laundry, bengkel | "Produk" → "Layanan", "Stok" → off, "Pelanggan" → "Klien" | POS (slot/therapist optional), Pelanggan (loyalty heavy), Transaksi | Pembelian (off), Stock Opname (off), Freshness (off) |
| **Hybrid (default)** | Most SMBs that don't fit one box | Standard Indonesian POS vocabulary | All modules | None |

**Design Implications**:
- Mode is selected at onboarding (or in Settings) and stored on `Outlet` (e.g., a new field `industryMode: 'fnb' | 'retail' | 'service' | 'hybrid'` — schema change, but additive, nullable, defaults to `'hybrid'`).
- Mode **never** hides data or breaks core. It only changes: (a) labels in the UI, (b) which module appears first in the sidebar, (c) which dashboard cards are emphasized, (d) which empty-state copy is shown.
- Mode is **reversible** — switching modes never deletes data. Only vocabulary shifts.
- The migration to introduce modes is **out-of-scope** for the POS pilot. The pilot will use `hybrid` mode. Mode work comes after the pilot proves out the design language.

### 1.4 Plan tiers (already implemented — for context)

| Tier | Price | Limits | Who it's for |
|------|-------|--------|--------------|
| **Free** | Rp 0 | 50 produk, 5 kategori, 2 crew, 100 pelanggan, 500 transaksi/bln, 1 outlet | Single-outlet warung just starting out |
| **Pro** | paid | Unlimited most things, 5 outlets, AI insights + forecasting | Growing chain / serious single-outlet |
| **Enterprise** | paid | Same as Pro + unlimited outlets + larger bulk uploads | Multi-outlet groups, franchises |

> Plan tier is orthogonal to business mode. A Free F&B outlet and a Pro Retail outlet both exist; the tier gates capacity and advanced features, the mode gates vocabulary.

**Design Implications**:
- Never blame the user for hitting a plan limit — show the limit **before** they hit it (e.g., "Produk 48/50 — tambah 2 lagi untuk mencapai batas Free").
- Upgrade prompts are always one click away from a limit, never modal-blocking the user's flow.
- Plan-gated features are visually distinct (ProGate blur + lock icon) but **never** silently disabled — the user must always know the feature exists.

---

## 2. User Role

### 2.1 The two roles Aether actually has

Aether has exactly **two roles** at the data level (`User.role`): `OWNER` and `CREW`. There is no Manager, no Admin, no Super-Admin at the application layer. (The `webmaster` tier is platform-internal, uses `COMMAND_SECRET`, and is NOT a user-facing role.)

| Role | Sees | Cannot see | Auth comes from |
|------|------|-----------|----------------|
| **OWNER** | Every page, every setting, every financial figure | (nothing) | NextAuth credentials, marked `OWNER` at signup |
| **CREW** | Only pages the owner explicitly grants via `CrewPermission.pages` (CSV: `pos,dashboard,...`) | Owner-only pages: Pengaturan, Plan & Pricing, Kelola Crew, Multi-Outlet | Owner creates crew account; permissions editable from "Kelola Crew" |

### 2.2 Role reality check (from HC observations)

- **OWNER** is usually the founder or their family member. They have 30 minutes a day to review the business and 5 hours a week to manage it. They care about: **money in, stock accuracy, crew trustworthiness**.
- **CREW** is usually a kasir (cashier) or gudang (stock clerk). They have one job per shift: **sell** or **restock**. They care about: **speed, no errors, no angry customers**. They do NOT care about dashboards, forecasts, or plan tiers.
- The biggest UX risk is **treating Crew like a mini-Owner** — showing them financial figures they can't act on, or settings they can't change. The redesign must aggressively narrow Crew's view to their job.

### 2.3 Role-based defaults

| Setting | OWNER default | CREW default |
|---------|---------------|--------------|
| Landing page after login | Dashboard | POS (or their first granted page) |
| Sidebar sections visible | All 3 (Utama, Operasional, Manajemen) | Only granted pages, grouped under "Pekerjaan Saya" |
| Financial figures (HPP, profit, margin) | Visible | Hidden — Crew only sees price and stock |
| Audit log access | Yes | No (always) |
| Plan & Pricing | Yes | No (always) |
| Crew management | Yes | No (always) |
| Settings (outlet name, theme, payments, promos) | Yes | No (always) |
| Customer delete | Yes | No (Crew can add, cannot delete) |
| Void transaction | Yes | Configurable per outlet (default: no) |

**Design Implications**:
- Crew's sidebar is **one section** called "Pekerjaan Saya" — not three. No "Manajemen" section. No "Operasional" section. Just the pages they were granted.
- Crew's POS shows **price + stock + product name** only. No HPP, no profit, no margin. (Today this is already true in POS — preserve it.)
- Crew's dashboard (if granted) shows **today's transaction count, today's revenue, low-stock alerts** — NOT profit, NOT forecasts, NOT plan usage.
- The redesign must add a **role-awareness** check to every dashboard card, every sidebar item, every empty state. If the user is Crew and the content is Owner-only, the content is omitted — never blurred, never "locked", just not rendered.

---

## 3. User Intent

### 3.1 The seven intents Aether must serve

Users come to Aether with one of seven intents. Every page should map to exactly one primary intent; pages that try to serve more than one become confusing.

| # | Intent | Example trigger | Primary page | Secondary pages |
|---|--------|----------------|--------------|-----------------|
| 1 | **"Saya mau jual"** | Customer walks in / order placed | POS | (none — POS is the entire flow) |
| 2 | **"Saya mau tahu stok"** | Owner wonders "is item X running low?" | Products (search) | Purchase, Stock Opname |
| 3 | **"Saya mau beli bahan/barang"** | Supplier delivery arrives / owner places order | Purchase (create PO → receive) | Products, Suppliers |
| 4 | **"Saya mau hitung"** | End-of-month physical count | Stock Opname | Inventory Movement |
| 5 | **"Saya mau lihat hasil"** | Morning review, weekly review, monthly review | Dashboard | Transactions, Audit Log, Insights |
| 6 | **"Saya mau atur"** | Change payment methods, add promo, add crew, change theme | Settings | Crew, Plan |
| 7 | **"Saya mau pindah barang"** | Stock transfer between branches (Pro+ only) | Transfer | Multi-Outlet |

### 3.2 Intent → first-screen mapping

When the user logs in, **where they land should depend on their last intent**, not always on a fixed default.

| Role | Default landing | Override rule |
|------|-----------------|---------------|
| OWNER | Dashboard | If last action was POS checkout within 4 hours → land on POS (continue shift) |
| CREW | Their first granted page | If their first granted page is POS → land on POS directly |

> Today both roles land on Dashboard. This is wrong for Crew (they bounce to POS anyway) and forgetful for Owner (they lose context). The redesign should track `lastIntent` in localStorage and use it to route landing.

### 3.3 Anti-intents (what users DON'T want)

These are flows users get dragged into today that the redesign must eliminate:

- **"I just want to add one product"** → today requires navigating to Products → clicking "Tambah" → filling 8 fields. The redesign should allow adding a product **from the POS** when a barcode scan returns no match (intent 1 interrupted by intent 2 sub-task).
- **"I just want to void one transaction"** → today requires Audit Log → find → void. The redesign should allow void from the Transactions page directly (where the user is already looking at the transaction).
- **"I just want to know what sold today"** → today requires Dashboard → wait for chart → switch tab. The redesign should show today's top 5 items as the **first card** on Dashboard, not the third.
- **"I just want to know what's expiring"** → today requires Inventory → Freshness → wait for heatmap. The redesign should show "expiring this week" count as a Dashboard badge.

**Design Implications**:
- The redesign introduces a **"quick action"** concept that lets the user complete a small intent without leaving their current page. (Today's `quick-actions.tsx` already does this for Dashboard → POS; extend the pattern.)
- Every list view should have **inline actions** (void, edit, restock, delete) — no "go to detail page first" detours.
- The number of clicks to complete any of the 7 intents should be: **Jual (3), Lihat Stok (1), Beli (4), Hitung (3), Lihat Hasil (1), Atur (2), Pindah (3)**. Anything beyond that is waste.

---

## 4. First-Time Journey

### 4.1 The cold-start problem

Today, a brand-new Free outlet signs up and sees an empty Dashboard with zero context. They have to figure out: (a) add products, (b) configure payments, (c) try POS, (d) come back to Dashboard. Most don't. The redesign must **shepherd** the first 10 minutes.

### 4.2 The 4-step first-time journey (proposed)

| Step | Screen | What user does | What Aether shows |
|------|--------|---------------|-------------------|
| **1. Perkenalan** | Onboarding modal (post-signup) | Pick business mode, pick plan (Free default), enter outlet name | Founder quote, 3-mode picker (F&B / Retail / Jasa / Hybrid), outlet name input |
| **2. Isi Produk Pertama** | Guided Products empty state | Add 1–3 products (or import Excel if Pro+) | Big friendly "Tambah Produk Pertama" CTA, shortcut to barcode-scan-and-add |
| **3. Coba POS Sekali** | POS with a hint bubble | Add the product to cart, click "Bayar", pick CASH, done | Tooltip: "Klik produk → lihat keranjang → Bayar". Confetti on first successful checkout |
| **4. Lihat Dashboard** | Dashboard with first data | See today's 1 transaction, today's revenue, freshness score | Empty-state replaced with "1 transaksi pertama Anda" card |

After step 4, the user is "warm" — they have data, they understand the loop, and the Dashboard starts being useful.

### 4.3 What the first-time journey must NOT do

- **No 12-field product form on first product.** Reduce to 3 fields: Nama, Harga, Stok. HPP and category can come later.
- **No "configure your payment methods" wall.** CASH is always available by default; QRIS/DEBIT are opt-in from Settings.
- **No "invite your crew" prompt.** Crew is a Pro-tier concern for most users; raise it after their first 50 transactions, not before.
- **No forced tour with next/prev buttons.** Hints should be contextual (one bubble at a time), dismissable, and never block the UI.

**Design Implications**:
- Build an `OnboardingProgress` tracker (localStorage, keyed by outletId): `{ pickedMode, addedFirstProduct, completedFirstSale, viewedDashboard }`. Show progress as a thin top banner that disappears when all 4 are done.
- Empty states must **never be silent**. Every empty state has: (a) what's missing, (b) why it matters, (c) one button to fix it.
- The 4-step journey is **skippable** but **not dismissable forever** — if the user skips, the progress banner stays until they complete step 2 (add product) at minimum.

---

## 5. Daily Operational Journey

### 5.1 The Owner's day

| Time | What they do | In Aether |
|------|--------------|-----------|
| 07:00 (open) | Review last night, check today's plan | Dashboard (today's revenue target, low-stock alerts, expiring items) |
| 09:00 (mid-morning) | Restock if needed | Purchase (create PO, receive goods, inventory updates) |
| 12:00 (lunch rush) | Watch transactions live | Transactions (live filter: today) |
| 17:00 (afternoon) | Spot-check crew | Audit Log (filter: today, by crew) |
| 21:00 (close) | Review the day | Dashboard (today's summary), Insights (top items, forecast) |
| Weekly Sunday | Plan next week | Insights (forecast), Reports (Excel export), Promos (set up weekly promo) |

### 5.2 The Crew's day

| Time | What they do | In Aether |
|------|--------------|-----------|
| 07:00 (shift start) | Log in, land on POS | POS (clean state, today's date, outlet name visible) |
| 07:00–21:00 | Sell, sell, sell | POS (cart, payment, receipt) |
| (rare) | Customer wants to know points | Customers (search by phone) |
| (rare) | Item not in system | Quick-add product inline (if permitted) |
| 21:00 (shift end) | Log out | (no other screen needed) |

### 5.3 The weekly and monthly loops

| Loop | Trigger | Primary action |
|------|---------|---------------|
| **Weekly stock count** | Sunday evening | Stock Opname (count → review → finalize) |
| **Weekly promo review** | Sunday evening | Promos (check active, deactivate stale) |
| **Monthly P&L review** | 1st of month | Dashboard → Laba & Rugi tab, Reports export |
| **Monthly freshness audit** | 1st of month | Inventory → Freshness heatmap, Waste Report |
| **Quarterly plan review** | Every 3 months | Plan & Pricing (upgrade if hitting limits) |

**Design Implications**:
- Dashboard must distinguish **"today" view** (default on entry) from **"review" view** (period selectable). Today = operational, Review = analytical. Don't mix them.
- The weekly/monthly loops should be **prompted, not enforced**. A non-intrusive banner: "Sudah akhir minggu — mau mulai Stock Opname mingguan?" with a "Mulai" button and a "Nanti" dismiss.
- Crew should NEVER see weekly/monthly prompts — those are Owner intents.

---

## 6. Navigation

### 6.1 Today's navigation (sidebar)

The sidebar today groups 13 pages into 3 sections:

| Section | Pages |
|---------|-------|
| **Utama** | Dashboard, Produk, Pelanggan |
| **Operasional** | POS, Transaksi, Pembelian & Inventori, Stock Opname (if inventory), Kirim Stock/Barang (if group) |
| **Manajemen** | Audit Log, Pengaturan, Kelola Crew, Plan & Pricing, Multi Outlet (if group) |

### 6.2 Problems with today's navigation

1. **3 sections is too many for Crew.** Crew sees their granted pages scattered across 3 sections — feels arbitrary.
2. **"Pembelian & Inventori" is two concepts merged.** Pembelian = beli bahan. Inventori = lihat stok + movement + freshness + waste. They should split.
3. **"Manajemen" mixes owner-config (Pengaturan, Crew, Plan) with audit (Audit Log) with admin (Multi-Outlet).** Audit Log is operational review, not management config.
4. **No "Pekerjaan Saya" concept.** The user's most-used pages aren't surfaced above the fold.
5. **Mobile bottom-nav has 5 items** but doesn't adapt to role or mode. Crew sees icons they can't tap.

### 6.3 Proposed navigation restructure (for the redesign)

**OWNER sidebar (4 sections, clearer names):**

```
PEKERJAAN SAYA          (auto-pinned: top 3 most-used by this user)
  - POS
  - Dashboard
  - Produk

JUAL & BELI
  - POS (Kasir)
  - Transaksi
  - Pembelian Bahan/Barang
  - Stock Opname
  - Kirim Stock/Barang     (if group)

STOK & LAPORAN
  - Inventaris (stok, movement, freshness, waste — one page, tabs)
  - Audit Log
  - Insights (AI, forecasting — Pro+)

PENGATURAN
  - Pengaturan Outlet
  - Kelola Crew
  - Pelanggan (loyalty, points)
  - Plan & Pricing
  - Multi-Outlet            (if group)
```

**CREW sidebar (1 section, role-narrowed):**

```
PEKERJAAN Saya
  - POS                     (if granted)
  - Dashboard (lite)        (if granted)
  - Produk (view-only)      (if granted)
  - Pelanggan (search-only) (if granted)
  - Transaksi (own only)    (if granted)
```

> Crew's sidebar is **one section, no headers, no grouping**. They have 1–5 pages, full stop.

**Mobile bottom-nav:**
- OWNER: POS, Dashboard, Produk, Transaksi, More (overflow to full sidebar)
- CREW: POS, (their 2nd granted page), (their 3rd granted page), Profile. No "More" — Crew's nav is always visible.

### 6.4 Navigation rules

1. **Current page is always visible** in the sidebar (highlighted, persistent — never hidden by scroll).
2. **Sidebar collapses on mobile** to bottom-nav + hamburger for the full menu.
3. **No more than 3 clicks** to reach any page from any other page.
4. **Page-switching preserves context** where possible: switching from POS to Products and back should not lose the cart (today it doesn't — preserve this).
5. **Mode-aware ordering**: in F&B mode, "Pembelian Bahan" is above "Stock Opname" (you buy bahan daily, count weekly). In Retail mode, "Stock Opname" is above "Pembelian" (you count weekly, buy irregularly).

**Design Implications**:
- Navigation config moves from a hardcoded array in `sidebar.tsx` to a function `getNavFor({ role, mode, plan, grantedPages, hasGroup })` that returns the tree. This makes the rules testable and the sidebar deterministic.
- The "Pekerjaan Saya" section tracks usage in localStorage (last 7 days, weighted by recency). Top 3 most-visited pages get pinned.
- Crew's "Dashboard (lite)" is a new variant of the Dashboard page that filters out financial figures — same component, different props. (Implementation: pass `role` to `<DashboardPage>` and conditionally render cards.)

---

## 7. Page Guidance

### 7.1 The Page Guidance contract

Every page in Aether must answer 4 questions for the user, **within 2 seconds of load**:

1. **Where am I?** (page title, breadcrumb if nested)
2. **What can I do here?** (primary CTA, secondary actions)
3. **What's the state?** (data summary: count, total, freshness, status)
4. **What should I do next?** (insight, alert, or empty-state CTA)

### 7.2 Page-by-page guidance (the 13 existing pages)

| Page | Where am I? | What can I do? | What's the state? | What's next? |
|------|-------------|----------------|-------------------|--------------|
| **Dashboard** | "Dashboard — [outlet name]" | Quick actions: Buka Kasir, Tambah Produk, Beli Bahan | Today's revenue, txn count, freshness score | Top insight (e.g., "Kopi Susu hampir habis — restock?") |
| **Produk** | "Produk — [count] item" | Tambah Produk, Import Excel (Pro+), Cetak Barcode | Total produk, kategori count, low-stock count | "3 produk stok menipis" alert |
| **Pelanggan** | "Pelanggan — [count] orang" | Tambah Pelanggan, Cari (by WhatsApp) | Total pelanggan, total points outstanding | "5 pelanggan belum transaksi 30 hari" |
| **POS** | "Kasir — [outlet name] — [kasir name]" | Scan/click produk → Bayar | Cart total, item count, sync status | (none — POS is the action) |
| **Transaksi** | "Transaksi — [filter]" | Filter (today/week/month), Export Excel (Pro+), Void | Txn count, total revenue, voided count | "1 transaksi perlu review" (if pending void) |
| **Pembelian** | "Pembelian — [tab: PO/Diterima/Inventori]" | Buat PO, Terima Barang, Lihat Movement | Open POs, received today, total batches | "2 PO menunggu diterima" |
| **Inventori** | "Inventori — [tab: Stok/Movement/Freshness/Waste]" | Adjust stock, Export Waste Report | Total batches, expiring soon count, waste this month | "4 batch expiring <7 hari" |
| **Stock Opname** | "Stock Opname — [status: draft/review/final]" | Mulai Opname, Lanjutkan Draft, Finalize | Draft count, last opname date, variance total | "Opname terakhir 14 hari lalu — mulai baru?" |
| **Kirim Stock** | "Kirim Stock — [if group]" | Buat Transfer, Lacak Pengiriman | In-transit count, completed this month | "1 transfer in-transit" |
| **Audit Log** | "Audit Log — [filter]" | Filter (by user/action/page/date), Export | Total events today, void count, login count | (none — read-only review) |
| **Pengaturan** | "Pengaturan — [tab: Outlet/Pembayaran/Promo/Tampilan]" | Edit, Save | (per-tab summary) | "Promo aktif: 2" |
| **Kelola Crew** | "Kelola Crew — [count] orang" | Tambah Crew, Edit Permission, Hapus | Crew count, plan limit | "1 slot crew tersisa (Free plan)" |
| **Plan & Pricing** | "Plan — [current tier]" | Upgrade, Lihat Usage | Usage bars (produk, kategori, crew, pelanggan, transaksi) | "Produk 48/50 — hampir batas Free" |
| **Multi-Outlet** | "Multi-Outlet — [group name]" | Tambah Outlet, Switch Outlet | Outlet count, plan limit | (none) |
| **Insights** | "Insights — AI & Forecast" (Pro+) | Lihat insight, Lihat forecast | Health score, top 3 insights | "Health score 78 — 2 aksi rekomendasi" |

### 7.3 Empty-state guidance

Every empty state follows this template:

```
[Icon — large, mode-appropriate]

[Title — what's missing, in plain Indonesian]
  e.g., "Belum ada produk"

[Description — why it matters, 1 sentence]
  e.g., "Tambahkan produk pertama untuk mulai menjual di kasir."

[Primary CTA — one button]
  e.g., "Tambah Produk"

[Secondary CTA — text link, optional]
  e.g., "Atau import dari Excel →"
```

**Never**: "No data found." / "Empty." / "Tidak ada data." — these are hostile. Always explain and offer a path forward.

**Design Implications**:
- Create a `<PageHeader>` component that takes `{ title, state, primaryAction, secondaryAction, alert }` and renders the top of every page consistently.
- Create an `<EmptyState>` component that takes `{ icon, title, description, primaryCta, secondaryCta }` and is used everywhere.
- Audit all 13 pages and ensure they all use `<PageHeader>` and `<EmptyState>`. (This is the POS pilot's first deliverable — apply to POS, then propagate.)

---

## 8. System Feedback

### 8.1 The 5 feedback channels

Aether communicates with the user through 5 channels. Each has a specific job:

| Channel | Used for | Example | Implementation |
|---------|---------|---------|----------------|
| **Toast (ephemeral)** | Action confirmed/rejected | "Transaksi tersimpan" | `sonner` (already in use) |
| **Inline status** | Page state changes | "Sync: 3 transaksi pending" | Status pill in `<PageHeader>` |
| **Banner (page-top)** | Important, dismissable | "Sudah akhir minggu — Stock Opname?" | `<Banner>` component (to build) |
| **Modal (blocking)** | Destructive confirmation | "Void transaksi #123? Tidak bisa dibatalkan." | `<AlertDialog>` (shadcn) |
| **Empty state** | No data, here's why + what next | "Belum ada transaksi hari ini" | `<EmptyState>` |

### 8.2 Feedback rules

1. **Toasts auto-dismiss in 3s** for success, **5s** for warning, **never auto-dismiss** for error (user must close).
2. **Never more than 1 toast at a time**. If two actions succeed in quick succession, queue the second toast after the first dismisses.
3. **Banners never block** — they sit below the page header, push content down, and have a dismiss (X) button.
4. **Modals are only for irreversible actions**: void transaction, delete product, delete crew, downgrade plan. Everything else is inline.
5. **Loading states must show progress**, not just spinners. "Memuat 24 produk..." is better than a spinner. "Menyinkronkan 3 transaksi..." is better than "Loading..."
6. **Errors must be actionable.** Not "Something went wrong." But: "Gagal menyimpan produk — kolom 'Harga' kosong. Klik untuk perbaiki." with the field highlighted.
7. **Offline state is first-class.** When POS is offline, the entire header turns amber, a "OFFLINE — transaksi disimpan lokal" banner appears, and the sync icon pulses. Never let the user wonder "did my sale go through?"

### 8.3 The 4 system states every user must always know

| State | Where shown | How |
|-------|-------------|-----|
| **Am I online?** | POS header (top-right icon) | Green dot = online, Amber dot = offline |
| **Is my data synced?** | POS header (sync icon + count) | "Synced ✓" or "3 pending sync" |
| **Am I on the right outlet?** | Sidebar header + POS header | Outlet name visible in both places |
| **Am I close to a plan limit?** | Sidebar footer + Dashboard card | "Produk 48/50" with amber when >80% |

**Design Implications**:
- Build a `<SystemStatus>` component that shows online/offline + sync state + current outlet. Mount in both sidebar header and POS header.
- Plan-usage indicator moves from a hidden Plan page to a persistent sidebar footer chip.
- Every API failure surfaces a toast with the failed action's name + a "Coba lagi" button (where retry is safe).

---

## 9. POS Pilot Redesign — Principles

> This section defines the **scope and rules** for the POS redesign. It is NOT the redesign itself — the redesign happens in a separate doc once this blueprint is approved.

### 9.1 The pilot's one job

Make **"buka POS → pilih produk → bayar → selesai"** feel like one continuous motion, not 4 separate screens.

### 9.2 POS redesign hard constraints (CANNOT touch)

- **Core checkout API** (`/api/pos/checkout/route.ts`) — logic stays
- **Transaction sync** (`/api/transactions/sync/route.ts`) — logic stays
- **FEFO consumption** (inventory batch selection) — logic stays
- **HPP calculation** — logic stays
- **Offline IndexedDB layer** — logic stays
- **Payment method validation** — logic stays
- **Promo application logic** — logic stays

### 9.3 POS redesign CAN touch

- Layout of the POS page (grid splits, panel sizes, responsive breakpoints)
- Visual hierarchy (what's prominent, what's secondary)
- Cart interaction patterns (add, edit qty, remove — gestures and shortcuts)
- Payment flow UX (modal vs inline, success animation, receipt preview)
- Product grid (search prominence, category chips, barcode-scan affordance)
- Customer selection (when to surface, how to add inline)
- Sync status visibility (where, how prominent)
- Empty cart state
- First-time-use hints

### 9.4 POS redesign success criteria

| Criterion | Measurement | Target |
|-----------|-------------|--------|
| **Time to first item in cart** | From POS load → first product click | < 3 seconds |
| **Time to checkout** | From first item in cart → "Bayar" click | < 8 seconds (1-item sale) |
| **Time to complete sale** | From "Bayar" click → success state | < 2 seconds (online), < 1 second (offline) |
| **Error recovery** | From error toast → user knows what to do | 100% (every error is actionable) |
| **Offline transparency** | User always knows sync state | 100% (visual indicator never absent) |
| **Crew comprehension** | New crew can complete a sale with 0 training | 90% (tested with 5-min think-aloud) |

### 9.5 POS redesign anti-goals (what NOT to do)

- ❌ Do NOT add features (no loyalty redemption UI, no split payments, no table management) — pilot is about clarity, not features
- ❌ Do NOT redesign the receipt format — out of scope
- ❌ Do NOT change the data model — `Transaction`, `TransactionItem`, `InventoryBatch`, `LoyaltyLog` schemas stay as-is
- ❌ Do NOT introduce new dependencies — use existing shadcn/ui + Framer Motion + Zustand
- ❌ Do NOT touch mobile native gestures that work today (swipe-to-delete in cart stays)

### 9.6 POS redesign deliverables (separate doc, post-approval)

1. POS Information Architecture diagram
2. POS Component tree (existing → proposed diff)
3. POS Interaction spec (every click, every keyboard shortcut, every state transition)
4. POS Visual spec (colors, spacing, typography — using existing Tailwind tokens)
5. POS Accessibility spec (keyboard nav, screen-reader labels, focus order)
6. POS Pilot implementation plan (file-by-file changes, no core touched)

---

## 10. Glossary — Indonesian terms used in this blueprint

| Term | English | Notes |
|------|---------|-------|
| Kasir | Cashier / POS station | Both the role and the screen |
| Crew | Staff | Aether-specific term, includes kasir + gudang |
| Pelanggan | Customer | |
| Tetap | Regular (customer) | F&B mode term |
| Klien | Client | Service mode term |
| Produk | Product | Generic |
| Menu | Menu item | F&B mode term |
| Layanan | Service | Service mode term |
| Bahan | Ingredient / raw material | F&B mode term |
| Stok | Stock / inventory | |
| Beli | Buy / purchase | |
| Jual | Sell | |
| Hitung | Count | As in stock opname |
| Lihat | Look / review | |
| Restock | Restock | Loanword, common in ID retail |
| Opname | Stock take | From Dutch "opname" |
| Pembelian | Purchase (noun) | |
| Pengaturan | Settings | |
| Pekerjaan Saya | My Work | Proposed nav section name |
| Void | Void | Loanword, common in ID POS |
| HPP | Harga Pokok Penjualan | COGS — cost of goods sold |
| FEFO | First-Expire-First-Out | Inventory consumption rule |
| Freshness | Freshness score | Aether-specific: % of stock not expiring soon |
| Sinkron / Sync | Sync | |

---

## 11. Open questions (deferred to blueprint review)

These are flagged for the founder/product review. They do NOT block the POS pilot.

1. **Should mode-switching be allowed after the outlet has data?** (Recommendation: yes, but warn that labels will change. Data never deletes.)
2. **Should Crew see the customer's loyalty points balance?** (Recommendation: yes, but only at checkout — not in customer search list. Helps Crew upsell.)
3. **Should the "Pekerjaan Saya" auto-pin be based on clicks or time-spent?** (Recommendation: clicks, weighted by recency — simpler, less creepy.)
4. **Should the system suggest a mode based on the first 3 products added?** (Recommendation: no — too presumptuous. Let user pick at onboarding, change in Settings.)
5. **Should Insights (AI) be available to Free users as a teaser?** (Recommendation: yes — show 1 insight read-only, blur the rest with ProGate. Drives upgrades.)

---

## 12. Approval gate

This blueprint is **DRAFT v1.0**. Before POS pilot redesign begins, the following must sign off:

- [ ] Founder (Ahtjong): business mode taxonomy + role defaults + first-time journey
- [ ] Product (Z.ai): navigation restructure + page guidance contract + feedback rules
- [ ] Engineering (Z.ai): confirm POS hard constraints are correct + identify any uncovered dependencies

Once approved, the POS Pilot Redesign doc will be drafted as `docs/POS-REDESIGN-PILOT.md`, scoped strictly to Section 9 of this blueprint.

---

**End of AETHER_UX_BLUEPRINT v1.0 (DRAFT).**

> Next action: Founder/Product review. Do NOT begin POS redesign until this blueprint is approved.
> Platform state at time of writing: see `docs/CHECKPOINT-PHASE-0.5.md` — Phase 0.5 closed, 0 live P0/P1/P2.
