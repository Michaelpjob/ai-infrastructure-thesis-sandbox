# Update Proposal — AI Infrastructure Portfolio Tracker

**File targeted:** `index.html`
**Goal:** reposition the tool from a real-portfolio mark-to-market dashboard into a **thesis-tracking science experiment** — users set hypothetical weights, the tool locks in a baseline, and on every return visit they see how their allocation would have performed since.

Not a brokerage. Not a portfolio. A sandbox where users mess with percentages and come back to see whether their thesis was right.

---

## 1. The repositioning

The current build is framed in dollar terms — "Project Value," "Live Invested," "Day Move." That language reads as if the user has a real book. Strip it. The unit is **percent of a hypothetical 100% allocation**, and the headline metric is **cumulative return since the user locked the scenario**.

Replace the dollar framing throughout. No "Project Value." No "money in." Just weights, prices, and the resulting hypothetical return curve.

The new tagline (header subline candidate): *"A thesis sandbox. Set your weights, lock in a baseline, come back to see how your bet did."*

---

## 2. Scenario lifecycle

Every scenario lives in one of three states:

- **`draft`** — user is still adjusting weights. No performance tracking. Auto-saves to localStorage on every change.
- **`tracking`** — user has clicked **Lock scenario**. The tool captured the current price of every position with a non-zero weight and stored it as the inception baseline alongside an inception timestamp. From this moment, all return math is anchored to that baseline.
- **`closed`** — user explicitly closes the scenario. The final return is frozen. No further price updates apply.

### The lock action

When the user clicks **Lock scenario**, show a modal:

> "Capture prices as of [now] and start tracking this allocation? You can fork it later to test variants."

On confirm: snapshot every active position's current price into `inceptionPrice`, set `lockedAt` to now, set `status` to `tracking`. The scenario name becomes immutable from this point (or editable but versioned — your call).

### Editing a tracking scenario

When a user changes a weight on a tracking scenario, **do not silently mutate it**. Instead, prompt:

> "This scenario is tracking from [date]. Edit it in place (history preserved, new weights applied going forward), or fork into a new draft scenario?"

Default to **fork**. That way the original baseline + return curve survive intact and the new draft starts fresh.

---

## 3. localStorage persistence

No backend. Everything lives in `localStorage` under a single key (e.g. `ai-infra-tracker-v1`). Schema:

```json
{
  "schemaVersion": 1,
  "activeScenarioId": "scn_abc",
  "scenarios": [
    {
      "id": "scn_abc",
      "name": "Base Case",
      "createdAt": "2026-05-07T03:48:00Z",
      "lockedAt": "2026-05-07T03:50:00Z",
      "closedAt": null,
      "status": "tracking",
      "positions": [
        {
          "symbol": "TSM",
          "name": "Taiwan Semiconductor",
          "theme": "Compute",
          "tier": "Core",
          "category": "Foundry",
          "weight": 6.0,
          "inceptionPrice": 178.32,
          "inceptionDate": "2026-05-07T03:50:00Z"
        }
      ],
      "themeTargets": { "Compute": 30, "Power": 45, "Edge": 25 },
      "riskLimits": { "maxSingle": 8, "maxHighRiskBucket": 25, "maxTheme": 50, "maxHighRiskSingle": 3, "maxWatchBucket": 5, "minCoreBucket": 30 },
      "snapshots": [
        { "date": "2026-05-07", "prices": { "TSM": 178.32, "GEV": 991.30 } }
      ],
      "notes": ""
    }
  ],
  "settings": {
    "provider": "stooq",
    "lastQuoteRefresh": "2026-05-07T03:50:00Z",
    "quoteCache": { "TSM": { "price": 178.32, "asOf": "2026-05-07T03:50:00Z" } }
  }
}
```

### Migration

- On first load: if no localStorage entry exists, seed with the existing Base Case as a `draft`. Do **not** auto-lock — the user must explicitly lock to start tracking.
- On schema mismatch: read `schemaVersion` and migrate. If migration fails, offer the user an export-then-reset path.

### Reset

Add a **"Reset to factory defaults"** button (Settings or Scenarios tab) with a hard confirmation modal. Wipes localStorage. Mention that exported JSON can be re-imported.

---

## 4. Performance engine

For each `tracking` scenario, on every render:

```
positionReturn[i]    = (currentPrice[i] - inceptionPrice[i]) / inceptionPrice[i]
positionContribution = weight[i] × positionReturn[i]
scenarioReturn       = Σ(positionContribution)   // for active positions only
```

### Headline metrics (replaces current dollar metric strip)

- **Cumulative return since inception** (signed %)
- **Annualized return** (only show if `daysSinceInception > 30`)
- **vs. equal-weight benchmark** of the same active universe
- **vs. SPY** as a passive market benchmark (requires provider support — fetch SPY at inception and now)
- **Max drawdown** from peak return (requires the snapshot series — §5)
- **Coverage %**: positions with valid `inceptionPrice` and `currentPrice` / total active positions

### Per-position view

Inside the Positions tab, surface for each name:
- Inception price · Current price · Position return % · Contribution to scenario return (weight × return)
- Sort by contribution descending — gives the user a clean "top contributors / top detractors" read.

---

## 5. Time-series snapshots

To draw the cumulative-return chart, you need more than two data points. Two options:

### Option A — Snapshot-on-visit (recommended for MVP)

Every time the user opens the app and prices refresh, append a snapshot to `scenario.snapshots`:

```json
{ "date": "2026-05-08", "prices": { "TSM": 181.40, "GEV": 1003.22, ... } }
```

Cumulative return curve = computed from the snapshots array, weighted by position weights at each snapshot.

Pros: zero extra API surface, works with any provider that returns spot prices.
Cons: sparse — if the user comes back weekly, the curve is weekly resolution.

### Option B — Backfill via provider history

If the provider exposes historical daily closes, fetch inception → today for each symbol on lock, populate `snapshots` densely. Dense curve, no return-visit dependency.

Pros: clean daily curve immediately on lock.
Cons: more API surface; failure modes when the provider lacks history for a ticker.

**Recommend:** ship Option A by default; expose a **"Backfill from provider history"** button that runs Option B for users whose provider supports it. Cap the snapshot array at, say, 365 entries (rolling window) to keep localStorage size sane.

---

## 6. Scenarios tab — promote to first-class

The Scenarios concept already exists in skeleton form. Make it the home of the tool:

### List view

For every scenario:
- Name · Status badge (`draft` / `tracking` / `closed`)
- Inception date (or "—" if draft)
- Days tracking
- Cumulative return (signed %)
- Drawdown from peak
- Active / Locked / Edit / Fork / Close / Delete actions

### Leaderboard view

Sort by cumulative return descending. Highlights the winning thesis among the user's scenarios.

### Compare view

Multi-select up to four scenarios → renders an overlaid cumulative-return chart from each scenario's inception. Useful for "Base Case vs Power-bull-tilt vs Counter-narrative" comparisons.

### Fork action

Clones a scenario as a new `draft` with the same positions and weights but no inception baseline. User adjusts weights and locks to start a new tracking run.

---

## 7. What to remove or rename

| Current element | Action |
|---|---|
| "Project Value" headline metric | Remove. Replace with **Cumulative return since inception**. |
| "Live Invested %" | Remove. Redundant with weight totals. |
| "Day Move" as headline | Demote to a secondary row inside Dashboard, alongside other diagnostics. |
| Dollar amounts everywhere | Strip. The tool is hypothetical. |
| `Pricing` tab | Either merge into a `Settings` tab or rename to **Quotes & data** with provider, refresh, cache info. Don't surface it as primary navigation. |
| Footer copy | Replace with: *"Tracking is hypothetical. Prices are end-of-day or delayed. This is a science experiment, not a brokerage."* |

---

## 8. What to add

- **Onboarding modal** on first load: 3 cards explaining (1) adjust weights → (2) lock scenario → (3) come back to see performance. Dismissible, never shown again.
- **Inception-baseline strip** in the header of every tracking scenario: "Tracking since [date] · [N] days · [+/-X.XX%] cumulative."
- **Cumulative-return chart** on Dashboard, line graph from inception to now. Use the snapshots array. Overlay equal-weight and SPY benchmarks if available.
- **Top contributors / top detractors** card on Dashboard — three names each, with contribution-to-return.
- **Fork button** on every tracking scenario in the Scenarios list and on the Dashboard header.
- **Export scenario as JSON** for backup or sharing. Import JSON to restore.

---

## 9. Edge cases

| Case | Behaviour |
|---|---|
| User adds a position to a `tracking` scenario | Capture **that position's** inception price at the moment of add. Per-position inception, not scenario-level. Tooltip explains. |
| User removes a position from a `tracking` scenario | Freeze its contribution-to-date; stop accruing. Show as struck-through in the position table with a "removed [date]" note. |
| Provider returns no price for a symbol | Flag as "unpriced" (already exists). Exclude from return math. Surface coverage % in the header. |
| User clears localStorage / new browser | Tool resets to seed Base Case in `draft`. No history. Offer JSON import as recovery. |
| Multi-tab / multi-session | Last write wins. Document. |
| User edits weight on a `tracking` scenario | Prompt: edit-in-place vs fork. Default fork. |
| Scenario locked with one position having no price | Block lock with a warning: "X positions have no price. Resolve or remove before locking." |
| User closes a scenario then re-opens (un-closes) | Allow toggling `closed` ↔ `tracking`. Closing freezes the snapshot stream; reopening resumes it. |

---

## 10. Out of scope for this update

- Accounts or cross-device sync (localStorage only)
- Real-money or brokerage integration
- Options, shorts, or non-equity instruments
- Tax / fee modeling
- Sharing scenarios via URL (could be a follow-up: encode JSON in fragment)
- Inference of past behaviour for scenarios that pre-date the lock action — only forward-tracking is valid

---

## 11. Success criteria

The update succeeds if a user can:

1. Open the tool, see the seed allocation, and lock a thesis in under 60 seconds.
2. Return in two weeks and see how their thesis performed vs. equal-weight and SPY.
3. Fork the scenario, adjust weights, and run a parallel experiment without losing the original baseline.
4. Compare two or more scenarios on the same chart.
5. Close the browser, reopen the next day, and find every scenario exactly where they left it.
6. Reset everything when they want a clean sheet, with a single confirmed click.

---

## 12. Implementation notes for Codex

- The schema in §3 is the source of truth — implement it exactly so future migrations are clean.
- Bump `schemaVersion` on any breaking change.
- The price-refresh path already exists; reuse it for the snapshot capture in §5 Option A. Don't introduce a parallel fetch path.
- The Lock / Fork / Close transitions should each be a single function that produces a new scenario object — keep them pure so they're easy to test.
- Charts: a small inline SVG line chart is sufficient. No new chart library required.
- Time math: store all timestamps as ISO 8601 UTC. Display in the user's locale.
