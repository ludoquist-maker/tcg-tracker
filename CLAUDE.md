# tcg-tracker

This repo is a data source, not an app. It holds `data.csv`, a flat file of
TCG (trading card game) release dates, preorder deadlines, and organized-play
sanctioning deadlines, plus `index.html`, a static viewer that fetches and
renders that CSV client-side.

## Pipeline

`data.csv` is one stage in a larger pipeline:

```
Gmail → Zapier → Claude → data.csv (this repo) → tcg-manager's nightly
Ingestion/tcg-tracker-sync.js → Supabase
```

Rows land here via a Zapier automation (prompt/code changes for that live in
`tcg-manager`'s `Reference/` folder, not this repo). `tcg-manager` then reads
this file every night and syncs it into Supabase. Because of that downstream
consumer, **this file's shape is a contract** — see the compatibility warning
below before changing column count.

`index.html` is a secondary, independent consumer: a browser-side viewer that
`fetch()`es `data.csv` and parses it with its own hand-rolled `parseCSV()`
(reads fixed positions `c[0]..c[4]`, not a header-driven length check — see
below).

## `data.csv` schema (12 columns, as of 2026-07-11)

```
title, game, type, date, note, ordered, created, advertised, release_date_op, platform, event_date_start, event_date_end
```

| Column | Meaning |
|---|---|
| `title` | Free text, not normalized — the event/product name as stated by the source. |
| `game` | A name resolvable by `tcg-manager`'s `GAME_NAME_MAP` alias table (covers common variants for all 20 games). |
| `type` | One of `release`, `preorder`, or `sanction` (see below). |
| `date` | The single most operationally important date for this row's `type` — see per-type meaning below. |
| `note` | Free text / context from the source. |
| `ordered`, `created`, `advertised` | **Legacy** — not read by `tcg-tracker-sync.js`. Left blank for all row types. |
| `release_date_op` | Populated only for `release` rows where the source explicitly stated an OP/prerelease date. Blank otherwise. |
| `platform` | *(added 2026-07-11)* Organized-play platform/portal (e.g. `Eventlink`, `carde.io`, `melee.gg`, `TCG+`, `Ravensburger OP`, or a bare name like `OP Store`). Only applies to `sanction` rows. Blank if unclear — never guessed. |
| `event_date_start` | *(added 2026-07-11)* The event's own start date, if stated. Only applies to `sanction` rows. |
| `event_date_end` | *(added 2026-07-11)* The event's own end date, if the source gives a range. Only applies to `sanction` rows. Blank for single-day events. |

### `type` values and what `date` means for each

- **`release`** — `date` is the product/set release date.
- **`preorder`** — `date` is the preorder/order-lock deadline.
- **`sanction`** — `date` is the sanctioning/event-registration **application
  deadline** (e.g. a publisher's "apply to run this event by DATE"
  requirement) — same operational role `date` plays for `preorder` rows. The
  event's own dates (if any) go in `event_date_start`/`event_date_end`
  instead.

`release`/`preorder` rows always leave `platform`, `event_date_start`,
`event_date_end` blank.

## History

### 2026-07-11 — Sanctioning schema change

Added `platform`, `event_date_start`, `event_date_end` (appended at the end;
nothing existing reordered/renamed) and a new `type=sanction` value, to carry
sanctioning/event-registration deadlines through to `tcg-manager`'s new
"Calendar & Tasks" feature.

This was a **shape-only** change — no `sanction` rows were added yet. Real
`sanction` rows start arriving once the Zapier-side prompt/code changes
(tracked in `tcg-manager`'s `zapier-sanctioning-pipeline-setup.md`) go live.

**Why every existing row had to be touched:** `tcg-tracker-sync.js` parses
this file by reading the header row, then for each data row:

```js
if (values.length >= headers.length) {
  const obj = {}
  headers.forEach((h, idx) => { obj[h] = values[idx] ?? '' })
  rows.push(obj)
}
```

The moment the header grows to 12 columns, any data row still sitting at 9
comma-separated values fails `values.length >= headers.length` and is
**silently dropped** — no error, just excluded from every future sync. So
the header bump and the backfill of three blank trailing fields on all 196
pre-existing rows had to land in the same commit. Verified post-change: every
line (header + all data rows) parses to exactly 12 fields, and simulating the
sync script's length check confirmed zero rows would be dropped.

`index.html`'s own `parseCSV()` reads fixed field positions rather than
checking length against the header, so it was unaffected by this change and
needed no update.
