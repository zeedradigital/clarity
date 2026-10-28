# Wealth Score — Design Spec

**Date:** 7 July 2026 · **App:** Clarity (`C:\landing-pages\gap\index.html`) · **Status:** approved by Faiz, pending spec review

## What it is

A **fuel gauge** on the right side of the earning speedometer — the second instrument in the dashboard cluster. Speed = how fast you earn; **fuel = how healthy this month's money behaviour is**. It is an *instrument*, not a score: it shows live state, it can rise and fall within the month, and it judges timing, never desire.

Name shown in UI: **Wealth Score**. Gauge label: `WEALTH`.

## The maturity ladder (fuel level)

Single tank, ~8 vertical segments, red → amber → green bottom-to-top. The month's fuel level is set by which rungs are currently true:

| Level | Zone | Condition |
|---|---|---|
| 0 — bottom | RED | nothing beaten yet — indulgence is winning |
| 1 | LOW AMBER | **Repayment > non-essentials** |
| 2 | MID AMBER | Repayment > essentials too |
| 3 | LOW GREEN | **Investment > non-essentials** |
| 4 — top | TOP GREEN | Investment beats everything (essentials + non-essentials) |

Defense first (repayment), offense second (investment). One-sentence pitch: *"Keep your repayment and investing above your indulgence, and the tank stays green."*

### Definitions (per viewed month, from `S.txs`)

- **Repayment** = own-money repayments only (`type==='repayment' && !rotating`). Card rotation never counts.
- **Investment** = expenses in category `invest`.
- **Essentials** = home food + `family`.
- **Non-essentials** = `personal` + `friends` + outside food.
- **Excluded from the duel entirely:** `business`, `other` (unlabelled money is never judged), income, EMI fixed commitments.

### Home food vs outside food

- Built-in home-keyword seed list (small, closed set): groceries, vegetables, rice, milk, provisions, market, maligai, saman (+ obvious variants).
- Food tags matching the list → **Home** (essential, invisible to the duel). All other Food tags default → **Outside** (non-essential).
- Correction: **Home/Outside toggle** on Food tags in the Settings "Tags & Categories" card. Stored in `S.foodMode = { tagKey: 'home'|'out' }`, overrides the seed list, remembered forever. Same learn-then-correct pattern as tag→category.

## Live behaviour

- The gauge recomputes on every render of the dashboard (every log, edit, delete). Rising and falling mid-month is intentional — instruments report state.
- Month close (first app open in a new month): if the previous month **ended at level 3 or 4 (green)**, award a permanent **green dot**. Dots are stored (`S.greenMonths = ['2026-07', ...]`), never removed, displayed as a small row of dots under the gauge — service stamps. Award happens once per month (idempotent check on render).

## Price tags (separate from the score — information, never points)

Voice = exchange rate in the user's own currency (debt-days), never judgment.

- **Everyday discretionary flag:** shown only in **red/amber months** (hero state bad/wait). Dashboard line near "Where money went": e.g. *"₹3,200 eating out this month — that's 3 days of debt progress."*
- **Big-ticket flag:** any single expense > **10% of average monthly income** (3-month avg) gets a price-tag line in the save flash **whenever debt exists**, even in green months: *"₹65,000 — that's ~2 months of debt progress."*
- Conversion: `debt-days = amount ÷ (avg monthly surplus ÷ 30)` using last-3-months average surplus; if avg surplus ≤ 0, fall back to *"= X% of this month's income."*
- Green month + normal spending → total silence. Scarcity of the message = weight of the message.

## UI changes

1. **Earn card** becomes the instrument cluster: speedometer (existing) left, fuel gauge right (~25% width), shared baseline. Tank fill maps rungs 0–4 to segment fill; segments coloured by zone; smooth fill transition. Green-dot row + `WEALTH` label under the gauge.
2. **Remove the Edit button + hidden edit row** from the earn card (duplicate of Settings → Monthly Income Target). The target *display* stays as the gauge context label.
3. **Remove the streak chip** from the header (`Streak 0d`). "Logged today ✓" text and consistency % chip stay.
4. **Goal tab** gets the Wealth Score breakdown card: current rung + what's needed for the next rung (e.g. *"Invest ₹1,400 more to enter green"*), the two sides' amounts (repayment/invest vs non-essentials), green-dot history, and this month's price-tag summary.
5. Tapping the gauge on Home → Goal tab.

## Data / storage

- `S.foodMode` — `{ tagKey: 'home' | 'out' }` (new)
- `S.greenMonths` — `['YYYY-MM']` (new)
- No changes to transactions, sync, or existing fields. Both new keys merge safely via `defaultState()` spread.

## Edge cases

- No repayments and no investments logged → level 0 (red) with the gentle caption *"Log an investment or repayment to start filling the tank"* — an invitation, not a scold.
- Month with zero non-essentials and any invest/repayment → conditions vacuously true → green; correct and desirable (clean month).
- Editing/deleting past-month entries recomputes that month's *display* if viewed, but **awarded green dots are never revoked**.
- Loan-linked repayments and manual repayments both count; rotation never counts.

## Out of scope (this build)

- Points, tanks-count, streak multipliers (superseded by the single-tank ladder).
- Income side of the duel (already covered by the speedometer).
- Notifications/reminders.
