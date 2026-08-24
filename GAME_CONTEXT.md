# McFatty game context

This is the living architectural and product context for the McFatty Roblox experience. Read it before planning or changing the game. Roblox Studio is the source of truth; this document is a maintained model of that source, not a substitute for inspecting it.

## Snapshot metadata

- Last verified through Roblox Studio MCP: **2026-08-23** (Asia/Kuala_Lumpur), after the **permanent restaurants, sliding doors, worker automation, grouped UI, seated meals, infinite run-upgrade, and endless pacing pass**.
- Connected Studio display: **McFatty**
- Place ID: `74686069419969`
- Universe/game ID: `10741900895`
- Place version observed: `112` before the restaurant-entrance edit.
- Save state at handoff: Studio's authoritative Edit DataModel contains the completed and verified changes, but **Save to Roblox is still pending**. The Studio MCP connection has no save command, and the Windows-control helper failed its initialize/retry/reset/retry sequence with `failed to write kernel assets: The system cannot find the path specified`.
- Creator ID: `11272741268`
- DataModel name reported inside Studio: `Place1`
- Studio mode during inspection: `Edit`
- Local source state at inspection: no Rojo project, package manifest, local Luau tree, or Git worktree was present; Studio contained the implementation.
- Implementation style: strict Luau modules, no external framework or package manager observed.

## Product summary

McFatty is a comedic idle/merge progression game set in a fast-food restaurant. The player buys mystery food bags, keeps food on a tray to generate Calories per second, merges matching food to improve its tier and income, and physically eats food to gain weight. Weight unlocks bags, tray slots, metabolism bonuses, offline capacity, larger character stages, and eventually a rebirth called a **New Diet**.

The central decision is **merge for income versus eat for weight**:

- keeping food produces Calories per second;
- a good three-item merge improves income and tier efficiency;
- eating removes food from the tray, grants weight, and permanently digests only part of that food's income;
- weight advances unlocks and the run goal;
- at the current rebirth target (`10,000 kg` on the first run, then `3.5x` more per completed diet), a New Diet resets the run while retaining permanent progress and increasing metabolism.

Since the Phase 1 progression pass the run is shaped by two gates and two currencies:

- **Kitchen Level** caps the highest food tier that can be produced by ANY path (merge, Auto-Merge, or bag roll), while tray slots buy only throughput. That split is what stopped a big tray plus Auto-Merge from climbing the ladder on its own.
- **Merging costs Calories**, on a curve steeper than the income it produces, so climbing inside your Kitchen is an investment rather than a formality.
- **Calories** buy bags, Run Upgrades and Kitchens, and all of it resets on a New Diet.
- **Diet Points**, paid out by a New Diet and scaled by how far past the target the run went, buy **Permanent Perks** -- the only purchase that survives. The intended loop is: progress fast, hit a soft wall (the next Kitchen, the next bag, the next upgrade all get expensive together), take the New Diet, spend the points, and find the old wall trivial.

The experience is intentionally loud and comedic: chunky UI, saturated colors, restaurant-scale character growth, milestone celebrations, and physical table-based eating.

The restaurant is also an automation layer inspired by Tap Titans 2:

- every plot permanently contains a complete restaurant, even while unclaimed; unclaimed signs read **AVAILABLE**, but no worker characters exist until an owner buys them;
- an owner can hire up to ten unique workers. Each worker owns a genuine six-slot tray, buys bags, merges only full groups, eats the highest food available, and spends from the owner's shared Calorie pool while respecting a configurable reserve;
- worker identity fixes the absolute food cap at T4/T8/.../T40, and the effective cap is always the lower of that identity cap and the owner's current Kitchen ceiling;
- worker levels are infinite. Personal production grows every level and fixed owner-wide buffs unlock at levels 10, 25, 50, and 100;
- workers add per-run digestion/Calories, not player kilograms. Their visible body growth is capped at about 1.8x;
- New Diet resets hires, worker levels, trays, kilograms, and digestion, while preserving the reserve, pause, and per-worker enable preferences;
- offline worker output is exactly 50% of its online rate within the existing offline-time cap;
- playing without staff remains possible but is tuned to take about three times as long in the opening run.

Progression is endless. Kitchen, Bigger Bites, Metabolism, food tiers, worker levels, and the visible difficulty band continue past 100. Finite convenience/chance upgrades retain intentional caps. The visible band is `diets + 1` (1x, 2x, 3x...), while the effective pacing divisor rises quickly through diet 8 and then asymptotically approaches 40x so staffed runs settle below the intended four-hour ceiling.

## Authority and data flow

```text
Player input
  -> client UI/world interaction
  -> ReplicatedStorage.Remotes.Action (intent only)
  -> PlayerService validation and rate limit
  -> Shared.Economy mutation of server-owned session state
  -> progress notifications and authoritative snapshot
  -> Sync/Notify remotes
  -> client UI, camera, audio, effects, and world presentation
```

Ownership rules:

- **Server:** live player and worker state, Calories, weight, all trays, upgrades, RNG, bag rolls, rewards, rebirth eligibility, persistence, milestone detection, character size, worker presentation, seated-meal locking, doors, and station prompt validation.
- **Shared:** pure economy rules, exact action-shape validation, the deterministic meal state machine, balance/config data, progression data, and number formatting.
- **Client:** input, local presentation, smooth Calories prediction between authoritative syncs, grouped Food/Staff/Progress/Records UI, drag/drop, local bite and upper-body meal animation for a server-reserved meal, collection-board rendering, camera framing, sounds, and effects.
- **Workspace/builders:** baked district geometry (permanent lots, roads, signs, boundary) plus the authoritative restaurant template in ServerStorage. Destructive full rebuilds remain explicit; boot migrations only rebuild a missing/legacy district, a missing template, or a missing station set.

The client never sends resulting currency, weight, tiers, rolls, or upgrade levels. It sends only named intent plus minimal identifiers or indices.

## Runtime lifecycle

### Server startup

`ServerScriptService.Server.Main` performs this sequence:

1. Ensures `ReplicatedStorage.Remotes` exists.
2. Ensures three `RemoteEvent` instances exist: `Action`, `Sync`, and `Notify`.
3. Runs `Balance.validate()` and refuses to start if an economy invariant fails.
4. Resolves whether DataStore persistence is available and warns when Studio uses memory fallback.
5. Runs `MapBuilder.ensureTodoWorld()` and `StationsBuilder.ensureTodoStations()`. Both are idempotent and non-destructive on a valid baked district: the first rebuilds only a missing/pre-district layout, restores a missing template, or additively restores a missing permanent restaurant; the second repairs the template's missing station set and performs the prompt-hardening sweep.
6. Starts `PlayerService` and `CharacterService`.
7. Starts `PlotService`, `SeatingService`, `WorkerService`, `StationService`, `DoorService`, and `LeaderboardService`.

`Workspace.Restaurants` is baked as the district: `RoadNetwork`, `Boundary`, and twenty permanent plot containers (`Plot01`..`Plot20`). Every plot already contains `Lot`, `OwnerSign`, and a complete empty `Restaurant` cloned from `ServerStorage.RestaurantTemplate`. The destructive `MapBuilder.build()` path is not called on normal boot; `ensureTodoWorld()` additively restores only a missing permanent restaurant.

`PlotService.start()` scans that folder. If it is not there yet, the scan is **retried once a second for thirty seconds** and joins are held in a pending queue rather than failing: scanning once and giving up is what used to leave every player on the server without a restaurant.

### Player join

1. `PlayerService` marks the user as loading to prevent duplicate concurrent loads.
2. `DataService.load()` claims the profile with a session lock or returns an in-memory Studio state.
3. `Economy.migrate()` fills missing fields from a fresh schema.
4. Offline Calories and bonus bags are applied.
5. The daily reward is claimed if eligible.
6. Player attributes are mirrored and an authoritative `Sync` snapshot is sent.
7. `CharacterService` binds character/attribute events and applies the current size stage.

### During play

- `RunService.Heartbeat` accrues Calories continuously on the server.
- The same heartbeat advances bounded worker tray cycles; excess cycle credit is retained rather than expanded into an unbounded lag-spike loop.
- `DoorService` opens registered double sliding doors from their E prompt, keeps collision disabled while open, and closes two seconds after the doorway sensor becomes empty.
- `SeatingService` anchors the owner at the booth and locks movement for the lifetime of a valid meal reservation, then restores the humanoid on completion, cancel, failure, respawn, or leave.
- A state snapshot is sent every `0.5` seconds.
- Autosave runs every `60` seconds.
- Progress events are checked immediately after successful mutations.
- The client predicts the visible Calories total between syncs and re-bases on every snapshot.

### Leave and shutdown

- On leave, `lastSeen` is updated, the profile is saved, and the session lock is released.
- `BindToClose` saves all sessions and releases locks.
- A lost/foreign active lock is not overwritten; live load failure kicks the player rather than risking a fresh save overwriting real progress.

## Networking contracts

All three remotes are created at runtime by the server entry point. In Edit mode, the `Remotes` folder may be empty.

### `Action`: client to server

`Action:FireServer(actionName, ...)` accepts these intents:

| Action | Arguments | Server behavior |
|---|---|---|
| `BuyBag` | `bagId: string` | Validate unlock, room, and Calories; server rolls one item; then auto-merge. |
| `FillTray` | `bagId: string` | Buy while legal, running auto-merge between purchases, with a hard maximum of `512` purchases per request. |
| `BuyUpgrade` | `id: string` | Validate known upgrade, level cap, and cost. Buying an `automerge` level immediately runs `runAutoMerge` so the raised tier cap applies to the tray already held. |
| `BuyUpgradeBulk` | `id: string, amount: 1|10|100|"max"` | Buy up to the selected amount; `"max"` is server-bounded to 1,000 iterations. |
| `BuyKitchen` | none | Buy the NEXT Kitchen level (there is only one, so no argument is accepted -- taking a target level would mean validating that the client is not asking to skip four). Runs `runAutoMerge` afterwards for the same reason `BuyUpgrade` does: the raised ceiling may already have full groups under it. |
| `BuyKitchenBulk` | `amount: 1|10|100|"max"` | Buy consecutive Kitchen levels without accepting a client-supplied target level. |
| `BuyPerk` | `id: string` | Spend **Diet Points** on one permanent perk level. Validates a known perk id, the level cap and the point balance. Buying `autochef` re-runs Auto-Merge so the perk pays out in the current run. |
| `ToggleAutoMerge` | none | Flip the saved toggle and merge eligible full groups when enabled. |
| `Merge` | `slotA: number, slotB: number` | Validate two occupied, distinct, matching slots and merge result. A two-item merge rolls the pair bonus on the session RNG. |
| `BeginEat` | `tier: number` | Reserve one owned tier and return deterministic bite count/duration; food remains owned. Additionally requires a **meal-budget token** and that the character is **within `Balance.EatRadiusStuds` (110) of its own plot**. |
| `CompleteEat` | none | Consume the reserved tier only after its server ready time. Early, replayed, or stale completion cancels without reward. |
| `CancelEat` | none | Clear the active reservation without consuming food. |
| `OpenPending` | `bagId: string` | Open one owed bag for free if the tray has room; server rolls it. |
| `HireWorker` | `index: integer 1..10` | Validate identity, Kitchen requirement, ownership, and live cost, then create the run worker. |
| `LevelWorker` | `index: integer 1..10, amount: 1|10|100|"max"` | Buy infinite personal levels with bounded bulk work. |
| `ToggleWorker` | `index: integer 1..10` | Toggle that worker's automation preference. |
| `SetWorkerReserve` | `percent: integer 0..90` | Set the protected share of the current Calorie pool. |
| `ToggleWorkersPaused` | none | Pause/resume every worker without losing state. |
| `Rebirth` | none | Validate `kg >= Progression.rebirthKgFor(diets)`, award Diet Points, then reset the run (including upgrades and Kitchen). |
| `Debug` | `what: string, value: number` | Studio-only: `setKg` or `addCalories`; rejected outside Studio. |

Every action first passes `Shared.ActionSpec`: exact argument count, bounded known IDs, finite integer slots/tiers, and Studio-only bounded debug values. Legacy one-shot `Eat` is rejected. Economy functions re-check state preconditions.

Rate limiting, and the ordering is load-bearing:

- the token bucket is `25` units per second and is **refilled and charged BEFORE validation**. Validation used to run first and return early, which made malformed input free -- a client fuzzing or spamming the remote never touched the rate limit at all;
- a **failed** validation costs `ActionSpec.invalidCost()` = `3` units, deliberately more than a well-formed action;
- an action refused for being over budget or in cooldown still costs `1`, so an emptied bucket cannot be hammered for free;
- `FillTray` costs `8` units and has a `0.6` second cooldown; `BeginEat` costs `2`; everything else costs `1`;
- `BeginEat` additionally spends from a **separate meal bucket** (`Balance.MealBurst` 12, `Balance.MealsPerSecond` 3). Packets are the wrong unit for the one action that produces weight;
- **a failed action does not send a `Sync` snapshot.** Every handler validates before mutating, so a failure leaves state unchanged and the client's copy correct; the explicit `Rejected`/`Meal` notification plus the 0.5s heartbeat cover it.

### `Sync`: server to client

Authoritative snapshot fields:

- `kg`, `peakKg`, `lifetimeKg`, `diets`, `difficultyBand`, `difficultyFactor`
- `calories`, `calPerSec`, `digestCalPerSec`, `workerCalPerSec`
- `tray` (cloned array of food tier numbers), `slots`
- `upgrades`, `upgradeCosts`
- `kitchen`, `kitchenName`, `kitchenMaxTier`, `kitchenCost` (the Kitchen ladder has no gameplay maximum)
- `ascension`, `peakTier`, `peakTierEver`
- `mergeCosts` (map of input tier to that player's LIVE Calorie merge price)
- `dietPoints`, `lifetimeDietPoints`, `perks`, `perkCosts` (nil per perk at max)
- `dietPointsIfNow` (what a New Diet taken at this instant would pay)
- `bagCosts` (map of bag ID to that player's LIVE price), `priceScale`
- `rebirthKg` (this player's current New Diet target)
- `autoMergeEnabled`, `autoMergeTier`
- `eatBites`, `chewSpeed`, `sizeStage`, `multiplier`
- `bigBiteMult`, `luckyBiteChance`, `doubleMealChance`
- `canRebirth`, `nextMilestone`
- `pendingBags`
- `discoveredFoods` (cloned map keyed by stable base-food IDs such as `food_SmallFries`)
- `foodStats` (lifetime per-base-food `{ eaten, kg, topAsc }` rows)
- `achievements` (map of server-earned achievement IDs to `true`)
- `workers` (ten server-derived rows with hire/level/cost/cap/tray/kg/digestion/meal/buff presentation), `workerReserve`, `workersPaused`

Do not remove or rename these without updating every client consumer and this document.

`kitchenCost`, `mergeCosts` and `perkCosts` are sent for the same reason as `bagCosts`: all three carry `priceScale` and/or a permanent perk discount, so a client recomputing them from config would show the wrong price to exactly the players who had earned the discount. `dietPointsIfNow` is sent because the "take the diet now or push further?" decision cannot be made against a number the player cannot see.

`bagCosts`, `priceScale` and `rebirthKg` exist because all three MOVE with completed diets. Any client that reads `Bags.List[i].cost` or `Progression.RebirthKg` directly is correct exactly once -- on a player's first run -- and then lies. `Client/UI/Shop`, `Client/UI/Tray`, `Client/UI/RebirthCard` and `Client/Main` all read the snapshot value with the config value as a pre-first-sync fallback only.

### `Notify`: server to client

| Kind | Payload | Presentation |
|---|---|---|
| `SizeStage` | stage number | Growth toast and world size-up effect. |
| `Milestone` | `{ kg, label }` | Toast, flash, confetti, and shake. |
| `Rebirth` | `{ diets, metabolism }` | New Diet modal and world effect. |
| `Offline` | `{ seconds, capped, calories, bags }` | Welcome-back modal. |
| `Daily` | `{ streak, bagId, count, day }` | Daily reward modal. |
| `Station` | station ID | Open/nudge the matching UI destination. |
| `Jackpot` | rolled tier number | Gold world burst and floating jackpot text. |
| `MergeBonus` | `{ tier }` | Gold "DOUBLE MERGE" float, confetti, and tray pop for a pair merge that paid two. |
| `Meal` | `{ status, tier?, bites?, duration?, gained?, kg?, reason?, lucky?, doubled?, meals? }` | Start, complete, or cancel the client plate/hold presentation. `lucky`/`doubled`/`meals` are presentational only — the meal is already resolved server-side, so a client ignoring them loses a celebration, not weight. |
| `Discovery` | `{ tier, id }` | Reveal and celebrate the collection cell. |
| `Ascension` | `{ ascension, tier, name }` | Menu Ascension crossed. Deliberately the loudest beat in the merge loop -- the player has just surpassed the entire menu and the whole roster restarts at a higher power level. Fired from a derived watermark checked after ANY mutation, so Auto-Merge and Fill Tray cascades raise it too, not only a hand merge. |
| `Achievement` | `{ id, name, desc, icon }` | A data-driven achievement was earned. Quieter than an ascension on purpose: several can land on one action. |
| `Rejected` | `{ action, reason }` | Explicit feedback for a well-shaped action that fails state validation. |

### Mirrored `Player` attributes

For read-only visibility outside the server VM:

- `Kg`
- `PeakKg`
- `Diets`
- `CalPerSec`
- `SizeStage`
- `ProfileLoaded`
- `WorkerCount`

`Sync` remains the real client state channel.

## Persisted player state

DataStore name: `McFattyPlayer_v1`; key format: `p_<UserId>`.

Document envelope:

- `state`: the economy state below;
- `lock`: `nil` or `{ jobId, at }`.

Session locks expire after `10` minutes and loads retry up to `5` times with bounded waits.

Economy state fields:

| Field | Meaning/lifecycle |
|---|---|
| `version` | Schema version, currently `5` (worker run state and automation preferences). |
| `peakKg` | Highest lifetime weight; permanent source for milestone rewards and bag unlocks. |
| `diets` | Completed New Diet count; permanent. |
| `lifetimeKg` | Total weight gained; permanent and shown on the record board. |
| `kg` | Current-run weight; resets to `60`. |
| `calories` | Current spendable currency; resets on New Diet. |
| `tray` | Ordered array of tier numbers; resets to starting tray. |
| `peakCalPerSecRun` | Current-run peak tray income used by the recovery floor. |
| `kitchen` | Current Kitchen level (`1`..unbounded); caps the highest tier that can be PRODUCED by any path. Clamped only at the bottom -- an upper clamp would restore the ceiling the infinite food ladder removed. **Resets to `1` on New Diet.** |
| `digestion` | Current-run raw, unmultiplied Cal/s retained from eaten food; resets on New Diet. |
| `incomeMultiplier` | Monetization hook, default `1`. |
| `bonusSlots` | Monetization hook, default `0`. |
| `offlineCapBonusHours` | Monetization hook, default `0`. |
| `upgrades` | Map of upgrade ID to level. **Per-run: cleared entirely by New Diet.** These are the Run Upgrades. |
| `workers` | Exactly ten per-run rows: `hired`, infinite bounded `level`, private six-slot `tray`, `kg`, raw `digestion`, `cycle`, `lastTier`, and `meals`. Hires and all run values reset on New Diet. |
| `workerReserve` | Saved preference in `[0, 0.9]`; defaults to `0.20` and survives New Diet. |
| `workersPaused` | Saved global automation preference; survives New Diet. |
| `workerEnabled` | Ten saved per-worker booleans; survive New Diet even though hires reset. |
| `dietPoints` | Permanent meta currency, paid out by New Diet and spent only on perks. |
| `lifetimeDietPoints` | Total points ever earned; permanent flex stat, never spent. |
| `perks` | Map of perk ID to level; **permanent, the only lasting purchase in the game.** |
| `autoMergeEnabled` | Saved toggle; defaults true. |
| `peakTier` | Highest global food tier produced in the current run; resets on New Diet. |
| `peakTierEver` | Highest global food tier ever produced; permanent high-water mark for Collection and achievements. |
| `pendingBags` | Map of bag ID to owed count from offline/daily rewards. |
| `discoveredFoods` | Permanent map of stable base-food IDs (`food_<BaseId>`) to `true`; all Menu Ascensions of one food share one entry. |
| `foodStats` | Permanent per-base-food ledger containing lifetime `eaten`, lifetime `kg`, and highest eaten Menu Ascension (`topAsc`). |
| `achievements` | Permanent map of known, server-earned data-driven achievement IDs to `true`. |
| `lastSeen` | Unix time for offline calculation. |
| `streak` | Daily streak length. |
| `lastClaimDay` | Integer UTC-like day bucket (`time // 86400`). |

Migration is **validating**, not merely additive. Missing fields are copied from a fresh default state, and then every field present in the document is coerced and clamped, because a save is untrusted input in the same sense a remote payload is -- a present-but-wrong value propagates into the economy and back into the next save, so one bad write becomes permanent:

- `kg`, `peakKg`, `lifetimeKg`, `calories`, `digestion`, `peakCalPerSecRun`: rejected if NaN/infinite, clamped to sane ranges. `peakKg` is reconciled upward against `kg`, since every permanent unlock derives from it;
- `diets`, `bonusSlots`, `streak`, `lastClaimDay`: floored integers in range;
- `incomeMultiplier`: floors at `1` (zero would freeze the economy outright);
- `tray`: non-integer, non-numeric and out-of-range global tiers are discarded through `Food.isValidTier` (bounded by `Food.AbsoluteMaxTier` for hostile-input safety), then trimmed from the end to the current slot count once slots are knowable;
- `upgrades` and `perks`: unknown IDs dropped; finite definitions clamp to their current `maxLevel`, while infinite Bigger Bites and Metabolism clamp only to the hostile-save sanity ceiling;
- `kitchen`: clamped at `MinLevel` and the hostile-save sanity ceiling; there is no gameplay maximum;
- worker rows are reconstructed to exactly ten entries; private trays are trimmed to six valid tiers and the lower of identity/Kitchen caps, levels are bounded, and reserve/enable/pause preferences are repaired independently;
- `dietPoints` and `lifetimeDietPoints`: floored integers, bounded at `1e9`, and lifetime is reconciled upward against the balance -- a corrupted balance here would buy every perk in the game permanently, with no way back;
- the tray is deliberately NOT clamped to the Kitchen ceiling. A save can legitimately hold food the current Kitchen could not make (bought at a higher Kitchen earlier in the session); the Kitchen gates production, and confiscating held food would read as the save eating the tray;
- `pendingBags`: unknown bag IDs and non-positive counts dropped;
- `lastSeen`: clamped to now, so a future timestamp cannot manufacture offline earnings;
- `discoveredFoods`: sanitized against the current food ladder.

`version` is set last.

Persistence failure behavior distinguishes **permanent** from **transient**, and conflating the two was a real bug:

- `apiAccessDenied` is permanent and set ONLY by an API-access error (Studio without API access, which never self-heals). In that mode in-memory saves are correct.
- Every other failure -- a throttle, an outage, a transient `GetDataStore` throw -- is temporary. The handle is retried on a `20` second cooldown. Previously a single failed `GetDataStore` latched the whole server into memory-only mode for its lifetime, so one bad second at startup meant every player who joined afterwards played on a throwaway save.
- Exhausting `LOAD_ATTEMPTS` no longer latches the server either: that is also what a genuinely contended profile looks like, which is the case the retries exist for.
- A live server that cannot reach the store **refuses the load** (the caller kicks with a rejoin message) rather than handing out a fresh profile that would overwrite real progress. Studio proceeds on memory.
- Saves that could not be written are queued in `pendingSaves` and retried by `DataService.flushPending()`, started from `DataService.start()`. `memory` is always shadowed so a recovered store has something to flush.
- Do not enable Studio access to live production data for routine tests.

## Economy rules and invariants

### Starting state and income

- Starting weight: `60 kg`.
- Starting Calories: `300`.
- Base tray slots: `6`; **hard ceiling `Balance.MaxSlots` = 18** on earned slots.
- Starting tray: `{ 1, 1, 1, 2 }`, intentionally containing one full merge group.
- Base recovery income: `15 Cal/s`.
- Recovery floor: `max(15, peak tray Cal/s this run * 0.10)` applied to the tray component only.
- Total income: `max(current tray rate, recovery floor) + digestion rate`.

Multiplier:

```text
milestone metabolism multiplier
* 1.5 ^ diets
* 1.10 ^ metabolism-upgrade level      -- COMPOUNDING, see Upgrades
* incomeMultiplier
```

The same multiplier scales tray Cal/s, digestion Cal/s, and kilograms gained per eat.

A second, **weight-only** multiplier is applied on top when eating:

```text
Economy.bigBiteMult = 1.08 ^ bigbites-upgrade level
```

It deliberately does not scale Cal/s or digestion. Because it is flat across
tiers it cannot move `KgRatio` against `MergeCost`, so the Eat-vs-Merge invariant
is unaffected at any level.

### Price scaling

Bag and upgrade prices are quoted in Calories while income is MULTIPLIED, so without a correction every completed diet made the whole ladder affordable within seconds and the part of a run spent climbing it disappeared. That is invisible in the rebirth arithmetic -- the target/metabolism relationship below is correct on its own, and a measured campaign still produced flat 5-7 minute runs at every diet.

```text
Economy.priceScale(state) = Balance.MetabolismPerDiet ^ (diets * Balance.PriceScaleExponent)
                          = 1.5 ^ (diets * 0.85)
```

Applied to `Economy.bagCost` and `Economy.upgradeCost`, and therefore to `Economy.buyBag`.

- Keyed on **`diets` only**, never on the full multiplier. Milestone and metabolism-upgrade gains are earned mid-run, and a shop whose prices rose every time the player crossed a milestone would punish the progress it was rewarding. Diets change only at a rebirth, where a step change in prices is legible.
- The exponent is below `1` on purpose: income scales `M`, prices scale `M^0.85`, so purchases per second still scale `M^0.15`. A diet buys a real (if modest) purchasing gain on top of its weight-per-meal gain, rather than reading as an inflation tax.
- Measured effect on the per-diet run length ratio (engaged campaign, seed 12345): `e=0` gives `0.85-1.03x` (runs getting SHORTER), `e=0.85` gives `1.11-1.23x`, `e=1.0` gives `1.14-1.32x`.

### Food tiers

Food progression is one positive **global tier** with a thirty-food menu that repeats through unlimited Menu Ascensions:

```text
menuIndex(g) = ((g - 1) % 30) + 1
ascension(g) = (g - 1) // 30
```

The registry holds thirty base foods exactly once. Tier 30 is The Whole Menu, tier 31 is Small Fries `★1`, and every later cycle repeats those base identities at a higher power. `Food.AbsoluteMaxTier = 1050` is a **floating-point** ceiling, not a gameplay one: every value here is an exponential in the global tier, and the steepest (`1.80^g`, the merge price) overflows a double at roughly tier 1,207. An infinite merge price is worse than a crash -- merging silently becomes unaffordable forever, and an infinite Cal/s would put NaN into weight and then into the save. 1050 keeps all three formulas finite (asserted at require time in `Config.Food`) and sits some six hundred diets past the shipped pacing. The highest ascension a player can complete is therefore `★34`. Getting past it needs a mantissa/exponent number type through the whole economy, which is separate work.

Generated values:

- tier Cal/s: `1.57 ^ (tier - 1)`;
- tier eat weight: `4.8 * 1.40 ^ (tier - 1)` kg before multipliers;
- merge Calorie cost: `15 * 1.80 ^ (tier - 1)`, charged per merge;
- merge shape: three matching items become two of the next global tier; a pair becomes one;
- `Food.AscensionPower = 1.57 ^ 30` scales acquisition and Kitchen costs across each full cycle so payback drift resets per menu rather than collapsing forever.

Key invariants are now checked over one thirty-food cycle: `KgRatio (1.40) < MergeCost (1.5)`, `CalRatio (1.57) > MergeCost (1.5)`, `MergeCalRatio (1.80) > CalRatio`, and `(MergeCost / CalRatio) ^ 29 >= 0.25` (current drift about `0.267`).

**Why low-tier food being the cheapest weight per Calorie is no longer the optimal strategy.** Because `KgRatio < MergeCost`, weight per Calorie falls `0.933^(n-1)` up the ladder -- tier 1 is and remains the cheapest weight in the game per Calorie, by design. That used to also make it the FASTEST weight in the game, so the correct play (for a player or a bot) was buy the cheapest bag, never merge, spam eat, and merging was a detour.

`Balance.MinMealSeconds` changes what is scarce. Weight is now **throughput-limited rather than Calorie-limited**, so the currency that matters is kilograms per MEAL -- which is `KgRatio^(tier-1)`, i.e. `17,300x` across one thirty-food cycle for the same gesture, and it multiplies by that again every ascension. Calories are cheap late; gestures are not. That is the real gameplay advantage of climbing the ladder, and it is why the ladder can afford to keep low tiers Calorie-efficient.

### Kitchen Levels

`Config.Kitchen` owns the cap on which tiers can be **produced**, and it is the answer to "a big tray plus Auto-Merge climbs the ladder on its own". The split it enforces:

- **Kitchen Level** decides the highest tier that can exist for that player;
- **tray slots** decide throughput and convenience, and nothing else.

The Kitchen ladder is unbounded in gameplay. Every level opens two more global tiers:

```text
maxTier(level) = min(2 + 2 * level, Food.AbsoluteMaxTier)
upgradeCost(level) = floor(2500 * (Food.CalRatio ^ 2 * 1.45) ^ (level - 1))
```

The fifteen authored names repeat with `Mk II`, `Mk III`, and later suffixes. The `1.45` wall margin is the measured throttle that prevents the infinite Kitchen from adding enough weight-per-meal every diet to make later runs shrink.

**Every production path is clamped**, and "every" is the point:

- `Economy.merge` refuses when the OUTPUT tier exceeds `Economy.kitchenMaxTier`;
- `Economy.autoMergeTier` clamps the Auto-Merge INPUT to one below the ceiling, so its last automatic merge cannot produce a locked tier;
- `Economy.mergeAll` uses the same input cap;
- **bag rolls are clamped too** (`rollForState`). A roll above the ceiling lands ON the ceiling, and a jackpot that was clamped away is not announced as a jackpot -- a celebration for something taken back reads worse than no celebration.

Kitchen level **resets to 1 on New Diet**, which is what makes it a repeatable wall rather than a one-time unlock.

### Merge costs

Merging charges Calories per merge: `Food.mergeCalCost(tier) * priceScale * PrepLine discount`, read through `Economy.mergeCost`. Item count alone could not gate the ladder -- slots, Auto-Merge and Fill Tray all make items cheaper to produce -- so the Kitchen fixes the ceiling and this fixes the slope beneath it.

Priced against **acquisition** cost rather than income, so a merge stays a constant share (~a fifth of one input item) at every tier: T1 15, T7 510, T12 9,640, T19 590,196.

- `MERGE_CAL_RATIO (1.80)` must stay above `CAL_RATIO (1.57)`, asserted by `Balance.validate()`. If merging got cheaper against the income it produces, every tier would pay for its own merge faster than the last -- the payback-drift collapse arriving through the other door.
- The top merge must stay cheaper than the top bag, or buying a tier would always beat making it. Also asserted.
- **Auto-Merge pays the same price.** It is a convenience, not a free lane; with no Calories it merges nothing.
- `findMergeGroup` returns the LOWEST-tier full group, which is now load-bearing rather than cosmetic: costs rise with tier, so if the cheapest group is unaffordable every other one is too, and a bulk merger can stop immediately and be right to stop.

### Eating and digestion

- Eating removes one owned tier from the tray and grants its scaled weight.
- `50%` of the food's raw Cal/s is moved into current-run digestion.
- Digestion is stored unmultiplied so later milestone, diet, or upgrade multipliers scale it consistently.
- `EatCalShare` must remain below `1`; at `1` eating would become income-neutral and destroy the merge decision.
- Physical eating is client-presented as plating followed by repeated bite clicks, but the server sees only the final `Eat(tier)` intent.
- Bite count falls with weight and is clamped from `5` down to `1`; the formula uses `EatBitesPerDecade = 1.8` from the `60 kg` baseline.
- **Meal duration is floored at `Balance.MinMealSeconds` (0.35s)**, and that floor is an economy rule rather than an animation one. A late-game player is at 1 bite and up to 12 bites/sec, so a meal used to resolve in 83ms and the server would accept ~10 meals a second. That made a remote-only eat bot trivial AND made tier 1 food optimal (see the food invariants above). 0.35s is above the fastest a human can drag-and-hold and well below the 1.25s an opening-weight meal already takes, so it binds only where the old duration had collapsed toward zero.
- Bite *rate* is `Economy.chewSpeed(state)` = `Balance.BitesPerSecond` scaled by the Fast Chew upgrade, so meal duration is `eatBites(kg) / chewSpeed(state)`. `Client/TableFood` derives its animation interval from the payload's `duration / bites` rather than a constant; a hardcoded 4/s made the client the bottleneck and Fast Chew visibly do nothing.
- A disconnect during a partially eaten client-side plate loses nothing because the server item remains in the tray until the final bite.
- `BeginEat` requires the character's `HumanoidRootPart` to be within `Balance.EatRadiusStuds` (110) of the player's own plot, measured **horizontally** (a stage-10+ player is forty studs tall and anchored well above the floor, so including Y would refuse to let the biggest players eat). Checked only at `BeginEat`; a meal already begun is not cancelled for walking away.
- The check FAILS OPEN on "no resolver wired" and "no plot assigned yet" (startup states the player does not control, and what the headless tests run under) and FAILS CLOSED on "no character" and on distance -- which is exactly what a remote-only client looks like.

### Bags

| ID | Name | Cost | Unlock | Normal tier distribution | Payback |
|---|---|---:|---:|---|---:|
| `snack` | Snack Bag | 100 | 0 kg | T1 70%, T2 25%, T3 5% | 81s |
| `big` | Big Bag | 310 | 120 kg | T2 50%, T3 32%, T4 14%, T5 4% | 126s |
| `mega` | Mega Bag | 1,620 | 350 kg | T4 32%, T5 30%, T6 22%, T7 16% | 196s |
| `feast` | Feast Bag | 6,770 | 1,200 kg | T6 30%, T7 28%, T8 22%, T9 20% | 303s |
| `banquet` | Banquet Bag | 27,300 | 4,000 kg | T8 28%, T9 28%, T10 24%, T11 20% | 470s |
| `monstrosity` | Monstrosity Bag | 110,000 | 9,000 kg | T10 26%, T11 28%, T12 26%, T13 20% | 730s |
| `pallet` | Pallet Bag | 436,000 | 30,000 kg | T12 26%, T13 28%, T14 26%, T15 20% | 1,130s |
| `truckload` | Truckload Bag | 1,730,000 | 110,000 kg | T14 26%, T15 28%, T16 26%, T17 20% | 1,752s |
| `franchise` | Whole Franchise Bag | 8,170,000 | 400,000 kg | T16 22%, T17 26%, T18 26%, T19 16%, T20 10% | 2,713s |

All bags carry a `0.1%` jackpot chance of one tier above the table's max. Prices shown are the diet-0 prices; the live price is `cost * Economy.priceScale(state)`.

**Costs are DERIVED, not typed.** Only the Snack Bag's `ANCHOR_COST` (100) is hand-set; every other rung is solved from a single constant:

```text
cost(k) = expectedCalPerSec(k) / (valuePerCalorie(1) / VALUE_FALLOFF ^ (k - 1))
```

with `VALUE_FALLOFF = 1.55`. Every rung is priced at exactly 1.55x worse Calorie-value than the one below it. Derived costs are rounded to three significant figures, a <0.5% error against rungs that differ by 55%, so rounding cannot flip either ordering invariant. A hand-typed cost is how the old 97x ladder happened.

The per-rung step is GENTLER than the old ladder's 1.75-1.95x, but there are eight steps instead of five, so the ladder ends 33.3x down instead of 21.6x -- more expensive end to end while each individual rung stays worth the upgrade. Skipping a rung means paying 2.4x worse Calorie-value for the tier jump instead of 1.55x, which is what stops players comfortably skipping several tiers.

Bag design invariant:

- cheaper bags have better expected Cal/s per Calorie spent;
- pricier bags have better expected Cal/s per tray slot/click;
- the **total** falloff from cheapest to priciest stays at or below `Balance.MaxBagValueFalloff`, raised `30 -> 45` for the longer ladder. Do not raise it again to accommodate a steeper PER-RUNG falloff -- that is the failure it catches.

Bag unlocks use `peakKg`, so they survive a New Diet.

### Run Upgrades

**Upgrade levels are per-run: a New Diet clears them all.** They used to be permanent, which is what made later runs hollow -- the panel was a checklist completed once, after which every diet was the same run at a larger multiplier with nothing left to buy. Permanent power now lives entirely in Perks. Auto-Merge is the one capability that does not come back dead, because the Auto-Chef perk sets the tier a fresh run's Auto-Merge starts at.

Run upgrades support +1/+10/+100/MAX purchase amounts. **Bigger Bites and Metabolism are infinite** and keep compounding past level 100; tray size, luck, Auto-Merge, bite speed, and chance effects remain finite where a hard cap has gameplay meaning.

Cost at the next level is
`floor(baseCost * costMultiplier ^ currentLevel * Economy.priceScale(state) * SecondWind discount)`.

Each definition carries a `perLevel` magnitude and a `mode`. **Percentage upgrades COMPOUND** (`mode = "mult"`): the effect is `(1 + perLevel) ^ level`, read through `Upgrades.factorAt`. Counts and probabilities stay additive, read through `Upgrades.amountAt`. `amountAt` returns `0` for a multiplicative upgrade rather than a plausible-looking wrong number.

The reason is the thing the rebalance was asked to avoid -- "price x10 while the benefit barely changes". At `+12%` per level added into `(1 + 0.12 * L)`, level 17 to 18 is a 2.7x price step for a 4% gain, so the top half of every percentage ladder was dead content wearing a price tag. Compounding makes every level worth the same headline percentage, which is what gives the exponential price curve something real to climb against.

| ID | Effect | Per level | Mode | Base cost | Mult | Max | At max |
|---|---|---|---|---:|---:|---:|---:|
| `automerge` | Auto-merge full groups up through the purchased tier. | +1 tier | add | 500 | 3.6 | 8 | T8 of 20 |
| `bigbites` | **Weight only** multiplier on every eat. | +8% | mult | 300 | 2.2 | 20 | 4.66x |
| `fastchew` | Scales `Balance.BitesPerSecond`. | +25% speed | add | 800 | 3.0 | 8 | 3.00x |
| `luckybite` | Chance a meal pays `LuckyBiteMult` (3x) weight. | +4% chance | add | 2,000 | 2.8 | 12 | 48% |
| `slots` | +1 tray slot per level. | +1 slot | add | 3,000 | 4.5 | 6 | +6 |
| `doublemeal` | Chance one meal consumes a second item **of the same tier** free. | +3% chance | add | 6,000 | 2.9 | 12 | 36% |
| `metabolism` | +Calories **and** weight (the shared multiplier). | +10% | mult | 8,000 | 2.4 | 18 | 5.56x |
| `fridge` | +offline cap hours per level. | +2 h | add | 4,000 | 3.0 | 8 | +16h |

Two entries carry most of the anti-"skip progression" work:

- **`slots` is the steepest curve in the file and the shortest ladder** (6 levels at x4.5, ~7.1M total). A tray slot is income, merge material and Fill Tray throughput at once, so it is the most compounding purchase in the game -- and at the old 1,200 x2.8 for ten levels it was also among the cheapest.
- **`automerge` caps at tier 8 of 20**, not near the top. At tier 9 of 12 it had become the whole merge game: a big tray plus a maxed Auto-Merge turned Fill Tray into an automatic high-tier-food machine. Everything above tier 8 is now a decision made by hand, which is where the Eat-vs-Merge choice lives.

`EconomyTests` asserts every upgrade's final level costs at most `1e11` (raised from `1e9` with the steeper curves; weight targets now run to eight figures across a dozen diets and the top of the metabolism ladder is deliberately a multi-run goal).

`luckybite` and `doublemeal` roll against the **server session's** `Random` stream, threaded through `Meal.complete` into `Economy.eatTier`. Omitting the stream disables both rolls, which is what keeps the headless tests deterministic. Neither effect feeds `digestion`: paying weight bonuses into permanent income would make eating out-earn holding and collapse the Eat-vs-Merge decision.

### Milestones and character stages

Milestone rewards derive only from permanent `peakKg`. Visual size stage derives from current `kg`, so rebirth shrinks the character.

| kg | Reward/label |
|---:|---|
| 75 | +1 slot — Shirt's getting tight |
| 120 | +10% metabolism and Big Bag unlock |
| 175 | +5% metabolism — Trousers have given up |
| 350 | +1 slot and Mega Bag unlock |
| 550 | +10% metabolism, 10-hour offline cap |
| 750 | +1 slot — Two seats, one you |
| 1,200 | +25% metabolism and Feast Bag unlock |
| 1,600 | +10% metabolism |
| 2,200 | +1 slot and +15% metabolism |
| 3,000 | +15% metabolism |
| 4,000 | +20% metabolism, 12-hour offline cap, Banquet Bag unlock |
| 5,000 | +1 slot and +20% metabolism |
| 6,500 | +25% metabolism |
| 8,000 | +25% metabolism — Half the restaurant |
| 9,000 | +1 slot, +30% metabolism, Monstrosity Bag unlock |
| 10,000 | First rebirth threshold — You've outgrown the restaurant |

**Milestone slots dropped from ten to six.** Six free + six purchasable + six base is `Balance.MaxSlots` (18) exactly, and `validate()` asserts that identity: more would be silently wasted at the cap, less would leave the ceiling unreachable and the cap decorative.

**Milestones past 10,000 kg are GENERATED**, because the ladder now has to reach eight figures and hand-authoring it would be a thousand near-identical lines that drift out of step with the rebirth curve. Each rung is `1.25x` the last, which puts 5-6 milestones inside every run regardless of which diet it is, up to `1e13` kg -- 108 entries in total. Each grants `+6% metabolism` and nothing else (the slot ceiling is already allocated). Because the grant is additive into a sum, the tail's contribution grows with the LOGARITHM of weight while the rebirth requirement grows geometrically; that relationship is the point, and it is why the ladder can run forever without catching up to the thing it paces.

The three post-rebirth bag unlocks are inserted at their EXACT unlock weights (30,000 / 110,000 / 400,000) and replace the generated rung that triggered them, so two milestones never fire a few hundred kilograms apart.

`Progression.rewardsFor` and `nextMilestone` use **prefix sums plus a binary search**, not a scan. `rewardsFor` sits inside `Economy.multiplier`, which the income accrual calls several times per player per frame; a linear scan over 108 entries would be tens of thousands of iterations per frame on a full server. `rewardsFor` returns a SHARED cumulative table -- callers must treat it as read-only.

The metabolism ladder is load-bearing. It previously granted 1.85x total from three entries, two of which land in the first tenth of the run, leaving nothing growing to carry the back half — while the 8,000 → 10,000 band is the largest in the ladder. It now reaches **3.10x**, and the three previously flavour-only milestones (2,700 / 7,000 and the 550 entry) carry real rewards. A first draft reaching 4.0x was measured as too strong: the final band collapsed to 24 seconds.

Size thresholds are `60, 75, 120, 175, 350, 550, 750, 1200, 1600, 2200, 3000, 4000, 5000, 6500, 8000, 9000, 10000` kg -- one per CORE milestone, so every first-run milestone produces a body change.

The visual ladder deliberately STOPS at the first rebirth target even though weight now runs to eight figures: the top stage is already larger than the room, and `Progression.sizeStage` clamps, so a 400,000 kg player renders as "has outgrown the restaurant" rather than as a rendering failure.

`CharacterService` uses matching scale arrays:

- overall scale: `1.0, 1.12, 1.28, 1.45, 1.7, 2.0, 2.2, 2.5, 3.0, 3.7, 4.7, 6.0, 8.0`;
- girth multiplier: `1.0, 1.12, 1.25, 1.38, 1.5, 1.62, 1.72, 1.82, 1.92, 2.02, 2.14, 2.24, 2.35`.

At stage `10` (`3,500 kg`) and above, movement/jumping is disabled and the root is anchored because the character is too large for the room. Earlier stages restore normal movement. R15 humanoid scale values produce extra width/depth; R6 falls back to uniform `ScaleTo`.

### Offline and daily rewards

- Offline earnings grant Calories only, never weight.
- Player-owned tray/digestion income uses the existing `75%` offline efficiency.
- Worker income is split out and earns at exactly `50%`; workers do not simulate tray purchases while away.
- Base cap: `8` hours; milestone caps replace it with `10` then `12` hours; fridge levels add hours.
- Under 60 seconds away grants nothing.
- Longest matching return bonus only:
  - 1+ hour: 3 Snack Bags;
  - 4+ hours: 1 Big Bag;
  - 8+ hours: 1 Mega Bag.
- Bonus and daily bags are stored as pending counts and rolled only when opened.

Daily seven-step reward ladder:

1. 3 Snack Bags
2. 2 Big Bags
3. 4 Big Bags
4. 2 Mega Bags
5. 4 Mega Bags
6. 2 Feast Bags
7. 4 Feast Bags

Consecutive days advance the streak, a missed day resets it, and the ladder wraps after day seven.

### Restaurant workers

Ten unique workers are ordered as Busser, Cashier, Prep Cook, Fry Cook, Server, Nutritionist, Sous Chef, Shift Manager, Trainer, and Executive Chef. Their fixed caps are T4, T8, T12, T16, T20, T24, T28, T32, T36, and T40. A worker cannot be hired before the Kitchen can reach its required tier, and its effective cap is `min(identity cap, kitchenMaxTier)`.

Each hired worker has a private six-slot tray. One bounded cycle attempts, in order: buy the highest affordable unlocked bag that can produce within the effective cap; merge only full groups; eat the highest tier if no purchase/merge can proceed. All purchases and merges draw from shared Calories and must leave the configured reserve. `Pause All` and individual toggles halt automation without deleting state.

Every worker level multiplies personal cycle/production forever. Owner buffs unlock at worker levels 10/25/50/100:

1. Busser: player eating speed.
2. Cashier: bag discount.
3. Prep Cook: merge discount.
4. Fry Cook: player weight per meal.
5. Server: player Cal/s.
6. Nutritionist: digestion retained.
7. Sous Chef: Kitchen discount.
8. Shift Manager: Run Upgrade discount.
9. Trainer: all worker speed.
10. Executive Chef: all worker output.

Worker kilograms never add to player kilograms. They increase the worker's per-run digestion and visible size; rendering caps at about 1.8x. `WorkerService` creates replicated seated block characters only for purchased workers, shows one active plate despite the six-slot underlying tray, and animates three eating patterns plus idle fidgets. Unclaimed restaurants and unpurchased stations contain no character models.

### New Diet / rebirth

Requirement: current `kg >= Progression.rebirthKgFor(diets)`, which is `10,000 * 3.5 ^ diets` rounded to two significant figures:

The snapshot also exposes an unbounded player-facing `difficultyBand = diets + 1`. `Progression.difficultyFactor` divides throughput: quadratic and front-loaded through diet 8, then still increasing but asymptotically approaching 40 so staffed campaigns stay under the roughly four-hour target.

| diet | target | diet | target |
|---:|---:|---:|---:|
| 0 | 10,000 | 4 | 1,500,000 |
| 1 | 35,000 | 5 | 5,300,000 |
| 2 | 120,000 | 6 | 18,000,000 |
| 3 | 430,000 | 7 | 64,000,000 |

The rounded number IS the threshold, not a rounded display of an exact one -- a player who reads "120,000 kg" must not be refused at 120,000 kg. The curve is unbounded; there is no last diet.

**Why the target has to grow FASTER than metabolism.** Metabolism scales Cal/s AND kg-per-eat, and Calories buy the food that becomes kilograms, so weight per second scales with the **square** of the multiplier. At the old flat 10,000 kg target and `1.75 ^ diets`, every run was `3.06x` faster than the previous one against a requirement that never moved -- the game solved itself around diet 3. Two changes, both needed:

- `Balance.MetabolismPerDiet` lowered `1.75 -> 1.5`, so the speedup is `2.25x` per diet;
- `Progression.RebirthGrowth = 3.5`, deliberately ahead of it.

The target-growth invariant is still asserted, but actual run time is now shaped by the separate difficulty divisor and by workers. Early runs ramp sharply; late runs can fluctuate when permanent perks and Kitchen breakpoints land, while the effective difficulty factor remains monotonic and staffed simulation stays beneath the intended ceiling.

Resets:

- current `kg` to `60`;
- Calories to the Head Start perk's grant (`0` without it);
- tray to `{ 1, 1, 1, 2 }`;
- current-run peak Cal/s to `0`;
- digestion to `0`;
- every worker hire, level, private tray, kg, digestion, cycle, last tier, and meal count;
- **all Run Upgrade levels** (cleared, not decremented -- a half-kept ladder makes the cost curve unreadable and the pacing unmeasurable);
- **Kitchen level to 1.**

Preserves:

- `peakKg`, `lifetimeKg`, and completed diet count;
- milestone rewards and bag unlocks derived from peak weight -- this is what keeps old content trivially easy instead of hard again, which is the explicit goal of the loop;
- **Diet Points and Perks**;
- food discoveries, monetization-hook fields, the Auto-Merge toggle preference;
- worker reserve, global pause, and ten per-worker enable preferences;
- pending bags and daily metadata.

Each completed diet multiplies metabolism by `1.5`, raises the next target by `3.5x`, and raises all Calorie prices by `1.5 ^ 0.85`. Diets 1-3 additionally unlock the Pallet / Truckload / Whole Franchise bags and with them food tiers 12-20.

### Diet Points and Permanent Perks

`Progression.dietPointsFor(kg, diets)` is the payout, multiplied by the Diet Guru perk in `Economy.dietPointsPreview`:

```text
points = floor(8 * ratio^0.6 * (1 + 0.35 * diets))
ratio  = clamp(kg / target, 1, 8)
```

Three brakes, each load-bearing:

- **the exponent is below 1** -- weight is exponential in time late in a run, so a linear payout would make "push twice as far" worth twice as much for a fraction of the extra time. At 0.6, doubling pays 1.52x: a real reward that never beats taking the next diet;
- **the ratio is capped at 8x** -- past that the payout stops growing, or "never rebirth" becomes optimal and the loop inverts;
- **the per-diet term is additive, not geometric** -- points keep up with compounding perk prices without a late diet drowning the whole ladder in one run.

Reference payouts at the minimum push: diet 0 -> 8, diet 4 -> 19, diet 9 -> 33. Below the target the payout is `0`, not a fraction.

`Config.Perks` ships **16 perks**, 4-10 levels each, ~1,741 Diet Points to max everything -- a lifetime ladder against an income of 8-30 points per diet. They are the only lasting purchase in the game.

| Perk | Effect | Mode | Max |
|---|---|---|---:|
| `stomach` Bottomless Stomach | +5% weight per eat | mult | 10 |
| `appetite` Golden Appetite | +5% Cal/s (income only) | mult | 10 |
| `portions` Bigger Portions | +4% Calories AND weight | mult | 8 |
| `masterchef` Master Chef | Kitchen upgrades -7% | mult | 8 |
| `prepline` Prep Line | merge cost -8% | mult | 8 |
| `autochef` Auto-Chef | fresh runs start with Auto-Merge at tier N | add | 6 |
| `headstart` Head Start | +1 tray slot and +2,500 starting Calories per level | add | 5 |
| `heavyeater` Heavy Eater | +5% Calories and weight past half the diet target | mult | 8 |
| `luckyeater` Lucky Eater | +2% Lucky Bite chance | add | 8 |
| `ironstomach` Iron Stomach | +2% Double Meal chance | add | 8 |
| `fasthands` Fast Hands | +10% chew speed | add | 6 |
| `mealprep` Meal Prep | Fill Tray buys +8 items per press | add | 4 |
| `franchise` Franchise Owner | bags -5% | mult | 8 |
| `secondwind` Second Wind | Run Upgrades -5% | mult | 8 |
| `freezer` Deep Freezer | +2h offline cap | add | 6 |
| `dietguru` Diet Guru | +8% Diet Points | mult | 8 |

**Invariant safety.** Every perk is either a FLAT multiplier (applies equally to a tier-n item and to the tier-(n+1) it merges into) or a cost/chance change, so none of them can move `KgRatio` against `MergeCost`. Heavy Eater looks like the exception and is not: it keys off the PLAYER's weight, never off food tier, and scales Calories and kilograms together exactly as metabolism does. `EconomyTests` pins Eat-vs-Merge at the top of both weight ladders.

Head Start slots are added OUTSIDE `Balance.MaxSlots`, like `bonusSlots`: the cap exists to stop EARNED slots from becoming a way to skip progression, and a perk bought with permanent currency must not silently do nothing for the players furthest along.

## World and presentation

### Place-level settings observed

- `Workspace.StreamingEnabled = true`.
- Gravity approximately `196.2`.
- Fallen-parts destroy height: `-500`.
- Camera mode: Classic.
- Character appearance loading enabled.
- Mouse lock option enabled.
- Lighting: ClockTime `14`, Brightness `1`, warm ambient/outdoor ambient.
- Lighting effects: `Sky`, `SunRays`, `Atmosphere`, `Bloom`, `DepthOfField`, and `Grade`.
- `Grade` is a `ColorCorrectionEffect` written by `MapBuilder.applyGrade()` (saturation `-0.09`, contrast `-0.02`, brightness `0`, faintly warm tint). It is the one global knob for scene punch: every tier palette stays authored at full strength so the act changes still read, and this takes a uniform slice off saturation at the end of the pipeline. `applyGrade()` is idempotent and runs from `ensureTodoWorld()` on every boot, not only on a destructive rebuild, so a place saved before the effect existed still gets one. `Bloom` is `0.30` intensity at a `1.95` threshold -- it was `0.4`/`1.8`, which lifted the whole cream-walled interior and was most of why the restaurant read as overexposed.
- Sound filtering is respected; ambient reverb is NoReverb.

### District, lots, and restaurants

`ServerStorage.MapBuilder` owns the baked district. `MapBuilder.build()` is destructive and deliberate (Edit mode only): it wipes `Workspace.Restaurants`, any legacy v1 roots, the default base and Terrain, then bakes the district plus `ServerStorage.RestaurantTemplate`. `ensureTodoWorld()` is the idempotent, non-destructive boot path.

The district is a vehicle-ready fast-food suburb (vehicles themselves are out of scope): twenty permanent lots in four rows of five, three parallel roads between the rows joined at both ends by connector roads into one continuous ladder with no dead end, and a landscaped green buffer ringed by a low hedge with an invisible collision backstop and deliberate expansion space beyond. The middle road is the boulevard (two carriageways around a planted median with hedges, streetlights and a mid-block crossing). Roads carry 32 studs of asphalt with 8-stud sidewalks, dashed lane markings, crosswalks and stepped curb ramps at every junction; sidewalks are continuous around the whole network. Lots are 140x150 studs: one flat grey-asphalt surface with a darker parking apron, one driveway with curb ramps, a frontage path aligned to the restaurant's road-facing entrance, and one freestanding double-sided roadside `OwnerSign`. Every lot carries `PlotIndex` (row-major and stable), an authoritative `PlotCFrame` CFrame attribute (its restaurant's local-to-world transform), and a compatible `PlotOrigin` (building centre). Rows 0-1 face +Z; rows 2-3 are rotated 180 degrees, so the two middle rows face each other across the boulevard and each outer row faces its outer road.

Restaurants are permanently baked into every plot. `ServerStorage.RestaurantTemplate` remains the authoritative local-coordinate source (pivot: invisible `PivotAnchor` at ground level), and `MapBuilder.buildPlot` clones it into all twenty lots. `PlotService` reuses the existing `plot.Restaurant`, resolves spawn/board/dial references, paints the owner's tier, and sets the sign. On leave it clears owner presentation and returns the sign to AVAILABLE without destroying the building. Unclaimed restaurants contain all furniture, counters, stations, doors, and decor but no worker characters.

Assignment order is deterministic and documented in `PlotService`: boulevard distance first (rows 1 and 2, then rows 0 and 3), centre-out columns (2, 1, 3, 0, 4), row ascending as the tie-break. The first player lands on **Plot08** (row 1, column 2). The sign shows the ACCOUNT username (`Player.Name`, never DisplayName, casing preserved, truncated to ten characters) as `<name>'s Restaurant`, written authoritatively on the server.

The room constant is `120` studs (one restaurant's interior) and is mirrored in `Shared.Config.Stations`; changes must keep both places aligned.

Each clone contains, exactly as the current restaurant design:

- `Floor`, `Walls`, `Counter`, `Lights`;
- a 28x24-stud road-facing double sliding-glass entrance inside `Walls`: split wall/header pieces, tier-painted trim, two collidable leaves and handles, an invisible occupancy sensor, and an E `ProximityPrompt` reachable from either side. `DoorService` slides both leaves outward and disables collision, then closes two seconds after the doorway is empty; the overhead marquee's exterior face says `COME ON IN`;
- `Booth`: benches, table, table tray, invisible `DropZone`, and `SeatAnchor`;
- `BoothSpawn`: a `SpawnLocation` positioned clear of the booth and facing it;
- `Stations`: object-only `Till`/`TillScreen`, `Fryer`/`FryerTop`, `ScaleBase`/`ScalePost`/`ScaleDial`, and `StaffBoard`, plus `RecordBoard` and `SizeLeaderboard`;
- `WorkerStations`: ten permanent table/chair/`SeatAnchor`/`PlateAnchor` arrangements. Purchased replicated characters live separately under runtime `WorkerCharacters`.

`ServerScriptService.Server.Services.PlotService` assigns one plot per player on join and stamps the `PlotIndex` attribute on the `Player`, which replicates; `Client/PlotRoot` resolves the local player's own optional `Restaurant` model from it (nil while unassigned, unreplicated or streamed out -- callers tolerate all three). Assignment is separate from profile loading so a player lands in their restaurant even while a slow DataStore read is retrying.

Plot ownership rules:

- `assign()` never yields between finding a free plot and claiming it, so concurrent joins cannot take the same plot.
- `scan()` is idempotent and carries existing ownership across by index, so a rescan cannot evict a player already standing in their restaurant.
- If `Workspace.Restaurants` is not present at boot, joins are held in a pending queue and the scan is retried once a second for thirty seconds, then drained. Scanning once and giving up is what used to leave every player without a restaurant.
- A player who cannot be given a plot is **kicked with a rejoin message**. The old behaviour sat a 21st player in plot 1 alongside its owner, which hid a misconfiguration behind a broken experience for two players -- and got worse once eating became location-aware, since both then resolve to the same restaurant.
- `Players.MaxPlayers` is **read-only at runtime** (verified: assigning it throws). Capacity must be set to the plot count in Game Settings by hand; `PlotService.warnIfOverCapacity()` logs the mismatch and names the fix.
- Restaurant occupancy and appearance are server-session state only; no plot identity or restaurant state is persisted.

The physical booth tray mirrors owned food client-side. `DropZone` is the raycast target that makes drag-to-eat usable even when the thin tray is hard to see. A successful `BeginEat` seats and anchors the owner at `Booth.SeatAnchor`; completion/cancel/failure restores movement. `MealCharacterAnimation` runs one of three procedural upper-body eating loops, matching the three worker presentation loops. `TableFood.trackedPosition` collapses inverted clamp ranges to viewport centre, so narrow/short windows cannot throw while tracking the bite button.

### Stations

`ServerStorage.StationsBuilder` bakes object stations into `ServerStorage.RestaurantTemplate` in local coordinates (`buildInto`); all permanent restaurant clones carry them. The old always-visible Cashier/Fry Cook/Nutritionist NPC models are gone. Prompts live on recognizable objects and carry `StationId`. The owner's `RecordBoard` is an 8-stud-high marquee above the entrance.

| Station ID | Presentation | Opens |
|---|---|---|
| `cashier` | Counter till | Food group |
| `frycook` | Fryer | Food group |
| `nutritionist` | Left-wall scale | Progress group |
| `staff` | Staff roster board | Staff group |

There is deliberately **no drive-thru/daily station**: the left rail's DAILY button is always reachable, including for an anchored stage-10+ player, so the daily flow lost nothing when it was removed.

Each occupied plot's `RecordBoard` shows its OWNER's `lifetimeKg` and display name, and `ScaleDial.ScaleDisplay` mirrors their current weight; `PlotService` paints both for everyone on a two-second tick rather than per-sync, so a visitor sees whose restaurant they are in. `SizeLeaderboard` shows a server-local, sanitized, deterministic top five of loaded players, written to every occupied plot's wall and coalesced to at most once per two seconds; `LeaderboardService` re-derives the board list on every refresh, so a newly assigned restaurant joins the rotation within one tick.

The left navigation rail remains available even though world stations exist, because very large/anchored players cannot reliably walk to them.

#### Restaurant tiers

Crossing a CORE milestone **retextures the player's restaurant** -- floor, walls, trim, counter, booth, ceiling, light panels and the menu-board sign. `ReplicatedStorage.Shared.Config.RestaurantTiers` holds seventeen palettes and `ServerScriptService.Server.Services.RestaurantSkin` applies one to the plot's `Restaurant` clone (the permanent lot, roads and signs are never repainted).

**Act I is tuned for a long look, not for a screenshot.** Tiers 1-2 ran a near-white cream against fully saturated pillarbox red on a 20-stud checker, with the booths in the same red under a lit ceiling and the HUD's own warm accents on top -- an enormous high-frequency value contrast in the room players sit in for hours. The red moved toward brick and the cream a shade off white; the later acts already run dark or cold and are untouched, and the neon tiers are supposed to be loud. `MapBuilder.PALETTE` mirrors tier 1 and must stay in step with it -- `RestaurantSkin` repaints every clone on assignment, so `PALETTE` only decides what a freshly baked template looks like before its first repaint, but a template baked in the old red would flash the old room for a frame on every assignment.

The tier is derived from **`peakKg`, not current `kg`**, and that asymmetry with body size is deliberate. `Progression.sizeStage` keys on current weight because the character really does shrink on a New Diet -- that is the joke. The BUILDING must not: a player who earned the gold franchise at 9,000 kg and then rebirths to 60 kg would otherwise watch their restaurant get demolished as the reward for progressing. `peakKg` is the same permanent watermark that already derives tray slots and bag unlocks, so the restaurant survives rebirth by the mechanism everything else permanent uses.

Thresholds are **`Progression.SizeStages` reused, not copied**. That array is already 1:1 with the sixteen CORE milestones, and a parallel ladder here would be a second copy of the same numbers that drifts the first time either is retuned. `RestaurantTiers.List` asserts equal length with it on load, the same bargain `CharacterService.SCALE` makes with the same array. Index 1 is the 60 kg starting room (the template's baked palette, listed explicitly so a fresh clone repaints to a known state); indices 2-17 are the milestones. The 108-entry generated tail **clamps** to tier 17 -- authoring a hundred restaurants is not a thing anyone should do, and it is the same answer `sizeStage` gives.

Painting is **server-side**, which is the whole reason the feature behaves as specified: the tier comes from one player's peak weight, so it is *their* restaurant, but the parts are ordinary replicated Workspace geometry, so **every visitor standing in the room sees it**. A client-side skin would be a private hallucination no visitor could see.

The floor's light/dark checkerboard is **recovered from tile position**, not from current colour -- every baked tile is just named `Tile`, and classifying by colour would work exactly once and then sort tiles by the *previous* tier's palette on every repaint after the first.

Two triggers, and both are needed. `PlotService.paintOwner` calls `applyTier` on its existing two-second tick, which covers joins, loads and a change of owner; and `PlayerService.setMilestoneHook` -- injected by `PlotService.start()` exactly as `setEatAreaResolver` is, because PlotService already requires PlayerService and the reverse would be a cycle -- repaints on the same frame as the toast, so the new building and the confetti read as one beat instead of two events two seconds apart. `applyTier` repaints **only on change**; the guard is load-bearing rather than an optimisation, since a repaint touches every tile, wall, bench and light in the room and it runs for every plot on a tick. The hook fires **once per batch, not once per milestone**: `Debug setKg 10000` crosses sixteen at once, and the hook re-reads `peakKg` so only the final tier matters. It is wrapped in `pcall` -- a cosmetic repaint must never take down the action that granted the milestone.

Stations, glass, worker furniture, the tray and `DropZone` are deliberately not repainted. On release the permanent restaurant is reset to tier 1, so the next owner never inherits the previous owner's skin. Checkerboard recovery still keys from stamped `PlotOrigin` and preserves parity on rotated rows.

### UI and input

The client creates `McFattyUI` under `PlayerGui`; `StarterGui` itself is empty. The UI is authored around `1280x720`, uses a `0.72` narrow-screen floor, changes the tray to five columns on narrow screens, and lifts the tray above the mobile safe area.

`McFattyUI` has `IgnoreGuiInset = false`, and `fitToViewport` subtracts `GuiService:GetGuiInset()` from the viewport height before computing the scale. Both halves matter and go together: with the inset ignored the ScreenGui's origin sits ~58px ABOVE the visible top of the screen and the stat bar authored at `y = 10` is drawn under Roblox's topbar, and dividing by the raw viewport height tells the layout it owns 720 authored pixels when the drawable area is ~662, which pushes the bottom-anchored cards up into the left column. The two OVERLAY guis -- `McFattyDragLayer` and `McFattyBitePrompt` -- keep `IgnoreGuiInset = true` and no `UIScale`, because they work in raw pointer pixels. Nothing else moved when the main gui changed: every remaining anchor is bottom- or right-relative and resolves to identical pixels either way, which is why tray drop hit-testing is unaffected.

The left column is a single fixed vertical budget in authored pixels, and the three things in it must not be sized independently:

| Band | Authored y | Owner |
|---|---|---|
| stat pills | `8`-`82` | `UI/HUD` |
| chip row | `92`-`130` | `UI/HUD` |
| navigation rail | `122`-`570` | `UI/Rail` (`Rail.Bottom`) |
| free-bag panel | `584`-`762` max | `UI/Pending` |
| tutorial pointer | tracks its target | `UI/Tutorial` (own unscaled `ScreenGui`) |

`Rail.Bottom` is exported and `Client/Main` passes it to `Pending.new`, so the panel follows the rail rather than repeating a literal. Rail captions live INSIDE their 80px button at a measured font size (`Rail.captionTextSize` picks the largest whole size whose unwrapped word fits); they used to hang below it in a taller, wider holder, which put UPGRADES off the left edge of the screen and drew DAILY straight through the free-bag panel. `Pending` scrolls past three rows -- nine bag types can be owed at once.

**The left column overflows at its worst case, and always has.** Five rail tiles plus three owed-bag rows reach roughly y=762 against a 720px design viewport, so the bottom of a full free-bag panel is clipped. `Rail.SIZE`/`GAP`/`TOP`, `Pending.ROW_H` and `Pending.MAX_VISIBLE_ROWS` are the knobs; the tile-size pass that took rail buttons from 74 to 80px paid for most of it out of `GAP` and `TOP` so `Rail.Bottom` moved ten pixels rather than forty. A real fix means either a scrolling left column or fewer visible owed-bag rows, and neither is done.

**One navigation surface at a time.** `GroupedMenu` replaces the six-tab strip with four top-level groups: Food (Shop/Upgrades), Staff, Progress (Perks/New Diet), and Records (Collection/Awards). The rail exposes Food, Staff, Progress, Records, and Daily; an in-world StaffBoard opens the same Staff destination. `Client/Main.syncNavVisibility()` stands the rail, free-bag panel, and tutorial pointer down while the menu is open.

Both right-hand panels are sized `UDim2.new(0, 360, 1, -288)` with a `UISizeConstraint` capping them at their own content height. They were sized to content -- 890px of shop rows on a 720px design viewport -- which put the last three bags and the scrollbar off the bottom edge, unreachable, with the New Diet card drawn through what remained. The `200`px they leave at the bottom belongs to the tray and that card.

Primary UI:

- two top HUD PILLS (weight, Calories) over one row of CHIPS (next milestone, rate, digestion). Two shapes and no third, so a glance separates balances from notes;
- the milestone is a chip only. It used to be drawn twice -- once as a fill along the bottom edge of the weight pill and once as the chip -- and the stripe cost the pill its vertical centre while saying less than the chip already did;
- bottom tray grid with persistent Fill Tray and Auto-Merge actions;
- visible +1/+10/+100/MAX purchase selector shared by Kitchen and Run Upgrade cards;
- Staff page with ten rows, hire/Kitchen-lock state, live cap/kg/Cal-s/meals/buffs, per-worker working toggle, +1/+10/+100/MAX leveling, shared reserve, and Pause All;
- right-side mutually exclusive bag shop and upgrades panel;
- left navigation rail for the purchase/progression surfaces plus Daily, with Collection and Awards available inside the full menu; captions stay inside the buttons and the rail hides while the menu is open;
- icon-first Collection page with one permanent row per thirty base foods, lifetime eat/weight totals, and highest Menu Ascension eaten;
- data-driven Awards page with twenty-five achievements; progress is derived from the authoritative snapshot and earned state is server-owned;
- compact early-run New Diet goal that expands from `3,500 kg`;
- conditional free-bag panel;
- first-run four-step tutorial;
- queued toasts and modal panels for daily, offline, milestones, and rebirth.

Tray controls:

- bare click on food does nothing;
- hold and drag a tile onto a matching tile to merge;
- drag a tile outside the UI onto the booth tray/drop zone to plate it;
- hold the tracked screen-space Eat button; local bites animate at `4/s`;
- release always ends the drag; no sticky cursor mode;
- **`E` fills the tray** (keyboard only; the button reads `FILL TRAY  [E]` when a keyboard is present). `E` is also Roblox's default `ProximityPrompt` key for the object stations and sliding doors, so `Client/Main` counts live prompts via `ProximityPromptService.PromptShown`/`PromptHidden` and stands the hotkey down while any is in range — the world prompt wins when one is shown, and the hotkey wins everywhere else. `gameProcessed` suppresses it while a text box has the keystroke. The key and the button share one legality check, `Tray:canFill()`;
- a max-tier match is refused with explicit feedback;
- merge results are never predicted client-side.

`TableFood` creates client-only pooled blocks under `Workspace.TableFood`:

- mirror blocks show owned tray food on the table;
- one plate block represents the currently server-reserved tier;
- tracked, screen-clamped widgets keep Bite and Drop Here reachable at any camera angle;
- the item stays in the authoritative tray until a timely `CompleteEat`;
- the plate clears only after the completion notification and confirming authoritative snapshot, or on cancellation/respawn/stale ownership.

`CollectionBoard` replaces the physical menu board locally per player. Undiscovered foods render as grey `???` cells; successful eating reveals the icon/name using persisted `tier_N` IDs.

### Camera, audio, and effects

- Camera distance is piecewise from a `12.5`-stud starter distance, grows with the actual character bounds, adds a narrow-screen buffer, and caps required minimum zoom at `55`.
- UI sounds use engine-built-in assets and a small reusable pool.
- Merge cascades rise in pitch.
- World effects scale with character size and clean themselves through Debris.
- Screen effects are fire-and-forget and self-cleaning; confetti is capped at `90` simultaneous pieces.

## Structure snapshot

These paths and counts were observed through MCP on the verification date. Differences are a signal to inspect and update this context; do not blindly force Studio back to these counts.

The counts below describe the final Edit DataModel after the permanent-restaurant rebuild. Workspace is dominated by twenty full restaurant clones.

### Root counts

| Root | Direct children | Descendants |
|---|---:|---:|
| `Workspace` | 3 | 4,840 |
| `ReplicatedStorage` | 2 | 18 |
| `ServerScriptService` | 1 | 13 |
| `ServerStorage` | 5 | 203 |
| `StarterPlayer` | 2 | 29 |
| `StarterGui` | 0 | 0 |
| `StarterPack` | 0 | 0 |
| `Lighting` | 6 | 6 |
| `SoundService` | 0 | 0 |
| `Teams` | 0 | 0 |

Top-level game structure:

```text
Workspace
├── Camera
├── Terrain
└── Restaurants
    ├── RoadNetwork   (RoadOuterA, Boulevard, RoadOuterC, ConnectorWest, ConnectorEast,
    │                  crosswalks and curb ramps; 366 descendants, all anchored parts)
    ├── Boundary      (Ground grass slab, hedge ring, collision backstops, buffer trees)
    └── Plot01 .. Plot20   (permanent; each: Lot, OwnerSign, Restaurant)
                            Restaurant holds Floor, Walls/doors, Counter, Lights, Booth,
                            BoothSpawn, object Stations, and ten WorkerStations)

ReplicatedStorage
├── Shared
│   ├── Economy
│   ├── Format
│   ├── ActionSpec
│   ├── Meal
│   └── Config
│       ├── Food
│       ├── Bags
│       ├── Progression
│       ├── Balance
│       ├── Upgrades
│       ├── Stations
│       ├── Kitchen
│       ├── Perks
│       ├── RestaurantTiers
│       ├── Achievements
│       └── Workers
└── Remotes

ServerScriptService
└── Server
    ├── Main
    └── Services
        ├── DataService
        ├── DoorService
        ├── PlayerService
        ├── CharacterService
        ├── PlotService
        ├── SeatingService
        ├── StationService
        ├── WorkerService
        ├── LeaderboardService
        └── RestaurantSkin

ServerStorage
├── PacingSim
├── EconomyTests
├── MapBuilder
├── StationsBuilder
└── RestaurantTemplate   (Model, not a script: the authoritative restaurant, 198 descendants)

StarterPlayer
├── StarterPlayerScripts
│   └── Client
│       ├── Main
│       ├── CameraController
│       ├── WorldEffects
│       ├── TableFood
│       ├── MealCharacterAnimation
│       ├── CollectionBoard
│       ├── PlotRoot
│       └── UI
│           ├── Theme
│           ├── Tray
│           ├── HUD
│           ├── Shop
│           ├── Panels
│           ├── Tutorial
│           ├── Pending
│           ├── UpgradesPanel
│           ├── PerksPanel
│           ├── Menu
│           ├── GroupedMenu
│           ├── StaffPanel
│           ├── CollectionPanel
│           ├── AchievementsPanel
│           ├── Rail
│           ├── RebirthCard
│           ├── Sounds
│           └── Effects
└── StarterCharacterScripts
```

### Exact script inventory and responsibility

Shared:

- `game.ReplicatedStorage.Shared.Economy` — pure authoritative rules and state mutations.
- `game.ReplicatedStorage.Shared.Format` — display-only number, weight, rate, and duration formatting.
- `game.ReplicatedStorage.Shared.ActionSpec` — exact, bounded remote action-shape validation plus action token cost/cooldown.
- `game.ReplicatedStorage.Shared.Meal` — deterministic begin/complete reservation state machine for server-timed eating.
- `game.ReplicatedStorage.Shared.Config.Food` — generated food ladder and merge shape.
- `game.ReplicatedStorage.Shared.Config.Bags` — bag prices, unlocks, drops, EV, and server RNG roll.
- `game.ReplicatedStorage.Shared.Config.Progression` — milestones, size stages, permanent derived rewards.
- `game.ReplicatedStorage.Shared.Config.RestaurantTiers` — seventeen restaurant palettes, one per size stage, plus `tierFor(peakKg)`.
- `game.ReplicatedStorage.Shared.Config.Achievements` — twenty-five data-driven achievement definitions, progress evaluators, and server-owned earn stamping.
- `game.ReplicatedStorage.Shared.Config.Balance` — global tuning and boot-time invariant validator.
- `game.ReplicatedStorage.Shared.Config.Upgrades` — RUN upgrade definitions and cost curve (levels reset on New Diet).
- `game.ReplicatedStorage.Shared.Config.Kitchen` — Kitchen ladder: per-level max producible tier, accelerating cost curve, structural validator.
- `game.ReplicatedStorage.Shared.Config.Perks` — the 16 permanent perks, their Diet Point cost curves, effect modes, and validator.
- `game.ReplicatedStorage.Shared.Config.Stations` — world station definitions and UI destinations.
- `game.ReplicatedStorage.Shared.Config.Workers` — ten worker identities, tier ceilings, hire/level curves, milestone buffs, automation cadence, reserve defaults, and offline/workerless tuning.

Server:

- `game.ServerScriptService.Server.Main` — server entry point and remote creation.
- `game.ServerScriptService.Server.Services.DataService` — session-locked DataStore or memory fallback.
- `game.ServerScriptService.Server.Services.PlayerService` — session ownership, actions, accrual, sync, autosave, progress events.
- `game.ServerScriptService.Server.Services.CharacterService` — stage-to-character rendering and movement lock.
- `game.ServerScriptService.Server.Services.PlotService` — per-player plot assignment in the documented central-first order, reuse/reset of each permanent restaurant, owner wall boards, restaurant tier repaints, and the eat-area resolver and milestone hook injected into `PlayerService`.
- `game.ServerScriptService.Server.Services.DoorService` — registers every restaurant's E prompt, slides both glass leaves/handles, controls collision, tracks doorway occupancy, and closes after two empty seconds.
- `game.ServerScriptService.Server.Services.SeatingService` — server-owned player meal seating, anchoring and movement restoration across every completion/cancel/lifecycle path.
- `game.ServerScriptService.Server.Services.WorkerService` — creates and updates purchased worker characters, capped visual growth, seated presentation, active plates, and procedural fidgets/eating loops.
- `game.ServerScriptService.Server.Services.RestaurantSkin` — applies a `RestaurantTiers` palette to a plot's permanent `Restaurant`; idempotent and server-side so visitors see it.
- `game.ServerScriptService.Server.Services.StationService` — validated ProximityPrompt to client station notification.
- `game.ServerScriptService.Server.Services.LeaderboardService` — server-local loaded-player top-five board with sanitized labels and coalesced refresh, written into every occupied restaurant.

ServerStorage tools:

- `game.ServerStorage.PacingSim` — headless campaign and single-run pacing model using shipped economy, worker automation, difficulty, Kitchen, upgrade, and perk rules.
- `game.ServerStorage.EconomyTests` — fresh-clone headless regression checks.
- `game.ServerStorage.MapBuilder` — destructive permanent-district rebuild (`build()`, Edit-mode only), restaurant template baker (`buildTemplate()`), and additive boot repair for missing permanent restaurant models (`ensureTodoWorld()`).
- `game.ServerStorage.StationsBuilder` — bakes Till/Fryer/Scale/StaffBoard objects and the overhead lifetime-mass marquee into the template (`buildInto`); explicit template station rebuild (`build()`) plus additive boot migration (`ensureTodoStations()`).

Client:

- `game.StarterPlayer.StarterPlayerScripts.Client.Main` — client entry point, UI composition, remote wiring, action dispatch.
- `game.StarterPlayer.StarterPlayerScripts.Client.CameraController` — camera framing for growing characters.
- `game.StarterPlayer.StarterPlayerScripts.Client.WorldEffects` — character/world particle and ring effects.
- `game.StarterPlayer.StarterPlayerScripts.Client.TableFood` — table mirror, plate, bite tracking, drag-to-world target, and viewport-safe tracked-button clamping.
- `game.StarterPlayer.StarterPlayerScripts.Client.MealCharacterAnimation` — three procedural seated upper-body eating loops for the local player.
- `game.StarterPlayer.StarterPlayerScripts.Client.CollectionBoard` — per-player menu collection and scale display.
- `game.StarterPlayer.StarterPlayerScripts.Client.PlotRoot` — resolves the local player's own optional permanent `Restaurant` from the replicated `PlotIndex` attribute; nil during early assignment, replication delay, or streaming, and callers tolerate all three.
- `game.StarterPlayer.StarterPlayerScripts.Client.UI.Theme` — visual tokens and widget constructors.
- `game.StarterPlayer.StarterPlayerScripts.Client.UI.Tray` — tray rendering and drag/merge gesture.
- `game.StarterPlayer.StarterPlayerScripts.Client.UI.HUD` — stats, local Calories prediction, milestone progress.
- `game.StarterPlayer.StarterPlayerScripts.Client.UI.Shop` — bag purchase/fill panel.
- `game.StarterPlayer.StarterPlayerScripts.Client.UI.Panels` — queued toasts and daily/offline/rebirth modals.
- `game.StarterPlayer.StarterPlayerScripts.Client.UI.Tutorial` — inferred first-run tutorial.
- `game.StarterPlayer.StarterPlayerScripts.Client.UI.Pending` — free pending bags panel.
- `game.StarterPlayer.StarterPlayerScripts.Client.UI.UpgradesPanel` — Run Upgrade panel, with the pinned Kitchen card above the list.
- `game.StarterPlayer.StarterPlayerScripts.Client.UI.PerksPanel` — permanent perk panel and Diet Point balance.
- `game.StarterPlayer.StarterPlayerScripts.Client.UI.GroupedMenu` — active four-group navigation shell: Food, Staff, Progress, and Records.
- `game.StarterPlayer.StarterPlayerScripts.Client.UI.StaffPanel` — worker hiring/leveling, automation toggles, shared reserve, Pause All, caps, output, and buff presentation.
- `game.StarterPlayer.StarterPlayerScripts.Client.UI.Rail` — left navigation and pending-reward pip.
- `game.StarterPlayer.StarterPlayerScripts.Client.UI.RebirthCard` — always-visible New Diet goal/action.
- `game.StarterPlayer.StarterPlayerScripts.Client.UI.Sounds` — pooled engine-built-in UI audio.
- `game.StarterPlayer.StarterPlayerScripts.Client.UI.Effects` — screen pop/float/confetti/flash/shake effects.
- `game.StarterPlayer.StarterPlayerScripts.Client.UI.Menu` — retained legacy six-tab shell module; `GroupedMenu` is the active composition path.
- `game.StarterPlayer.StarterPlayerScripts.Client.UI.CollectionPanel` — one row per base food with discovery and lifetime per-food statistics.
- `game.StarterPlayer.StarterPlayerScripts.Client.UI.AchievementsPanel` — data-driven award progress and earned-state rendering.

## Testing and development tools

### Balance validator

The server runs `Balance.validate()` on boot. It checks bag ordering/value, merge/eat ratios, digestion share, end-tier payback drift, and folds in `Kitchen.validate()`, `Perks.validate()`, and `Workers.validate()` so boot has one gate to pass. It also pins structural economy invariants such as `MERGE_CAL_RATIO > CAL_RATIO` and the top merge costing less than the top bag. Economy/config work is incomplete if this fails.

### Economy tests

`ServerStorage.EconomyTests` covers (**168 checks, all passing** as of 2026-08-23, after the permanent-restaurant/worker/endless-progression pass):

- starting income/unlocks;
- `3 -> 2` and `2 -> 1` manual merges;
- merge position stability and rejection cases;
- bulk/auto-merge behavior;
- eat-versus-merge and digestion invariants;
- tray capacity and anti-softlock floor;
- offline rewards/caps/no offline weight;
- rebirth reset/preservation behavior;
- daily streak and wrapping;
- pending bags;
- starting `300` Calories and bounded `FillTray` work/reporting;
- exact hostile action shapes including extra args, fractions, NaN, infinity, legacy `Eat`, and non-Studio debug;
- Auto-Merge affordability in the opening minute, its tier cap, and its paused state;
- finite upgrade caps being reachable and infinite Bigger Bites/Metabolism accepting bulk levels beyond 100;
- Bigger Bites scaling weight only (not Cal/s, not digestion) and preserving Eat-vs-Merge at max level;
- Fast Chew shortening the hold without changing the bite count;
- Lucky Bite and Double Meal payout, their miss case, their no-RNG case, and Double Meal never consuming a food of a different tier;
- total bag value falloff and tier-10 weight-per-Calorie sanity;
- pair-merge bonus hit and miss branches, full groups never rolling it, the tray count never growing past the slot limit, Auto-Merge being unable to farm it, and the no-RNG case;
- a raised Auto-Merge level merging the tray already held;
- meal early completion, success, replay, and stale ownership;
- discovery award, migration sanitization, and New Diet preservation;
- balance validation;
- the rebirth curve growing after a diet, a fresh diet being unable to immediately rebirth again, and the effective endless difficulty divisor increasing toward its ceiling;
- bag AND upgrade prices scaling with completed diets, mid-run metabolism NOT moving prices, and `buyBag` charging the scaled price;
- meal duration being floored regardless of Fast Chew level;
- a fully upgraded tray landing exactly on `Balance.MaxSlots`, with `bonusSlots` stacking on top;
- migration rejecting malformed trays, clamping weight/Calories/diets, dropping unknown upgrades and pending bags, and refusing a future `lastSeen`;
- invalid actions carrying a rate-limit price above a valid one;
- climbing the food ladder beating tier-1 spam on weight per MEAL;
- **Kitchen**: a fresh run starting at Kitchen 1, merging INTO the ceiling being allowed and PAST it refused, buying the next Kitchen unblocking exactly that merge, Auto-Merge stopping at the ceiling however high its level, the reported Auto-Merge reach being clamped (not just its behaviour), and 60 top-bag rolls never beating the ceiling;
- **merge prices**: the deduction happening, an unaffordable merge being refused rather than given away, Auto-Merge being unable to merge for free, and prices outrunning income per tier;
- **New Diet**: run purchases wiped (upgrades cleared, Kitchen back to 1, digestion zeroed, workers unhired/reset), worker automation preferences and the permanent layer kept, and the payout matching the previewed number exactly;
- **Diet Points**: pushing paying more but sublinearly, the 8x cap making "park forever" never optimal, a below-target diet paying nothing, and later diets paying more;
- **Perks**: refusal without points, unknown ids refused, the deduction, accelerating prices, a maxed perk refusing further levels, Eat-vs-Merge surviving maxed weight perks, and Auto-Chef giving a fresh run a working Auto-Merge with no upgrade bought;
- **new-field save validation**: an impossible Kitchen level repaired, Diet Points repaired, perk levels clamped and unknown perks dropped, and lifetime points reconciled upward against the balance;
- bulk Kitchen/upgrade hostile-shape validation and exact +1/+10/+100/MAX behavior;
- worker hire order, ten-worker limit, identity/Kitchen tier caps, private six-slot trays, shared reserve, pause/enable toggles, level costs and milestone buffs;
- worker kilograms remaining separate from player kilograms while adding permanent run digestion, offline worker production applying exactly 50%, and malformed worker actions being refused;
- `BuyKitchen`/`BuyKitchenBulk`, `BuyUpgradeBulk`, and worker action contracts accepting only their documented shapes.

The Phase 1 checks live inside an immediately-invoked function rather than inline. That is a compiler constraint, not a style choice: Luau reserves one register per local for the whole function and caps at 200, which `Tests.run` had nearly reached -- a plain `do` block does not give the registers back, so adding fixtures inline fails to COMPILE with "Out of local registers". Merge fixtures also go through a `workshop()` helper that sets the top Kitchen and a large Calorie balance, because a merge-shape test that fails on the tier-4 starting ceiling is testing the wrong thing.

**Run it with a fresh CLONE of the test module too.** It already clones `Shared`, but Studio memoizes the test ModuleScript itself, so requiring `ServerStorage.EconomyTests` directly after editing it silently runs the previous version -- which looks exactly like an edit that did nothing.

Run it through MCP in Edit mode. It clones Shared to avoid ModuleScript cache staleness. Clean up `ServerStorage.__TestSandbox` after the run if it remains.

### Pacing simulator

`ServerStorage.PacingSim` drives the shipped `Shared.Economy` rules and models a finite action budget, bounded fill-tray purchasing, Kitchen and upgrade investment, paid merges, Auto-Merge, deterministic meal holds, Diet Point spending, the ten worker automations, worker tier/Kitchen caps, and the endless difficulty divisor. Use a fresh clone after economy/config edits because Studio memoizes ModuleScripts.

`Sim.campaign(policy, seed, diets)` plays consecutive diets on one state; it is the honest way to measure the rebirth curve because the upgrades, Kitchen, perks, worker purchases, and automation state earned in earlier runs dominate later performance. `Sim.play` is the single-run primitive and `Sim.run` wraps a fresh state.

Current staffed campaign measurement, seed `12345`:

| diet | visible band | seconds | minutes |
|---:|---:|---:|---:|
| 0 | 1x | 949 | 15.8 |
| 1 | 2x | 1,749 | 29.1 |
| 2 | 3x | 2,750 | 45.8 |
| 3 | 4x | 3,702 | 61.7 |
| 4 | 5x | 6,158 | 102.6 |
| 5 | 6x | 7,379 | 123.0 |
| 6 | 7x | 8,496 | 141.6 |
| 7 | 8x | 10,119 | 168.7 |
| 8 | 9x | 12,021 | 200.4 |
| 9 | 10x | 12,839 | 214.0 |
| 10 | 11x | 13,744 | 229.1 |
| 11 | 12x | 13,163 | 219.4 |
| 12 | 13x | 10,845 | 180.8 |

All thirteen runs reached their target. The effective difficulty factor is monotonic even though earned perks, worker buffs, and Kitchen timing make individual run times fluctuate; the longest measured staffed run was **3h 49m**, below the four-hour late-game target. A workerless diet-0 run measured `2,721s` versus `949s` staffed, or **2.87x longer**, close to the intended approximately-3x penalty without making staff mandatory.

`PacingSim` floors meal duration exactly as `Meal.begin` does, funnels all generated tiers through the Kitchen ceiling, and prices bags/upgrades/merges through the same Economy functions as the server. Do not claim pacing numbers from comments as current measurements; rerun the fresh-clone campaign after relevant changes.

### Runtime verification

Use Play mode through MCP for client/server behavior, then inspect console output. Relevant cases include:

- fresh join with Studio memory fallback;
- buy, fill, manual pair/full merge, Auto-Merge, BeginEat, timed hold, CompleteEat, cancel, replay, and stale ownership;
- full and empty trays;
- milestone and body-size transitions;
- respawn at a non-default size;
- stage-10 movement lock and post-rebirth unlock;
- station prompts and navigation rail parity;
- pending bags, offline/daily modals, and rebirth;
- mobile/small viewport layout and touch dragging;
- malformed/replayed actions and rate limiting;
- missing `Restaurant`, `Stations`, `Booth`, `Tray`, or `DropZone` behavior where relevant.

Latest Play verification on **2026-08-23**, after the **permanent restaurants, workers, grouped UI, seated meals, doors, and endless progression pass**:

- a fresh-clone `EconomyTests` run passed **168/168**, and the final static compile pass parsed **24/24 affected scripts**; the Edit DataModel contains **55 scripts** at the exact paths in this inventory;
- Edit mode contains twenty plots, twenty complete permanent restaurants, **200** worker stations, twenty door prompts, and zero legacy Cashier/Fry Cook/Nutritionist character models; `Workspace.Restaurants` has 4,837 descendants;
- the final fresh-clone staffed `PacingSim` campaign reached every diet-0..12 target, measured 15.8 minutes early and at most 3h49m late, while workerless diet 0 was 2.87x slower;
- Play reached `[McFatty] server ready`; Plot08 assigned normally. The only console warnings were the known `MaxPlayers=60` versus twenty plots, Studio DataStore API-off fallback, and Animation Mirror plugin messages;
- a real server action hired worker 1 after adding test Calories: `WorkerCount` became 1, exactly one `Worker_01_busser` appeared under the owner's `WorkerCharacters`, and it showed exactly one visible `ActivePlate`; malformed worker id/amount/reserve actions did not mutate the count;
- the live UI tree contained `GroupedMenu`, the Staff page, Pause All, all ten worker rows, +1/+10/+100/MAX controls, and the Food/Staff/Progress/Records/Daily rail destinations;
- a real meal anchored and seated the player with movement locked (`WalkSpeed=0`, `AutoRotate=false`), and cancel restored the original unanchored `WalkSpeed=16`, `AutoRotate=true` state;
- the first narrow-window meal attempt exposed an inverted `math.clamp` range in `TableFood.trackedPosition`; the client module was fixed and the complete live flow was rerun without that error;
- `DoorService` registered every E prompt. Studio's keyboard injector did not fire `ProximityPrompt.Triggered`, so the service's open/occupancy/close path was exercised through a temporary Studio-only test attribute that was removed afterward: leaves/handles moved from approximately `±6.55` to `±19.05`, collision disabled, occupancy kept the entrance open, and moving the player clear closed it after two seconds at the original positions with collision restored.

**Not verified in this pass:** a true two-client visitor session, and the final E keystroke-to-`Triggered` hop through the automation tool. The production prompt is directly connected and its complete door state machine passed, but the tool itself emitted no prompt event. The Studio save checkpoint also remains pending because the Windows-control helper failed to initialize and the Studio MCP connection exposes no save command.

Earlier Play verification on **2026-08-23**, after the **restaurant entrance pass**:

- `MapBuilder` compiled from a fresh clone, then `buildTemplate()` rebuilt only `ServerStorage.RestaurantTemplate`; after the final two-sided marquee label, the template has 147 descendants and twenty `Walls` children, including split front wall/header pieces, a three-piece frame, two glass leaves, and two handles;
- the live Plot08 runtime clone carried the expected entrance structure; both leaves reported `CanCollide=false`, `CanTouch=false`, and `CanQuery=false`;
- a real client character navigated from the booth side through the doorway to the frontage side (server root reached approximately `0, 3.25, -46.75`) and then back inside (approximately `0, 4.22, -65.41`);
- the initial enlarged-character check exposed the old 56x20 `RecordBoard` directly across the opening; `StationsBuilder` moved and resized it to a 56x8x2.2 marquee at local `(0, 29, 60)`, above the door, with the existing owner record inside and `COME ON IN` outside;
- after that correction, a 2,200 kg size-stage-10 character with an approximately `11.52x9.20x6.02` bounding box navigated fully outside (`z=-46.85`) and back inside (`z=-65.12`) through the entrance;
- an Edit-mode clone on rotated Plot13 preserved entrance/path orientation; a centre ray through the opening hit nothing while an adjacent ray hit `Wall2Right`, proving the rotated entrance is open only where intended;
- exterior screen inspection confirmed the double doors, frame, handles, and frontage path are centred and visually coherent;
- console reached `[McFatty] server ready` with no new errors; only the already-known MaxPlayers/DataStore and Animation Mirror plugin warnings appeared.

**Not verified in the entrance pass:** a true two-client visitor session. Access has no ownership code or collision gate, so the same replicated opening is structurally available to every character, but the Studio connection provided one client only.

Latest Play verification on **2026-08-21**, after the **dynamic restaurant district** pass:

- Edit-mode validation: exactly twenty unique `PlotIndex` values in stable row-major placement; every lot has `Lot` + `OwnerSign`, a valid `PlotCFrame` (rows 2-3 rotated 180 degrees) and a double-sided sign reading AVAILABLE; no baked restaurants; `RestaurantTemplate` carries Floor/Walls/Counter/Booth/Lights/BoothSpawn/Stations, a `PivotAnchor` PrimaryPart pivot at ground level, and 3 prompts; RoadNetwork and Boundary are 366 + 69 anchored-part descendants with no per-object scripts (664 BaseParts under `Restaurants` total);
- clone simulation in Edit mode: a template clone pivoted onto a rotated row-2 plot kept correct interior geometry (spawn position/facing, tray, scale dial, record board and menu board all mirrored correctly), and an unrotated row-1 plot was spot-checked the same way;
- Play mode: server reached `[McFatty] server ready` (so `Balance.validate()` passed) with only the expected DataStore-fallback and capacity warnings; the first player was assigned **Plot08** (row 1, column 2, on the boulevard); the sign read `PolarBearQ's Restaurant` -- a 16-character username truncated to ten with casing preserved, on both faces; `RespawnLocation` matched the clone's `BoothSpawn` and the character spawned there;
- `FillTray` + `BeginEat` + `CompleteEat` over the real remote moved 60 -> 63.6 kg with digestion income live; the cashier prompt opened the shop UI; the collection board disabled the baked menu SurfaceGui and mounted its local one; the `SizeLeaderboard` row inside the nested clone rendered the player;
- `Debug setKg 12000` repainted the clone to tier 17 (OUTGROWN) with checkerboard parity intact (18 light + 18 dark tiles);
- the eat presence check refused `BeginEat` from ~4,000 studs away (no weight change) and accepted it again back at the booth;
- killing the character respawned it at its own `BoothSpawn`; kicking the player destroyed the restaurant immediately and reset both sign faces to AVAILABLE with lot and roads intact; a fresh Play session reassigned the same lot cleanly.

**Not verified in the district pass:** multi-player allocation (the central-first order is deterministic in code and was verified single-player only), sidewalk/crosswalk walkability for enlarged characters, and the manual Game Settings change to `MaxPlayers = 20` (still pending at handoff; the boot warning names it until then).

Previous Play verification, after the **Phase 1 progression pass** (Kitchen, merge costs, Run Upgrades, Diet Points, Perks):

- server boot reached `[McFatty] server ready` (which includes a passing `Balance.validate()` with the new Kitchen/Perks/merge-price invariants) with no errors beyond the expected Studio DataStore API-off fallback;
- `EconomyTests` **139/139** and all 54 scripts compiled in that historical pass; its then-current Phase 1 campaign was also measured;
- the snapshot carried every new field: `kitchen`/`kitchenName`/`kitchenMaxTier`/`kitchenCost`/`kitchenMaxLevel`, `mergeCosts`, `dietPoints`, `perks`, `perkCosts`, `dietPointsIfNow`;
- a real `BuyKitchen` through the remote moved K1 -> K2 and the ceiling T4 -> T6;
- the full loop end to end: bought a Run Upgrade, `Debug setKg 12000`, previewed 8 Diet Points, `Rebirth` paid exactly 8, and afterwards `diets=1`, upgrades **cleared**, Kitchen back to **K1**, `kg=60`; `BuyPerk stomach` then charged 2 points and granted level 1;
- UI verified on screen: the PERKS rail button (with its unspent-points pip), the perks panel showing the balance and all 16 rows sorted by price, and the pinned Kitchen card reading "HOME KITCHEN Lv 1/9 / Makes food up to TIER 4 / NEED 3.53K Cal" at diet-1 prices.

Previous Play verification, after the "late game is free" rebalance and exploit-hardening pass:

- server boot reached `[McFatty] server ready`, which includes a successful `Balance.validate()`, with no project error or warning beyond the expected Studio DataStore API-off fallback and a Roblox Assistant-plugin version notice;
- `EconomyTests` **107/107**; `PacingSim` campaign as tabulated above;
- the snapshot carried the new `rebirthKg` (10,000), `priceScale` (1), and the full nine-entry `bagCosts` map at diet-0 prices (snack 100 ... franchise 8,170,000);
- a real `BuyBag` then `FillTray` through the remote bought and merged correctly; a full timed meal took 60.00 -> 62.40 kg (tier 1 at multiplier 1) and fired `Discovery`;
- an early `CompleteEat` was refused without consuming food;
- **40 well-formed-but-rejected `Merge` actions and 40 malformed `BuyBag` actions each produced ZERO `Sync` snapshots**, and the malformed batch produced no notifications at all;
- **meal rate limit**: 40 begin/cancel pairs at 7.9/s yielded 25 started and 15 refused with "Slow down -- finish your mouthful", matching the 3/s + burst 12 budget;
- **eat presence check**: with the SERVER moving the character 4,243 studs from its plot, `BeginEat` was refused; back home it started normally. (A client-side teleport does NOT reproduce this -- the server never sees it, which is itself worth knowing: the first attempt to test this measured a character the server still had 2.8 studs from its booth.)
- **rebirth milestone bug fixed**: `Debug setKg 10000` fired 16 milestones once; the subsequent `Rebirth` fired exactly ONE `Rebirth` notification and ZERO milestones, where the old `lastMilestoneKg = 0` would have re-fired all 16 toasts, flashes and confetti bursts;
- after that rebirth: `diets=1`, `kg=60`, `rebirthKg=35,000`, `priceScale=1.41`, snack 100 -> 141, monstrosity 110,000 -> 155,263, tray slots preserved at 12, `canRebirth=false`;
- `Players.MaxPlayers` confirmed read-only at runtime (`Unable to assign property MaxPlayers`), which is why capacity is a boot warning plus a kick rather than a clamp;
- plot assignment worked: `PlotIndex=1`, `ProfileLoaded=true`, 20 plots found under `Workspace.Restaurants`.

**Known layout gap:** at the narrow-screen floor the tutorial card and the five-column tray still share the bottom-left corner. The authored canvas at `0.72` is ~542x1092, the tray sits at `y = 718`-`988` and `x = 35`-`507`, and the tutorial card at `x = 12`-`262`, `y = 954`-`1080` runs into it. It only bites during the first run on a phone; the desktop column is clear.

**Not verified in the Phase 1 pass:** the perks/upgrades panels at narrow (phone) viewports -- the perks panel reuses the upgrades panel's constraint math and adds a taller header, and the rebirth modal grew from 260 to 300px; neither was checked at the 0.72 scale floor. The rail is also one button longer now (five tiles), which moves `Rail.Bottom` and therefore the free-bag panel and tutorial card beneath it. The known narrow-screen tutorial/tray overlap below is unchanged and untested against the taller rail.

**Not verified in the earlier pass:** client UI layout for a 20-slot tray at narrow viewports, the collection board with 20 cells, mobile/touch dragging, offline/daily modals, and the DataStore pending-save retry path (Studio has API access off, so only the memory-fallback branch was exercised). The tier 13-20 tray/board visuals were extended and asserted equal-length but not looked at on screen.

## Known architectural constraints and change-sensitive couplings

- Studio is currently the only implementation source. Local Luau edits do not affect the game.
- `Remotes` are runtime-created; do not infer a missing contract from an empty Edit-mode folder.
- The room size/counter reference is mirrored between `MapBuilder` and `Config.Stations` because the client cannot require ServerStorage.
- `Progression.SizeStages`, `CharacterService.SCALE`, and `CharacterService.GIRTH` must have identical lengths (17), and `CharacterService` asserts it on load. The size ladder intentionally stops at the first rebirth target while weight continues to eight figures.
- Food ratios, bag prices, digestion share, bite counts, and action assumptions jointly determine pacing; change them as a system and re-run the simulator.
- **`Food.FoodsPerMenu`, `Food.CalRatio`, and `Food.AscensionPower` are a system.** Payback drift is bounded over one thirty-food cycle, then bag/Kitchen prices must scale by the same full-cycle income factor so the unbounded ladder resets its economics at every Menu Ascension.
- **`Balance.MetabolismPerDiet`, `Progression.RebirthGrowth`, and `Progression.difficultyFactor` are one pacing system.** Weight per second scales roughly with the square of the metabolism multiplier, while the difficulty divisor supplies the endless late-run brake and asymptotically caps at 40x. `validate()` pins the structural relationship; `PacingSim` pins the observed campaign.
- **`Economy.kitchenMaxTier` is the ONLY tier gate, and every production path must funnel through it.** Merge, `mergeAll`, `runAutoMerge` and the bag roll (`rollForState`) all clamp against it; adding a new way to acquire or create food without clamping re-opens the bypass the Kitchen exists to close.
- **Auto-Merge's cap is an INPUT tier, one below the Kitchen ceiling.** Clamping it to the ceiling itself would let its last merge produce a locked tier; `Food.AbsoluteMaxTier` is only a hostile-input sanity bound.
- **`Food.MergeCalRatio` must stay above `Food.CalRatio`**, or the tier ladder pays for its own merges faster and faster and climbing becomes free again. `validate()` asserts it.
- **`findMergeGroup` returning the LOWEST-tier group is load-bearing now.** Merge prices rise with tier, so a bulk merger that cannot afford the cheapest group can correctly stop; returning an arbitrary group would abort MERGE ALL on an expensive tier while affordable work remained.
- **Perks must stay flat across food tiers.** Any perk whose magnitude varies with the tier of the food eaten can move `KgRatio` against `MergeCost` and silently kill the Eat-vs-Merge decision. Heavy Eater keys off the PLAYER's weight for exactly this reason. `EconomyTests` pins the invariant at maxed perk levels.
- **Diet Points must not be scaled by `Economy.priceScale`.** `priceScale` exists to keep CALORIE prices honest against a multiplied Calorie income; points have no such inflation, and scaling perk prices would make the ladder unreachable for the players who had earned the most points.
- **Run Upgrades, Kitchen, worker hires/levels/trays/kg/digestion reset on New Diet; perks, `peakKg` unlocks, and worker automation preferences do not.** Moving anything across that line changes the whole pacing curve -- rerun the campaign, not just the tests.
- **`Balance.MaxSlots` must equal base + milestone + purchasable slots exactly.** `validate()` asserts it; more is silently wasted, less makes the cap decorative.
- **`Balance.MinMealSeconds` is load-bearing for the economy, not just for feel.** At zero, weight becomes Calorie-limited again, tier 1 food becomes the optimal strategy, and remote-only eat spam becomes free.
- **`Economy.priceScale` must key on `diets` only.** Keying it on the full multiplier would raise prices every time a player crossed a milestone.
- Percentage upgrades read through `Upgrades.factorAt`, not `1 + amountAt`. `amountAt` deliberately returns `0` for a multiplicative upgrade so a wrong call site fails loudly rather than plausibly.
- `Food.Registry` must contain exactly `Food.FoodsPerMenu` distinct base foods and distinct tray glyphs. Theme colours key by the base menu index, while names/icons come from the registry; all Menu Ascensions reuse those identities.
- Anything player-facing that shows a bag price or the rebirth target must read the SNAPSHOT (`bagCosts`, `rebirthKg`), never the config -- both move with completed diets.
- `PlayerService` must not require `PlotService` (PlotService requires PlayerService). The eat-area resolver is injected via `PlayerService.setEatAreaResolver` in `PlotService.start()`.
- `Progression.rewardsFor` returns a SHARED table; treat it as read-only.
- Player actions use tray indices for merge but tier identity for `BeginEat`. The reservation must still own that tier at completion.
- Bulk and Auto-Merge intentionally ignore pairs even though manual pair merging is allowed. That is also what confines the pair-merge bonus to manual play.
- `Economy.merge` takes an OPTIONAL `rng`. Only the server passes one; omitting it disables the pair bonus, which is what keeps `mergeAll`, `runAutoMerge`, the simulator, and the tests deterministic.
- A truthy THIRD return from an action handler reaches `onAction` as `special`, which fires a `Jackpot` notification. `Merge` reuses that slot for its bonus flag, so its branch must be tested BEFORE the generic `special` branch or a pair merge announces itself as a bag jackpot.
- Raising the `automerge` level does not re-merge the tray on its own; `PlayerService.BuyUpgrade` calls `runAutoMerge` explicitly. Removing that makes the upgrade look inert until the next purchase. `BuyKitchen` and `BuyPerk("autochef")` call it for the same reason -- both raise the ceiling over a tray the player is already looking at.
- The `E` hotkey and the station prompts share a key by design. Adding another prompt anywhere in the world automatically suppresses the hotkey in its radius; adding another keyboard binding on `E` would not.
- Auto-Merge is wired on every acquisition path (`BuyBag`, `FillTray`, `OpenPending`, and enabling the toggle) and was never code-broken; at `upgrades.automerge = 0` it correctly does nothing, and the always-visible tray toggle then reads as a broken feature. Its price is what makes it feel present or absent.
- Chance-based eating effects must roll on the server session's `Random`. Do not let the client supply, influence, or predict those rolls.
- `Economy.eat` / `eatTier` take an OPTIONAL `rng`. Omitting it disables Lucky Bite and Double Meal; several tests and the deterministic paths rely on that.
- Meal reservation/timing is server-owned while bite visuals are client-only. Food is not removed until timely completion; changing this handshake affects networking, replay safety, disconnects, and tutorial confirmation.
- UI drag hit-testing relies on viewport-space coordinates and unscaled overlay ScreenGuis. Mixing them with inset-inclusive mouse coordinates breaks drops. `McFattyDragLayer` and `McFattyBitePrompt` must keep `IgnoreGuiInset = true` and stay out of the main `UIScale`; `McFattyUI` must keep `IgnoreGuiInset = false` so the top HUD clears the topbar.
- The left column is a fixed authored budget shared by the five-entry grouped rail, the free-bag panel and the tutorial card, and the right column is shared by the contextual Food/Staff/Progress/Records content, the tray and the New Diet card. Growing any one of them can overlap a neighbour rather than reflowing. `Rail.Bottom`, `Pending.MAX_VISIBLE_ROWS`, and the panels' `BOTTOM_RESERVE` are the main knobs.
- The game is designed for StreamingEnabled; world-facing client modules must tolerate important instances being absent or late.
- Restaurants are permanent clones of `ServerStorage.RestaurantTemplate`, baked into every lot by `MapBuilder` and retained while unclaimed. World-facing client code must still treat `plot.Restaurant` as optional during early boot, replication delay, or streaming, and never look for `Booth`, `Stations`, boards, or `BoothSpawn` directly under the plot.
- `MapBuilder.build()` destroys/recreates `Workspace.Restaurants` and `ServerStorage.RestaurantTemplate`; `StationsBuilder.build()` rebuilds the template's station set. Inspect and confirm the exact target before running them.
- `ensureTodoWorld()` and `ensureTodoStations()` are the boot paths and must remain idempotent and non-destructive on a valid baked district. `ensureTodoWorld()` may additively restore a missing permanent restaurant at startup, but must not replace or regenerate healthy restaurants.
- Character appearance loading can reset scale values, so size is reapplied after `CharacterAppearanceLoaded`.
- Uploaded catalog media is intentionally avoided in current sound/particle code to reduce asset moderation/deletion failures.

## Context refresh checklist

When Studio differs from the structure snapshot:

1. Re-enumerate the affected hierarchy and every script path through MCP.
2. Read all new/moved/affected scripts completely.
3. Trace new entry points, requires, remotes, attributes, tags, persistence fields, builders, and UI consumers.
4. Update the snapshot metadata, tree, inventory, ownership, lifecycle, networking, state, gameplay, UI/world, tests, and risks that changed.
5. Keep historical claims out of the current-state sections; record only what Studio now contains.
6. Then design and implement the requested change through MCP.

Update this document for behavioral or contract changes even when the instance tree itself did not change.
