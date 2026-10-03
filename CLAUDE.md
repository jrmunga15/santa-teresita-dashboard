# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Property sales management dashboard for **Parcelación Sta. Teresita 2** (Ahuachapán, El Salvador). Manages 102 land lots, their owners (lote-habientes), installment payments, balances, and receipts.

## Development

No build system. The entire app is a single `index.html` file (~4900 lines). To develop:

- Open `index.html` directly in a browser, or serve it with any static file server:
  ```
  npx serve .
  # or
  python -m http.server
  ```
- Firebase credentials are hardcoded in the `<script type="module">` block at the top of the file (project: `santa-teresita-2`).

## Architecture

**Everything lives in `index.html`** — HTML structure, CSS (lines ~77–562), and JavaScript (lines ~1338–4883). There are no external JS or CSS files.

### Data Model

Three Firestore collections, mirrored as in-memory arrays:

| Variable | Firestore Collection | Description |
|---|---|---|
| `clientes` | `clientes` | Lot owners with contact info and lot assignments |
| `pagos` | `pagos` | Payment records (recibo number, amount, date, balance) |
| `lotes` | `lotes` | Lot status and ownership overrides |

Two additional **static/hardcoded** arrays that never go to Firestore:

- `catalogoLotes` (line ~1390) — all 102 lots from the land plan (id, manzana, area, etc.)
- `datosFinancieros` (line ~1496) — financial data per lot (price, interest rate, months, monthly payment). **This takes precedence over Firestore** for financial fields.

The `lotes` live array is built by merging `catalogoLotes` + `datosFinancieros` + Firestore data (Firestore wins for `estado_plano`; `datosFinancieros` wins for financial fields).

### Firebase Initialization Flow

1. User logs in via Firebase Auth (`signInWithEmailAndPassword`)
2. `onAuthStateChanged` fires → calls `window.initFirestoreListeners()`
3. Firestore `onSnapshot` listeners on all three collections keep data in sync
4. Each snapshot callback re-renders the relevant pages

### Pages / Sections

Navigation via `navigate(page)` (line ~1673). Pages are `<div class="page">` elements toggled with `.active`:

- `dashboard` — financial stats, upcoming payments, lot status grid
- `clientes` — lote-habiente list and edit modal
- `pagos` — payment ledger with month/search filters
- `lotes` — lot overview by status
- `recibos` — print/PDF receipts (uses html2pdf.js)
- `saldos` — running balance per lot with full payment history
- `amortizacion` — 180-installment amortization tables (precalculated per lot)
- `cotizaciones` — quote generator for prospective buyers

### Key Global Functions

- `renderDashboard()` / `renderClientes()` / `renderPagos()` / etc. — called after every Firestore snapshot
- `openModal(id)` / `closeModal(id)` — modal management
- `calcIntereses(saldoAnterior, tasaAnual)` — interest calculation for receipt preview
- `calcSaldoLote(loteId)` — single source of truth for a lote's real capital/interest paid to date, computed from actual payment amounts (not the theoretical amortization table). Used by the Saldos page and both Amortización stat views (screen + print).
- `mesFromFecha(fecha)` — derives the "Mes AAAA" label straight from a "YYYY-MM-DD" string's digits. Never use `new Date(fecha).toLocaleDateString(...)` for this: it depends on the browser's timezone (can roll a day-1 date into the previous month) and locale data (abbreviates "septiembre" as "sept", not "sep").
- `auditarSaldos()` (Saldos page, "🔍 Auditar Saldos" button) — recalculates every cuota/Abono payment's expected saldo from the previous payment's stored saldo and flags any mismatch. Self-service version of the audit used to find and fix every saldo bug described below.
- `seedFirestore()` — seeds Firestore from hardcoded data once, guarded by a `_meta/seeded` marker doc (not a live document count — see "Known data-integrity fixes" below)
- `fixEstadosFirestore()` — reconciles stale `estado_plano` values on load, skips lotes that already have a `propietario_id`

### Normalization Rules (enforced in `onSnapshot` for `pagos`)

- `mes` field: capitalize first letter ("mar 2026" → "Mar 2026")
- `cliente_id`: coerce to integer
- `monto` / `saldo`: coerce to float
- `id`: derived from recibo number if missing

## UI

- Fonts: DM Serif Display (headings), DM Sans (body) via Google Fonts
- CSS custom properties in `:root` for all colors and spacing
- Responsive: sidebar collapses to hamburger + bottom nav bar on mobile (≤768px)
- Language: Spanish (El Salvador locale)

## Known data quirks (not bugs — confirmed intentional)

- **I-1 / I-2 pricing looks swapped.** I-1 (238.44 m², the larger lot) sold for $15,750 while I-2 (201.17 m², smaller) sold for $18,750 — backwards from what area alone would suggest. Confirmed with the business owner: this reflects the actual negotiated prices, not a data-entry error. Don't "fix" this.
- **Multiple receipts share one "Factura No." in their `notas`.** When several lote-habientes pay their cuota on the same collection day, their receipts are issued under a single consolidated tax invoice number (e.g. "Factura No.63" appears on 4 different recibos for 4 different lotes/clientes on the same date). This is the expected accounting process, not a numbering bug — confirmed by the business owner on 2026-10-03.
- **A few sold lotes (F-5, F-6 historically) predate `datosFinancieros`.** A one-time migration that used to force certain lotes back to `estado_plano: "Disponible"` was removed (it was guarded by a per-browser `localStorage` flag, so it kept re-running on any new device and reverting lotes that had since been legitimately sold — see git history around "Remove lot-resetting migration" for the incident). If a lote's status ever looks wrong, use "🔍 Auditar Saldos" and check `estado_plano` vs `propietario_id` consistency before assuming new data is at fault.
