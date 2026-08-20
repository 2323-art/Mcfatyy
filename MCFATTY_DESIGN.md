# McFatty's — v1 Design Spec

Status: design locked via grill session. Numbers below are first-pass; expect to retune after playtest.

## 1. Premise
Idle + merge game. Buy mystery meal bags, then **Eat** food for Weight or **Merge** it for income.
Get catastrophically, absurdly fat. Parody branding only ("McFatty's" / "Big M") — no real IP.

## 2. Two resources, distinct jobs

| Resource | Source | Spends on | Resets on rebirth |
|---|---|---|---|
| **Calories** | Passive Cal/s from food on tray | Bags | Yes |
| **Weight (kg)** | **Eating food only** | Nothing — it's progression | Yes |
| **Lifetime Mass** | Sum of all kg ever gained | Nothing — flex stat | **No** |

Weight is never granted passively. Ever. Not offline, not AFK, not by overflow.

## 3. Core loop
Buy bag -> food lands on tray -> **Eat / Merge / Keep** -> weight or income grows -> milestone unlocks -> repeat -> Rebirth.

Three decisions, one per timescale:
- **Which bag?** (spend vs save)
- **Eat or merge?** (progression vs economy)
- **Rebirth or push on?** (cash out vs continue)

## 4. Food tiers (12 in v1)

Four constants ARE the economy. They are generated from ratios in code, not hand-typed:

```
MERGE_INPUTS  = 3      items consumed per merge
MERGE_OUTPUTS = 2      items produced per merge
CAL_RATIO     = 1.65   Cal/s gained per tier
KG_RATIO      = 1.275  eat-weight gained per tier
KG_SCALE      = 2.75   pacing dial (see below)
```

**A merge takes whatever is there.** Three of a tier become two of the next; two become one. Both
are always allowed — a match is never refused — and the rates are deliberately not symmetric:

| Shape | Cost per output | Income | What it is for |
|---|---:|---:|---|
| 3 → 2 | 1.5 | **+10%** | the income play |
| 2 → 1 | 2.0 | **−17%** | buys a tray slot and a tier |

A pair merge *loses* income: two tier-n items earn 2x, the one tier-(n+1) they become earns
CAL_RATIO (1.65x). That cannot be tuned away — making 2→1 positive needs CAL_RATIO > 2 and
invariant 1 caps it at ~1.70 — so the asymmetry IS the design: bundling three is the reward.
**No script may merge a pair.** MERGE ALL and AUTO-MERGE take full groups only, or the upgrade
would quietly cut the income of whoever bought it.

`MERGE_COST = MERGE_INPUTS / MERGE_OUTPUTS = 1.5` is the number the invariants are measured
against, **not MERGE_INPUTS and not the 2→1 fallback**. Balance must hold for the BEST rate a
player can reach, because that is the one they converge on; the worse path is automatically safe
once 1.5 checks out.

| Tier | Food | Cal/s | Eat (kg) |
|---|---|---:|---:|
| 1 | Small Fries | 1.00 | 2.00 |
| 2 | Cheeseburger | 1.65 | 2.55 |
| 3 | 6pc Nuggets | 2.72 | 3.25 |
| 4 | Chili Dog | 4.49 | 4.15 |
| 5 | Bucket Soda | 7.41 | 5.29 |
| 6 | Family Pizza | 12.23 | 6.74 |
| 7 | Combo Platter | 20.18 | 8.59 |
| 8 | Slab of Beef | 33.30 | 10.95 |
| 9 | Popcorn Vat | 54.94 | 13.97 |
| 10 | The Whole Menu | 90.65 | 17.81 |
| 11 | Deep-Fried Everything | 149.57 | 22.71 |
| 12 | The McMonstrosity | 246.79 | 28.95 |

**Invariant 1 - CAL_RATIO vs MERGE_COST sets pacing.** Calories to field a tier-n item are
`MERGE_COST^(n-1) x bagCost`; its output is `CAL_RATIO^(n-1)`. So payback scales by
`(MERGE_COST / CAL_RATIO)^(n-1)`. If CAL_RATIO exceeds MERGE_COST, every tier pays back
faster than the last and the endgame collapses. **This was measured, not guessed:** the original
2:1 / x3 table gave a tier-12 payback of 1.2 seconds and a full run in **2 minutes 54 seconds**.
At 1.5 vs 1.65 the drift is a deliberate, gentle 0.35x - identical to the old 2 vs 2.2, because
1.5/1.65 = 2/2.2.

**Invariant 2 - KG_RATIO must be below MERGE_COST (1.5), not merely below CAL_RATIO.**
Eating three tier-n gives `3 x kg(n)`; merging first and eating both results gives
`2 x KG_RATIO x kg(n)`. At KG_RATIO = 1.5 they tie, and since merging *also* grants income it
would strictly dominate. At 1.275, eating directly gives ~18% more weight. Verified in
EconomyTests: three tier-5s = 15.86 kg eaten directly vs 13.48 kg merged first.

**Consequence worth knowing before it looks like a bug:** weight per Calorie is `0.85^(n-1)`, so
**low-tier food is the cheapest weight in the game**. Eating fries is the efficient way to get fat;
merging is purely an income play. That is what makes "eat now or invest" a real question.

**All three ratios were re-derived, not re-guessed, when the merge went 3->2.** Each is the old
2:1 value scaled by 1.5/2, so payback drift, the eat-vs-merge margin and weight-per-Calorie all
land on exactly the numbers the 2:1 economy was tuned to. What changed is the *shape* of a merge,
not its price. The one thing that could not be preserved is the ladder's **span**: KG_RATIO is
capped below 1.5, so twelve tiers now cover 14.5x in weight where 2:1 covered 343x. KG_SCALE
absorbs that (1.0 -> 2.0); see §12.

Both invariants are asserted by `Balance.validate()`, which runs on server boot and refuses to
start a broken economy.

## 5. Bags — 6 rungs, fixed price

**Rule: cheap bags = best value per Calorie. Expensive bags = best value per tray slot and per
button press.** Asserted in both directions by `Balance.validate()`.

| Bag | Cost | Drops | Unlock | EV Cal/s |
|---|---:|---|---:|---:|
| Snack Bag | 100 | T1-T3 (70/25/5) | 0 kg | 1.2 |
| Big Bag | 870 | T2-T5 (55/30/12/3) | 110 kg | 2.5 |
| Mega Bag | 11,250 | T4-T7 (45/30/18/7) | 300 kg | 7.9 |
| Feast Bag | 165,000 | T6-T10 (40/28/18/10/4) | 1,000 kg | 25.7 |
| Banquet Bag | 1,440,000 | T8-T11 (40/30/20/10) | 3,500 kg | 62.9 |
| Monstrosity Bag | 10,000,000 | T9-T12 (35/30/25/10) | 6,000 kg | 108.5 |

Each rung is ~3x worse per Calorie but 2-6x better per slot. Every bag has a **0.1% jackpot**
pulling one tier above its cap. Unlocks test against `peakKg`, so they survive New Diet.

**Ladder density is load-bearing.** Because merging costs MERGE_COST (1.5) items per output,
fielding a tier-n item takes `1.5^(n-1)` tier-1 items. A player stuck on the cheapest bag needs
*hundreds* of purchases to climb. Widening the gaps between rungs, or gating the upper bags behind
late weights, silently reintroduces the grind.

**Bag prices are coupled to MERGE_COST, and that coupling is invisible until it bites.** A bag
competes with building the same tier by hand out of cheap food. When the merge went 2:1 -> 3:2,
MERGE_COST fell 2 -> 1.5 and every price above Snack Bag was suddenly 2-12x too expensive for what
it delivered: the top of the ladder became worthless and the simulated engaged run went 48:48 ->
81:30. `Balance.validate()` did **not** catch it — it only compares bags to each other, and they
were all wrong by a smooth curve. The costs above are the original hand-tuned numbers scaled by
`(1.5/2)^(expectedTier - 1.35)`.

## 6. Tray
- Start **6 slots**, grow to **~16** via weight milestones.
- Tray full -> **bag purchase blocked** ("TRAY FULL"), never auto-resolved.
- Every verb is a DRAG. **Drop a tile on a match to merge** (two become one of the next, three
  become two); **drop a tile on the tray on the table to plate it**, then click it once per bite to
  eat. A bare click on a tile does nothing. MERGE ALL and EAT ALL were removed; AUTO-MERGE is the
  only bulk merge left.
- Every match highlights green and merges. Requiring three was tried and reported as the merge
  being broken — two matching tiles read as a merge in every other game of this shape.
- Eating costs **`Balance.eatBites(kg)` clicks**, falling from 5 at 60 kg to 1 by 6,000 kg. Rising
  counts would tax progress and pile presses onto the back half of a run; falling counts pay the
  player back for weight they earned.
- Nothing is ever deleted, overflowed, or auto-eaten. Scarcity is the point — it's what makes Mega Bags worth their worse EV.

## 7. Weight milestones (Run 0)

Deliberately front-loaded — the first three land inside the opening 3.5 minutes. A new player who
sees no body change early simply leaves. Even spacing across 45 minutes tests badly.

| kg | Reward |
|---:|---|
| 75 | +1 tray slot — "Shirt's getting tight" |
| 110 | +10% Metabolism, **Big Bag** |
| 175 | +1 tray slot |
| 300 | +1 tray slot, **Mega Bag** |
| 550 | +1 tray slot, offline cap 10h |
| 1,000 | +25% Metabolism, **Feast Bag** |
| 2,000 | +2 tray slots |
| 3,500 | +1 tray slot, offline cap 12h, **Banquet Bag** |
| 6,000 | +2 tray slots, **Monstrosity Bag** |
| 8,000 | +50% Metabolism |
| 10,000 | **YOU'VE OUTGROWN THE RESTAURANT** — rebirth unlocked |

Tray grows 6 -> 15 slots. Size stages are aligned 1:1 with these thresholds, so every milestone
produces a visible body change.

`calMult` is a **Metabolism** multiplier: it scales Cal/s *and* kg-per-eat together. The pacing
tuning assumes both. Applying it to income only would silently lengthen every run — which is why
the labels say Metabolism, not Cal/s.

**Everything permanent derives from one stored number, `peakKg`.** Nothing is accumulated or
"claimed", so there is no double-grant bug, no drift, and the entire permanent state can be
recomputed from scratch after any save corruption or rebalance.

## 8. Rebirth — "New Diet"
**Resets:** Weight, Calories, Cal/s, tray contents, food.
**Survives:** tray slot count, bag unlocks, offline-cap upgrades, restaurant tier, Lifetime Mass.
**Grants:** permanent **Metabolism** multiplier (Cal/s and kg-per-eat), and a **new size era ceiling**.

| Diet | Weight ceiling | Visual peak |
|---|---|---|
| Run 0 | 60 -> 10,000 kg | fills / outgrows restaurant |
| Diet 1 | -> 100,000 kg | building-sized |
| Diet 2 | -> 1M kg | neighborhood |
| Diet 3 | -> 100M kg | city |
| Diet 4 | -> 10B kg | country |
| Diet 5+ | absurd | planet -> star -> galaxy |

Rebirth is a **spectacle**: character rapidly deflates 10,000 -> 60 kg, then "DIET #N COMPLETE / Metabolism x4.2 / New maximum size: CITY CLASS".

**v1 ships Run 0 + the first rebirth only.** Later size eras are post-launch.

## 9. Offline / AFK
- **Offline:** `Cal/s x time x 75%`, capped at **8h** (upgradable via Fridge -> Freezer -> 24/7 Kitchen).
- **AFK in-game:** 100% Cal/s, nothing auto-eaten.
- **Weight never grows while away.** Non-negotiable — it's what protects the Eat/Merge decision.
- **Return screen:** offline Calories + **bonus bags** (1h -> 1 bag, 4h -> another, 8h -> rare bag) + current weight + next milestone. Then the player immediately thinks "I'm 13 kg short, I'll eat these instead of merging."
- The "while you were away you ate 48,291 burgers" line stays as **flavor text only** — it grants no weight.

## 10. Presentation
- **Tray = 2D ScreenGui grid.** Drag item onto item to merge; tap-hold to eat.
- **Restaurant + character = 3D.** All 3D effort goes into the fat guy.
- **Movement:** walk freely while normal-sized; **anchored once huge** ("you're too fat to move"). The anchor threshold is also the render-mode swap point.
- **Character visuals: DEFERRED.** All systems call a single `SetSizeStage(n)`. Whatever art approach is chosen later plugs into that. Nothing is blocked by this.
- Never scale the avatar linearly with weight. Discrete stages; at extreme sizes change camera/environment instead of growing studs.

## 11. v1 scope

**In:**
- Core loop (bags / eat / merge / tray)
- 12 food tiers, 4 bags, ~10 weight milestones
- Rebirth #1
- Offline income + return screen + bonus bags
- **Tutorial, guided first 60s** — forced bag, forced merge, forced eat, reach 100 kg, then stop teaching. Mandatory: three non-obvious verbs.
- **Daily reward / streak** — rewards are mostly **bags, not Calories**, so the economy stays intact.

**Patch 2:** leaderboards (Lifetime Mass, most rebirths, highest tier), random events (McHappy Hour, food truck, fry rain), shared restaurant, later size eras, monetization.

**Cut regardless:** chair-collapse and stuck-in-door gags (animation/physics cost, near-zero validation value).

## 12. Pacing — measured, not estimated

A headless simulator (`ServerStorage.PacingSim`) drives the **shipped** `Economy` module and
models a real player: one press buys one FILL TRAY, merges one group, or eats one item, on a
button-press budget.

| Player | Presses/sec | Time to first rebirth | Presses | Merges / Eats |
|---|---:|---:|---:|---:|
| Engaged, no upgrades | 1.5 | **48:05** | 4,327 | 216 / 1,338 |
| Engaged, smart upgrades | 1.5 | 49:14 | 4,431 | 248 / 1,342 |
| Casual, half-watching | 0.6 | 95:24 | 3,434 | 164 / 1,050 |
| Value hoarder, cheapest bag | 1.0 | 108:06 | 6,486 | 444 / 2,138 |

Presses include **bite clicks**: eating costs `Balance.eatBites(kg)` presses, not one, and at ~2
per eat averaged over a run it is the largest single line in the budget. That is a purely
client-side cost the server never sees, so PacingSim is the only thing in the codebase that can
catch it — it charges them explicitly. On its own the plate pushed the engaged run 48:34 → 60:27;
`KG_SCALE` 2.0 → 2.75 bought it back.

Opening beats for the engaged run: 75kg at 1:16, 110kg at 2:17, 175kg at 3:07, 300kg at 4:54.

**Watch the merge/eat mix.** Under 2:1 the same engaged run merged 1,594 times and ate 1,003; it
now merges ~230 and eats ~2,000. Three of a kind is simply much rarer on a 6-16 slot tray than two
of a kind, so the tray leans on AUTO-MERGE and on eating rather than on hand-merging. Total presses
are unchanged (4,363 vs 4,392), so the run did not get more expensive — but it did change
character, and that is a design question, not a bug.

**The press budget is not optional realism.** An earlier sim allowed unlimited actions per second
and concluded that spamming the cheapest bag was *four times faster* than any other strategy —
because 34,000 purchases cost it nothing. Click budget is exactly what expensive bags are sold
for, so without a budget the sim cannot see the tradeoff it exists to measure.

**KG_SCALE is the pacing dial.** Reach for it before touching bag prices:

- Scaling bag costs uniformly slows the opening by the same factor as the endgame — a x4 pass
  buried the first milestone 15 minutes deep.
- Steepening only the upper bags does nothing at all: players just buy a cheaper rung.
- KG_SCALE moves total run time a great deal while barely touching the opening. Measured sweep
  (before bite clicks were charged): 1.0 -> 81:30, 2.0 -> 48:34, 4.0 -> 29:36, 8.0 -> 19:03,
  16.0 -> 12:52, while the first milestone only moves 2:10 -> 1:23 -> 1:06 -> 0:53 -> 0:47. With
  bite clicks charged, **2.75 -> 48:05**.

KG_SCALE is 2.75 rather than 1.0 for two reasons. First, KG_RATIO is hard-capped below MERGE_COST
(1.5) by invariant 2, so twelve tiers now span 14.5x in weight where 2:1 spanned 343x — late-game
food weighs far too little against a 10,000 kg target, and the un-rescaled run measured 81:30
against the old 48:48. Second, eating costs `Balance.eatBites(kg)` presses instead of one, which
pushed a corrected 48:34 back out to 60:27 on its own. 2.75 buys back both: **48:05 with 4,327
presses** against the 2:1 build's 48:48 / 4,392. Re-measure if any of the three ratios,
`EatCalShare`, or the bite counts move.

## 13. Technical constraints

- **Server-authoritative.** The client sends *intents* ("fill tray with mega bags"), never values.
  The server validates, rolls RNG, and replies with state. Verified: 5 forged/malformed requests
  (rebirth under threshold, out-of-range slots, unknown action, non-string action, wrong arg type)
  all rejected without mutating state or erroring.
- **Per-UserId session objects** from day one. No globals, no singletons. This is what makes the
  shared-restaurant retrofit a one-day job instead of a rewrite.
- **Session-locked saves.** Roblox can briefly run two servers holding one player (teleports,
  rejoin during shutdown). Without a lock both write and the later wins — which players experience
  as "the game deleted my progress". Every load claims via `UpdateAsync`; every save re-checks it
  still owns the claim, and a save that lost the lock is *discarded*, not forced.
- **DataStore is fetched lazily and guarded.** An unpublished place throws on `GetDataStore`, and
  at the top level that throws at *require* time, taking down every dependent module. It now
  degrades to in-memory saves instead of breaking the boot.
- **The join path is idempotent and self-healing.** `PlayerAdded`, the start-up backfill, and a
  heartbeat catch-up can all reach the same player; loading twice would clobber live progress.
- **No BigNumber module.** Luau doubles reach 1e308; planetary mass is 6e24. Full precision is
  kept internally and the UI abbreviates (113 Cal/s, 5.84K, 1.27M).
- **Balance invariants are asserted on boot.** `Main` refuses to start a build where merging is
  strictly dominant or the endgame collapses. Both are easy to break with an innocent tuning tweak
  and impossible to spot by eye.

### Bootstrap and softlock guards (not in the original concept, but required)

The first sim run opened **zero bags**: an empty tray earns 0 Cal/s, so the first bag is
unaffordable forever. The same hole is a mid-game softlock, because eating destroys the item *and*
its income — a player who ate their whole tray could never buy again.

- `StartingTray = {1, 1, 1, 2}` — players open with food already earning, and with one complete
  merge group so the tutorial's merge step is satisfiable on a fresh save.
- `BaseCalPerSec = 15`, plus a floor of 10% of the run's peak Cal/s. Scaling with peak rather than
  a flat number keeps recovery proportional, so a late-game player who feasts their tray away
  rebuilds in about the same number of seconds as an early one.

## 14. Build status

Everything below is built and verified running in Studio (place **"Place1" — unsaved, save it**).

```
Workspace/Restaurant             baked map: checkerboard floor, walls, windows,
                                 counter, McFatty's board, booth, lighting, spawn
ReplicatedStorage/Shared/
  Config/  Food, Bags, Progression, Balance, Upgrades
  Economy                        all rules, pure functions, no Roblox APIs
  Format                         number abbreviation (1.27M, 5.84K)
ServerScriptService/Server/
  Main                           boot, remotes, invariant gate
  Services/ DataService          session-locked saves, lazy guarded DataStore
            PlayerService        per-UserId state, income, remotes, daily, upgrades
            CharacterService     size stages -> body scale, anchoring
StarterPlayer/.../Client/
  Main                           UI bootstrap, responsive UIScale, sound routing
  CameraController               keeps the camera outside a huge body
  UI/  Theme, HUD, Tray, Shop, UpgradesPanel, Pending,
       Panels, Tutorial, Rail, RebirthCard, Sounds
ServerStorage/
  MapBuilder, PacingSim, EconomyTests          tooling, not shipped
```

**Interface** follows the reference layout: four stat pills across the top
(weight with milestone bar, Cal/s, Calories, rebirths), a left icon rail
(shop / upgrades / rebirth / daily, with a notification pip), a right panel that
swaps between BUY BAGS and UPGRADES, a floating NEXT MILESTONE card, the tray as
a tile strip with tier badges and per-item rates, a MERGE / EAT / AUTO-MERGE
action bar, and the NEW DIET card bottom-right.

**Upgrades** are the Calorie sink beyond bags — Auto-Merge tier, Bigger Table,
Walk-In Fridge, Slow Metabolism. All permanent, all surviving New Diet, priced
steeply so they are a late-run and across-diets purchase rather than competing
with food in the opening minutes.

**Auto-Merge** combines food up to its purchased tier and never eats, so the
Eat-vs-Merge decision on high-tier food stays with the player; it only removes
the busywork of pairing fries. It runs between purchases inside FILL TRAY,
because it frees slots.

**`EconomyTests` passes 34/34** — merge cascades, the Eat-vs-Merge tradeoff, tray
limits, softlock recovery, offline (Calories yes, weight no), the offline cap,
rebirth preserving slots/bags/metabolism, daily streak advance/reset/wrap, and
pending-bag consumption.

**Verified live:** income accrual, the full buy -> merge -> eat loop over real
remotes, tray-full blocking, five forged/malformed requests rejected without
mutating state, size stages 1-13 with the width/height ratio climbing 0.62 ->
0.91, upgrade purchases, auto-merge tiers, daily reward and free-bag opening.

### Bugs the build surfaced

- **`Model:ScaleTo()` multiplies Humanoid scale values.** Mixing it with explicit
  girth compounded, and because ScaleTo is absolute a second call at an unchanged
  stage no-opped while girth had already been reset -- the character became
  *thinner* on respawn. Now sets absolute scale values only.
- **The default Baseplate z-fights the restaurant floor.** Both top surfaces sat
  at y = 0, producing jagged grey tearing across the tiles. MapBuilder now
  removes it and drops the floor to y = 0.05.
- **`NormalId.Front` is the -Z face**, so the menu board faced away from the room.
- **Roblox Instances reject arbitrary fields**, so caching child labels on slot
  buttons failed silently then errored on read.
- **`GetDataStore` throws at require time** in an unpublished place, taking down
  every dependent module. Now lazy and guarded.
- **White UI text over a white wall is invisible.** Rail captions and the tray
  counter draw onto the 3D world, so they need text outlines; panel text does not.
- **A toggle initialised to its own open value closes itself.** `setPanel("shop")`
  at startup hid the shop.
- **Studio memoizes ModuleScripts**, so balance edits silently did nothing until
  the sim loaded a fresh clone. A helper module cannot fix this -- it gets cached
  too; the loader snippet lives in PacingSim's header.

## 15. Not built / open

- **Monetization** — deliberately out of v1. `IncomeMultiplier`, `BonusSlots` and
  `OfflineCapHours` exist in player state so passes drop in without re-tuning.
- **Leaderboards and random events** — patch 2, by decision.
- **Size eras past run 0** (building -> city -> planet) — these need environment
  and camera swaps, not just bigger bodies.
- **Character art** — currently avatar scaling. Everything downstream reads one
  integer (`SizeStage`), so a bespoke model swaps in without touching logic.
- **Upgrades are not modelled in the pacing sim.** The sim's policies buy bags
  only, so measured run times are an upper bound; a player who buys Slow
  Metabolism will finish faster than 35 minutes.
- **VFX** — sound is in (engine built-ins, pitch-rising merge cascades), but
  there are no particles, no merge animation, and no eating animation.
- All numbers pending real playtest; the sim is a model, not a player.
