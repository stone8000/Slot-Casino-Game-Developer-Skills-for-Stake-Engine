# RGS Event Contract

Authoritative source: math-sdk `src/events/event_constants.py` and `src/events/events.py`.
See also `https://stake-engine.com/docs/math/source-files/events`.

## Book Structure

Each line of `books_<mode>.jsonl` is one round:

```json
{
  "id": 1,
  "payoutMultiplier": 0,
  "events": [ ... ],
  "criteria": "0",
  "baseGameWins": 0,
  "freeGameWins": 0
}
```

- `payoutMultiplier` is **x100 scaled** (`1.5x` -> `150`). See `currency-rules.md`.
- It must equal the terminal `finalWin.amount`.

## Event Shape

```json
{
  "index": 0,        // contiguous, starts at 0 (math-sdk uses len(book.events))
  "type": "reveal",  // see canonical list below
  "...": "type-specific fields, flattened at top level (not nested under `fields`)"
}
```

## Canonical Event Types

From `EventConstants`:

| Group | Types |
| :--- | :--- |
| Reveal | `reveal` |
| Win | `winInfo`, `setWin`, `setTotalWin`, `finalWin`, `wincap` |
| Free spins | `freeSpinTrigger`, `freeSpinRetrigger`, `updateFreeSpin`, `freeSpinEnd`, `enterBonus` |
| Tumble | `tumbleBoard`, `setTumbleWin`, `updateTumbleWin` |
| Special symbols | `updateGlobalMult` |

`enterBonus` is defined but emitted by no official sample game — treat it as optional.

### Game-Specific Events

Games may define their own events in `games/<name>/game_events.py` (e.g. `0_0_expwilds`
emits `newExpandingWilds` / `updateExpandingWilds`). Declare them to the validator with
`--allow-event`, otherwise they are reported as `unknown_event_type` warnings.

## Required Ordering

Observed canonical streams:

```
# no win
reveal -> setTotalWin -> finalWin

# free-spin round
reveal -> setTotalWin -> freeSpinTrigger -> updateFreeSpin
       -> reveal -> updateGlobalMult -> setTotalWin -> updateFreeSpin
       -> ... -> winInfo -> setWin -> setTotalWin -> freeSpinEnd -> finalWin
```

## Required Constraints

- `index` must be contiguous and start at 0.
- At least one `reveal` event.
- `finalWin` is the **canonical terminal event** and must be last. `roundResult` is a
  legacy alias; do not emit both.
- Exactly one `finalWin` per round.
- At least one `setTotalWin`, and the last one must equal the terminal amount.
- `setWin` must occur before `finalWin`.
- Top-level `payoutMultiplier` must equal the terminal amount.
- `sum(winInfo totals)` must equal the terminal amount — **except on capped rounds**
  (see below).
- Optional: if a `winInfo` event carries `basePayout`, `appliedMultiplier`, and `payout`
  together, then `payout == (basePayout * appliedMultiplier) / 100` must hold.

## Max-Win (wincap) Rounds

When `win_manager.running_bet_win >= config.wincap`, math-sdk emits a `wincap` event and
truncates the payout (`src/events/events.py::wincap_event`). On these rounds the raw
`winInfo` totals legitimately **exceed** the terminal amount. Rules become:

- `wincap.amount` must equal the terminal amount.
- `sum(winInfo totals)` must be `>=` the terminal amount.

Example: WildParty round 21 sums `winInfo` to `599750` but caps at `500000` (5000x).
Enforcing strict equality here produces a false blocker.

## Validation Command

```bash
node scripts/validate-rgs-events.mjs --input path/to/books.jsonl --format text
node scripts/validate-rgs-events.mjs --input path/to/books.jsonl --format text \
  --allow-event newExpandingWilds,updateExpandingWilds
```

Accepted input formats:
- JSONL with one round object per line (standard `books_<mode>.jsonl`)
- JSON array of events
- JSON object with `events`
- JSON object with `rounds`

## Frontend Rule

Treat the event stream as the single source of truth for displayed outcomes. Never
recompute payouts in the frontend when the events already carry them.
