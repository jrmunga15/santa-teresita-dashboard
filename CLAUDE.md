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
- `seedFirestore()` — seeds Firestore from hardcoded data if < 5 client records exist (runs once on first load)
- `fixEstadosFirestore()` — reconciles stale `estado_plano` values on load

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
