# Menu Ascension + Visual Tray — implementation brief

Status: **IMPLEMENTED AND VERIFIED** in Studio (place `74686069419969`). `GAME_CONTEXT.md`
carries the shipped architecture; this file is kept as the record of the design
decisions and, at the bottom, of where measurement overruled the design.

Decisions taken by the user (2026-08-23):

- roster ~30 foods, `CAL_RATIO` retuned 1.60 → 1.57
- Menu Ascension **resets on New Diet**; highest-ever is kept for the Collection
- Kitchen becomes an **infinite level ladder** (no top level)
- all four phases in scope: tray visual, ascension core, Collection, achievements

---

## 0. The invariant this change had to solve first

`Balance.validate()` asserts payback drift `(MergeCost / CalRatio)^(MaxTier-1) >= 0.25`.
A merge is 3 → 2, so income only improves when `CalRatio > 1.5`, which means drift
is strictly below 1 — and an **unbounded** ladder therefore drives it to zero. The
endgame collapses into seconds. Infinite merging is only safe if every Menu
Ascension scales **costs** by the same factor it scales **income**.

The rule the whole design rests on:

```text
within one menu cycle   income outruns cost slightly   (the climb)
across an ascension     cost scales exactly with income (the reset)
```

Drift becomes **periodic**, not monotonic. The validator check changes from
"across the whole ladder" to "across one menu cycle", plus a new check that
ascension is cost-neutral.

---

## 1. Numbers

```text
FOODS_PER_MENU   = 30
CAL_RATIO        = 1.57     (was 1.60; capped by 1.5 * 4^(1/29) = 1.5733)
KG_RATIO         = 1.40     (unchanged, must stay < MergeCost 1.5)
MERGE_CAL_RATIO  = 1.80     (unchanged, must stay > CAL_RATIO)
MERGE_CAL_BASE   = 15
KG_SCALE         = 3.6      (PacingSim re-measures; expect it to move)
```

Derived:

```text
drift per cycle       (1.5/1.57)^29   = 0.2665   >= 0.25   OK
income per ascension  1.57^30         = 7.5e5
Small Fries ★1 vs Final Food ★0       = 1.57x   (exactly one tier step)
```

That last line is the user's stated design principle, and it falls out of the
formula for free — no special case needed.

## 2. Internal representation

**One unbounded integer, the GLOBAL TIER `g >= 1`.** Everything else is derived.

```lua
menuIndex(g) = ((g - 1) % FOODS_PER_MENU) + 1
ascension(g) = (g - 1) // FOODS_PER_MENU
calPerSec(g) = CAL_RATIO ^ (g - 1)
eatKg(g)     = KG_SCALE * KG_RATIO ^ (g - 1)
mergeCost(g) = floor(MERGE_CAL_BASE * MERGE_CAL_RATIO ^ (g - 1))
mergeResult(g) = g + 1        -- ALWAYS. There is no MAX any more.
```

`Food.Tiers` stops being a 20-entry array and becomes a 30-entry **base registry**
keyed by menu index (`id`, `name`, `icon`, `colour`). `Food.get(g)` computes.

This keeps the tray array, every remote payload, and the save format as plain
integers — no shape migration on `tray`.

`Food.MaxTier` is deleted. `Food.AbsoluteMaxTier = 3000` (≈ ★100) is added purely
as an `ActionSpec` sanity bound so a malformed client cannot send `1e18`; the
economy still re-checks against the Kitchen ceiling.

## 3. Roster

30 real fast-food items, redesigned as a coherent menu (fries → burgers →
chicken → breakfast → desserts → combo meals → the absurd finale). The current
tiers 13–20 joke names ("The Heat Death Of Hunger") are retired: they exist only
because the ladder had to stop somewhere, and it no longer does.

Every food needs a distinct icon. There are **no image assets in the project
today**, so phase A ships emoji glyphs at large size (the tray card is ~78% icon)
and the registry carries an `image` field that is `nil` for now — swapping to real
assets later is a data change, not a code change.

## 4. Kitchen — infinite ladder

```lua
Kitchen.maxTier(level) = 2 * level + 2       -- unbounded, same 2-tiers-per-level
Kitchen.name(level)                          -- 9 base names, then "<name> ★N"
Kitchen.upgradeCost(level) = BASE * GROWTH ^ (level - 1)   -- never nil
```

`Kitchen.MaxLevel` is removed; `clampLevel` keeps only the lower bound. The
`validate()` check "the ladder ends exactly at `Food.MaxTier`" is removed — there
is no end. It is replaced by:

- every level must open at least one tier;
- cost growth **per tier** must exceed `CAL_RATIO`, or climbing eventually becomes free.

**Re-anchor required.** The shipped `COST_GROWTH = 6.0` per 2-tier level is 2.449x
per tier against income growth of 1.57x per tier. Over a finite 20-tier ladder that
is a deliberate wall; over an infinite one it diverges so hard that Menu Ascension
★1 (Kitchen 15) is unreachable in any realistic number of diets.

New shape: `GROWTH = CAL_RATIO^2 * WALL_MARGIN`. `WALL_MARGIN` is the single dial
and it is what decides run length. Start at 1.15 and let PacingSim set it against
these targets:

- diet 0 finishes around Kitchen 5 (tier 12), as it does today;
- Menu Ascension ★1 (Kitchen 15, tier 30) lands around diet 6–10;
- each diet still ends on a Kitchen the player can see but would rather not buy.

Do **not** ship a value that was not measured.

## 5. Bags — menu-relative, 9 rungs

Bags keep 9 rungs and keep `VALUE_FALLOFF = 1.55` (1.55^8 = 33.3, inside the 45
cap). Drop bands are re-expressed as **menu indices 1..30** (~3.3 indices per rung)
instead of absolute tiers 1..20.

At roll time the band is lifted into the player's current cycle:

```text
globalTier = state.ascension * FOODS_PER_MENU + menuIndex
```

and the price is scaled by `CAL_RATIO ^ (FOODS_PER_MENU * state.ascension)`, which
is exactly the income gained per ascension. That is the cost-neutrality half of
§0, and it is what stops bags becoming free after the first cycle.

Existing behaviour that stays: a roll above the Kitchen ceiling lands **on** the
ceiling, and a jackpot that was clamped away is not announced as a jackpot.

The `unlockKg` gates stay as they are for the first cycle. The "top three bags are
the rebirth reward" framing survives unchanged.

## 6. State and persistence

New / changed fields:

| Field | Scope | Meaning |
|---|---|---|
| `ascension` | run | `(peakTier - 1) // 30`, resets to 0 on New Diet |
| `peakTier` | run | highest global tier **produced** this run |
| `peakTierEver` | permanent | highest global tier ever produced |
| `discoveredFoods` | permanent | rekeyed `tier_N` → `food_<menuIndex>`, 30 max |
| `foodStats` | permanent | `[menuIndex] = { eaten, kg, topAsc }` |
| `achievements` | permanent | `[id] = true` |

**Migration.** Old saves key discovery as `tier_1..tier_20` against the *old*
20-food roster. The new roster is a different 30-food list, so index N no longer
names the same food. Remap `tier_N → food_N` and accept that the name behind a
discovered cell shifts. The alternative — dropping discovery entirely — is worse,
and the place has not shipped, so the blast radius is Studio test profiles.

Load-time clamping follows the existing rules in `GAME_CONTEXT.md`: bad tiers are
discarded, `ascension` is **derived rather than trusted**, and the tray is still
NOT clamped to the Kitchen ceiling.

## 7. Tray UI — the visual card

```text
┌────────────┐
│         ★2 │
│            │
│   [FOOD]   │
│            │
└────────────┘
```

**Removed** from the slot: food name, `T{n}`, Cal/s — and with them the entire
`fitTextSize` / `NAME_MIN_SIZE` / `fitCache` name-fitting machinery and the
`Badge` and `Rate` labels.

**Added:**

- icon at ~78% of the cell, centred, the only thing in the card;
- `★N` badge, top-right, drawn **only** when ascension >= 1, gold pill, ~16% of cell;
- desktop: hover shows a small tooltip above the card — `Large Fries ★2`;
- touch: a press that never crosses the drag threshold shows the same tooltip.
  This is safe — the current code states a bare click on a tile does nothing, so
  there is no verb to collide with.

The tooltip never occupies tray space; it is drawn in the existing drag layer.

Card colour stays keyed to **menu index** (so matching pairs are still findable at
a glance). Ascension is carried by the badge plus a border treatment, never by hue
— otherwise every card in a late cycle would be the same colour.

## 8. Ascension visuals — thresholds, not a growing glow

```text
★0     plain card
★1     thin gold border
★5     gold border + corner notch
★10    prestige border
★25    higher-grade plate
★50    rare treatment
★100+  reused high-ascension treatment, and it STOPS here
```

Past ★100 the **number** carries the progression. A ★5,000 food must not be
brighter than a ★100 one.

## 9. Collection

One entry per **base food**, 30 total, forever. `Large Fries` and `Large Fries ★7`
are the same entry.

New `Client/UI/CollectionPanel` as a Menu tab:

```text
LARGE FRIES
Menu Ascension     ★2
Weight Gain        4.82M
Times Eaten        1,284
Highest Ascension  ★7
[large preview]
```

The physical `MenuBoard` SurfaceGui stays as the ambient version but is reduced to
icon + discovered state; it is no longer the only place detail lives.

Division of responsibility, and it is strict:

- the **HUD** answers "what is on my tray right now";
- the **Collection** answers "what is this food and what does it do".

## 10. Achievements

Net-new; nothing exists in the game today. Data-driven only — one config table,
one server checker, **no script per achievement**.

Collection achievements count **base foods**: eating `Large Fries`,
`Large Fries ★1` and `Large Fries ★2` is ONE unique food. Ascension gets its own
ladder (★1, ★5, ★10, ★25, ★100, continuing from data).

## 11. Explicitly not built

The post-rebirth endgame layer. The architecture must not assume rebirth is the
final progression system — `foodStats`, `peakTierEver` and the achievement config
are all permanent-scoped so a later layer can read them — but no endgame currency
or mechanic is added now.

## 12. Verification gate

Per `AGENTS.md`, none of this is done until:

- `Balance.validate()` passes on server boot with the rewritten invariants;
- `ServerStorage.EconomyTests` passes (merge, eat, bag, upgrade, offline, rebirth);
- `ServerStorage.PacingSim` has been **re-run** and `WALL_MARGIN` / `KG_SCALE` set from it;
- Play mode confirms the tray, the tooltip, an ascension merge, and the Collection;
- `GAME_CONTEXT.md` is updated in the same pass.


---

# WHAT CHANGED BETWEEN THIS DESIGN AND WHAT SHIPPED

Four things in the plan above were wrong. Each was caught by running something
rather than by reasoning harder, and each is worth keeping because the wrong
version is the intuitive one.

## 1. `WALL_MARGIN` 1.15 -> 1.45, and it is not the dial the plan said it was

The plan set `WALL_MARGIN` as "the dial that decides when Menu Ascension is
reached" and guessed 1.15. PacingSim disagreed on both counts:

- ascension timing barely moved with it (diet 10 at 1.15, still diet 10 at 1.06);
- what it actually controls is **whether each New Diet is longer than the last**.

`Balance.validate()` proves diets should grow ~1.56x (target 3.5x vs metabolism^2
= 2.25x). That proof quietly assumed climbing the Kitchen was a one-off gain --
true while the ladder ended at nine levels. On an infinite ladder every diet buys
more levels forever, each worth `KgRatio^2` = 1.96x weight per meal, so the real
speedup is ~4.4x against a 3.5x target and **runs shrink**. Measured at 1.15 they
went 20:13 down to 9:09 across one cycle.

| `WALL_MARGIN` | runs across one cycle |
|---|---|
| 1.20 | 23:54 -> 10:35 (halving) |
| 1.30 | 25:20 -> 17:45 (shrinking) |
| **1.45** | **22:47 -> 46:01 (growing)** |
| 1.55 | balloons to 81:23 |

## 2. `KG_SCALE` 3.6 -> 4.8

A longer menu costs more merges to climb, which costs more eats to fund, so diet 0
had drifted to 718 eats against the ~620 an earlier pass established as the point
where the clicking stops reading as grind. 4.8 buys it back (17:17, 624 eats)
without touching a single ratio.

## 3. `AbsoluteMaxTier` 3000 -> 1050, and the [*]100 tier does not exist

The plan called 3000 "a sanity bound so a forged 1e18 cannot reach the
exponentiations". The bound's real job turned out to be different: **1.80^g, the
merge price, overflows a double at about tier 1,207**, so 3000 sat well past the
point where the arithmetic dies. A seeded test at [*]120 produced `inf` Cal/s and
`inf` merge price with nothing reporting a problem.

Consequences, all of them forced rather than chosen:

- the ceiling is now [*]35, and the last ascension a player can COMPLETE is [*]34;
- the visual threshold ladder ends at [*]30 instead of [*]100;
- the `asc50` / `asc100` achievements became `asc30` / `asc34`.

The design asked for a [*]100 capstone. It cannot exist in double-precision
arithmetic at these ratios, and no retuning moves that -- passing it requires a
mantissa/exponent number type through the entire economy, which is a separate
piece of work. Asserts in `Config.Food`, `Client/UI/Theme` and
`Config.Achievements` now fail at require time if any of the three ladders drifts
past what the game can reach.

## 4. The Collection is rows, not a tile grid with a detail pane

Thirty foods with four statistics each fit in a scrolling list without a second
navigation step. A master/detail split on a phone hides twenty-nine rows to show
one. Every number the sketch asked for is present, one tap shallower.

# MEASURED PACING, SHIPPED CONFIG

Engaged play, seed 12345, `Sim.campaign`:

```text
d0  17:17  624 eats  K6   -- 22:57 casual
d11 27:45           K16  [*]1
d29 42:07           K30  [*]2
```

Ascensions land roughly eighteen diets apart and every diet is longer than the one
before it.

# VERIFICATION PERFORMED

- `ServerStorage.EconomyTests`: **154/154 pass**, including nine new cases covering
  the ascension rollover, the bag-price ascension factor, the one-row-per-base-food
  ledger, `peakTier` reconciliation, the legacy `tier_N` discovery migration, and
  data-driven achievement evaluation.
- `Balance.validate()` passes on server boot with the rewritten invariants.
- `ServerStorage.PacingSim` re-run; `WALL_MARGIN` and `KG_SCALE` are set from it.
- Play mode: clean boot, no runtime errors, icon-only tray rendering, 30-cell
  collection board, six-tile rail.

# NOT VERIFIED

- The tray tooltip (hover and tap) and the ascension celebration were not exercised
  by hand -- Studio's play session kept dropping between MCP calls, so the [*]badge
  thresholds and the Collection/Awards panels were verified by construction and by
  a seeded state, not by a screenshot of each one.
- The place has **not been saved to Roblox**; `SavePlace` cannot be driven from the
  Edit DataModel over MCP.
