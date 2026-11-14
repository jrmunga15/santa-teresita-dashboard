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
- Hosted on Netlify (`santateresita.netlify.app`), auto-deployed from `main`. `_headers` forces `Cache-Control: no-cache, no-store, must-revalidate` on every path — without it, mobile browsers in particular can keep serving a stale cached `index.html` for weeks after a fix ships, which looks exactly like a device-specific bug (same account, same data, different behavior on phone vs. computer) but is really just an old bundle. Added 2026-11-14 after this exact symptom on a phone that hadn't refreshed since several fixes earlier.
## Architecture

**Everything lives in `index.html`** — HTML structure, CSS (lines ~77–562), and JavaScript (lines ~1338–4883). There are no external JS or CSS files.

### Data Model

Three Firestore collections, mirrored as in-memory arrays:

| Variable | Firestore Collection | Description |
|---|---|---|
| `clientes` | `clientes` | Lot owners with contact info and lot assignments |
| `pagos` | `pagos` | Payment records (recibo number, amount, date, balance) |
| `pagosPendientes` | `pagos_pendientes` | Draft receipts awaiting admin approval — see "Roles" below. Never merged into `pagos`; approving one creates a brand-new `pagos` doc. |
| `lotes` | `lotes` | Lot status and ownership overrides |
| `costos` | `costos` | Project-wide lotificación expenses (land, permits, infrastructure...). Admin-only, completely separate from `pagos`/`lotes` — see "Costos" below. |

Two additional **static/hardcoded** arrays that never go to Firestore:

- `catalogoLotes` (line ~1390) — all 102 lots from the land plan (id, manzana, area, etc.)
- `datosFinancieros` (line ~1496) — financial data per lot (price, interest rate, months, monthly payment). **This takes precedence over Firestore** for financial fields. **Never add a lote here with placeholder/zero values just because it's currently unsold** (`propietario_id: null` is fine for a lote with a genuine fixed pre-set price — see A-1 — but `valor/reserva/financiar/meses/cuota` all `0` is not): that zero entry permanently overrides whatever gets correctly saved to Firestore later when the lote is actually sold, since the merge logic only restores financial fields from Firestore `if (!datosFinancieros[cl.id])`. This exact bug happened to C-1 and C-2 (removed 2026-11-14) — C-1 had a real buyer (Carlos Martínez) correctly assigned and saved in Firestore, but the app kept showing $0/"Contado" everywhere (Lotes table, lote detail, client profile) no matter how many times the assignment was redone, because the hardcoded zero entry silently won every time. `AMORT_MAP`/`LOTE_PAGO_INFO` have the same "lote id → fixed historical data" shape and the same risk: both also had stale C-1/C-2 entries (an `AMORT_MAP` entry pointing at a generic precomputed table that didn't match the real terms, and a `LOTE_PAGO_INFO` due-date tied to a Feb-2026 reservation that was later cancelled) and were removed in the same fix. If a lote genuinely isn't sold yet, it simply shouldn't appear in any of these three maps at all.

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
- `costos` — ledger of lotificación expenses (admin-only; see "Costos" below)

### Key Global Functions

- `renderDashboard()` / `renderClientes()` / `renderPagos()` / etc. — called after every Firestore snapshot
- `openModal(id)` / `closeModal(id)` — modal management
- `calcIntereses(saldoAnterior, tasaAnual)` — interest calculation for receipt preview
- `calcSaldoLote(loteId)` — single source of truth for a lote's real capital/interest paid to date, computed from actual payment amounts (not the theoretical amortization table). Used by the Saldos page and both Amortización stat views (screen + print).
- `calcularSaldoEsperado(concepto, loteInfo, saldoAntes, monto)` — the formula-driven saldo for a given concepto (cuota/Abono via interest math, `financiar` for Reserva, `0` for Contado/Pago Saldo/Saldo Total, `null` for a free-form "Otro" with no fixed rule). `registrarPago()` and `aprobarPendiente()` both use this to compute the saldo they actually save, instead of trusting whatever ended up in the "Saldo Pendiente" form field — that field is only a live preview (`onLoteChange()`/`onConceptoChange()` call `autoFillMontoSaldo()` to fill it as the user picks lote/concepto), and on at least one mobile browser it was observed staying empty while Monto filled correctly, silently saving a recibo with `saldo: 0`. `guardarEdicionRecibo()` (the manual "✏ Editar" form) deliberately does NOT use this — editing is meant to let the admin freely override any field, including saldo, for corrections.
- `mesFromFecha(fecha)` — derives the "Mes AAAA" label straight from a "YYYY-MM-DD" string's digits. Never use `new Date(fecha).toLocaleDateString(...)` for this: it depends on the browser's timezone (can roll a day-1 date into the previous month) and locale data (abbreviates "septiembre" as "sept", not "sep").
- `auditarSaldos()` (Saldos page, "🔍 Auditar Saldos" button) — recalculates every cuota/Abono payment's expected saldo from the previous payment's stored saldo and flags any mismatch. Self-service version of the audit used to find and fix every saldo bug described below.
- `auditarIntegridad()` (Dashboard, "🔍 Auditar Integridad" button) — structural check: duplicate `lotes`/`clientes` docs, orphaned `pagos`/`lotes` references, `estado_plano` vs `propietario_id` mismatches, a recibo number shared by different clients. Queries Firestore directly with `getDocs()` instead of using the in-memory `lotes` array, because that array's build (the `lotes` `onSnapshot` merge) already dedupes by id and would hide the exact class of bug this is meant to catch (see the next bullet).
- `seedFirestore()` — seeds Firestore from hardcoded data once, guarded by a `_meta/seeded` marker doc (not a live document count — see "Known data-integrity fixes" below)
- `fixEstadosFirestore()` — reconciles stale `estado_plano` values on load, skips lotes that already have a `propietario_id`

### Roles & Approval Queue

- `EMAILS_LIMITADOS` (top of the `<script type="module">` block) lists emails with restricted access. A limited account can only: generate a receipt (the "+ Registrar Pago" button, always in the topbar regardless of page), track it in **Pendientes**, and print an already-approved one in **Recibos**. Dashboard, Lote-habientes, Pagos Recibidos, Lotes, Saldos por Lote, Amortización, Cotizaciones and Costos are all hidden from its nav (`PAGINAS_PROHIBIDAS_LIMITADO` in `navigate()`, plus the matching `.nav-*` CSS classes under `body.usuario-limitado`).
- `registrarPago()` sends a limited account's receipts to `pagos_pendientes` (status `pendiente`) instead of writing `pagos` directly.
- A limited account also can't see the "🗑 Borrar" or "✏ Editar" buttons on an existing recibo (`.btn-borrar-recibo`/`.btn-editar-recibo`, hidden by the same CSS) — not that they'd work anyway, since `firestore.rules` already denies it any write (create/update/delete) on `pagos`. The admin account is never given the `usuario-limitado` body class, so it keeps both buttons and full write access, before and after a recibo is approved.
- If the admin later deletes a `pagos` doc that came from an approved `pagos_pendientes` draft, `borrarRecibo()` finds that draft (matched by `recibo_final`) and stamps `recibo_borrado_en`/`recibo_borrado_por` on it instead of leaving it silently stuck showing "✓ Aprobado" for a recibo that no longer exists — `renderPendientes()` then shows a "🗑 Recibo No. X borrado el DD/MM/YYYY" badge instead.
- Everyone can see the **Pendientes** page; only non-limited (admin) accounts get the "Aprobar"/"Rechazar" buttons there (`renderPendientes()`).
- `aprobarPendiente(docId)` does NOT trust the `saldo_sugerido` the limited user saw at draft time — it recomputes the saldo from the *current* `pagos` state using the shared `esConceptoInteresable()`/`calcularNuevoSaldoPorMonto()` helpers, in case another payment for the same lote was approved in the meantime. Only then does it create the real `pagos` doc and assign the recibo number.
- This in-app gating is UI-only. The matching enforcement lives in `firestore.rules` (see below) — **the email list in both files must be kept in sync by hand**, there's no shared source of truth between them.

### Costos

- Simple admin-only expense ledger (`costos` collection), unrelated to any lote or pago — tracks project-wide spend since the lotificación started. Categories (Honorarios Profesionales, Aranceles, Mano de Obra, Alquileres, Prestaciones de Ley, Tasas Alcaldia, Equipo Oficina, Materiales de Construcción, Comisiones por Venta, Gastos Varios) mirror the business's own historical "Segmento" ledger (imported 2026-11-08 from `GASTOS_INICIALES_PROYECTO_PARCELACION_2024_A_2026`, 76 entries, 2024-07-16 to 2026-03-31) — don't rename them without checking that file, existing `costos` docs keep whatever string was in their `categoria` field.
- `abrirModalCosto(docId?)` / `guardarCosto()` / `borrarCosto(docId)` handle the single add/edit modal (`modal-costo`) and deletion — no approval flow, since only the admin can ever reach this page.
- The Dashboard's "Rentabilidad" row and the Costos page itself both show **Utilidad Neta** = `pagos` total − `costos` total, turning red when negative (`renderDashboard()`, `renderCostos()`).
- Unlike `pagos`/`lotes`/`clientes`, nothing else in the app reads `costos`, so `firestore.rules` denies a limited email even `read` access to it, not just `write`.

### Firestore Security Rules

`firestore.rules` (deployed separately via the Firebase console or `firebase deploy --only firestore:rules` — this repo has no CI wired to auto-deploy it) mirrors `EMAILS_LIMITADOS`: a limited email can read everything the app needs but can't write to `clientes`/`lotes`/`pagos`/`_meta`, and in `pagos_pendientes` can only `create` a doc attributed to itself with `estado: 'pendiente'` — never approve/reject (`update`/`delete`) one. Adding or removing a restricted user means editing the email in *both* `index.html`'s `EMAILS_LIMITADOS` and `firestore.rules`' `esLimitado()`.

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
- **A lote can end up with two `lotes` docs sharing the same `id` field.** `seedFirestore()` seeds any lote that already had financial data with an auto-generated Firestore doc ID (`doc(collection(db,'lotes'))`), but every write path that assigns/reconfirms an owner (`confirmarAsignarPropietario()`, `guardarCliente()`, the auto estado-update in `registrarPago()`) writes to `doc(collection(db,'lotes'), loteId)` — i.e. the lote's own string id AS the doc ID. For a lote seeded with a random doc ID, that first reassignment creates a second, separate doc instead of updating the first one, and which of the two the app displays becomes non-deterministic (picked up a real incident: lote C-1, 2026-11-14 — looked fine on one device/session and showed "Contado" with stale pre-sale numbers on another, purely by chance of snapshot ordering). The `lotes` onSnapshot handler's merge now resolves this deterministically — it prefers the doc whose Firestore doc ID equals the lote's id, falling back to a field match only if that doesn't exist — but an existing duplicate pair isn't cleaned up by that fix alone; the stale doc (random doc ID, usually `estado_plano: "Disponible"`/no `propietario_id`) should be deleted once found.
