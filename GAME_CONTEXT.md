# McFatty game context

This is the living architectural and product context for the McFatty Roblox experience. Read it before planning or changing the game. Roblox Studio is the source of truth; this document is a maintained model of that source, not a substitute for inspecting it.

## Snapshot metadata

- Last verified through Roblox Studio MCP: **2026-08-20** (Asia/Singapore)
- Connected Studio display: **McFatty**
- Place ID: `74686069419969`
- Universe/game ID: `10741900895`
- Place version observed: `15`
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
- at `10,000 kg`, a New Diet resets the run while retaining permanent progress and increasing metabolism.

The experience is intentionally loud and comedic: chunky UI, saturated colors, restaurant-scale character growth, milestone celebrations, and physical table-based eating.

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

- **Server:** live player state, Calories, weight, tray contents, upgrades, RNG, bag rolls, rewards, rebirth eligibility, persistence, milestone detection, character size application, and station prompt validation.
- **Shared:** pure economy rules, exact action-shape validation, the deterministic meal state machine, balance/config data, progression data, and number formatting.
- **Client:** input, local presentation, smooth Calories prediction between authoritative syncs, UI construction, drag/drop, local bite animation for a server-reserved meal, collection-board rendering, camera framing, sounds, and effects.
- **Workspace/builders:** baked restaurant and station geometry. Destructive full rebuilds remain explicit; additive boot migrations create only missing TODO-era world pieces.

The client never sends resulting currency, weight, tiers, rolls, or upgrade levels. It sends only named intent plus minimal identifiers or indices.

## Runtime lifecycle

### Server startup

`ServerScriptService.Server.Main` performs this sequence:

1. Ensures `ReplicatedStorage.Remotes` exists.
2. Ensures three `RemoteEvent` instances exist: `Action`, `Sync`, and `Notify`.
3. Runs `Balance.validate()` and refuses to start if an economy invariant fails.
4. Resolves whether DataStore persistence is available and warns when Studio uses memory fallback.
5. Runs additive `MapBuilder.ensureTodoWorld()` and `StationsBuilder.ensureTodoStations()` migrations. They add missing Drive-Thru/leaderboard/scale pieces without destroying existing roots.
6. Starts `PlayerService` and `CharacterService`.
7. Starts `StationService` and `LeaderboardService`.

`Workspace.Restaurant` is already baked. The destructive `MapBuilder.build()` path is not called on normal boot.

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
| `BuyUpgrade` | `id: string` | Validate known upgrade, level cap, and cost. |
| `ToggleAutoMerge` | none | Flip the saved toggle and merge eligible full groups when enabled. |
| `Merge` | `slotA: number, slotB: number` | Validate two occupied, distinct, matching slots and merge result. |
| `BeginEat` | `tier: number` | Reserve one owned tier and return deterministic bite count/duration; food remains owned. |
| `CompleteEat` | none | Consume the reserved tier only after its server ready time. Early, replayed, or stale completion cancels without reward. |
| `CancelEat` | none | Clear the active reservation without consuming food. |
| `OpenPending` | `bagId: string` | Open one owed bag for free if the tray has room; server rolls it. |
| `Rebirth` | none | Validate `kg >= 10,000`, then reset the run. |
| `Debug` | `what: string, value: number` | Studio-only: `setKg` or `addCalories`; rejected outside Studio. |

Every action first passes `Shared.ActionSpec`: exact argument count, bounded known IDs, finite integer slots/tiers, and Studio-only bounded debug values. Legacy one-shot `Eat` is rejected. The server uses a token bucket of `25` units per second; `FillTray` costs `5` units and has a `0.25` second cooldown. Economy functions re-check state preconditions.

### `Sync`: server to client

Authoritative snapshot fields:

- `kg`, `peakKg`, `lifetimeKg`, `diets`
- `calories`, `calPerSec`, `digestCalPerSec`
- `tray` (cloned array of food tier numbers), `slots`
- `upgrades`, `upgradeCosts`
- `autoMergeEnabled`, `autoMergeTier`
- `eatBites`, `sizeStage`, `multiplier`
- `canRebirth`, `nextMilestone`
- `pendingBags`
- `discoveredFoods` (cloned map keyed by stable `tier_N` food IDs)

Do not remove or rename these without updating every client consumer and this document.

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
| `Meal` | `{ status, tier?, bites?, duration?, gained?, kg?, reason? }` | Start, complete, or cancel the client plate/hold presentation. |
| `Discovery` | `{ tier, id }` | Reveal and celebrate the collection cell. |
| `Rejected` | `{ action, reason }` | Explicit feedback for a well-shaped action that fails state validation. |

### Mirrored `Player` attributes

For read-only visibility outside the server VM:

- `Kg`
- `PeakKg`
- `Diets`
- `CalPerSec`
- `SizeStage`
- `ProfileLoaded`

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
| `version` | Schema version, currently `2`. |
| `peakKg` | Highest lifetime weight; permanent source for milestone rewards and bag unlocks. |
| `diets` | Completed New Diet count; permanent. |
| `lifetimeKg` | Total weight gained; permanent and shown on the record board. |
| `kg` | Current-run weight; resets to `60`. |
| `calories` | Current spendable currency; resets on New Diet. |
| `tray` | Ordered array of tier numbers; resets to starting tray. |
| `peakCalPerSecRun` | Current-run peak tray income used by the recovery floor. |
| `digestion` | Current-run raw, unmultiplied Cal/s retained from eaten food; resets on New Diet. |
| `incomeMultiplier` | Monetization hook, default `1`. |
| `bonusSlots` | Monetization hook, default `0`. |
| `offlineCapBonusHours` | Monetization hook, default `0`. |
| `upgrades` | Map of upgrade ID to level; permanent. |
| `autoMergeEnabled` | Saved toggle; defaults true. |
| `pendingBags` | Map of bag ID to owed count from offline/daily rewards. |
| `discoveredFoods` | Permanent map of known stable `tier_N` food IDs to `true`; awarded only by a successful eat and preserved by New Diet. |
| `lastSeen` | Unix time for offline calculation. |
| `streak` | Daily streak length. |
| `lastClaimDay` | Integer UTC-like day bucket (`time // 86400`). |

Migration is additive: fields missing from an older save are copied from a fresh default state. Discovery keys are sanitized against the current food ladder before `version` is set to the current version.

Studio persistence behavior:

- `GetDataStore` and API-access failures downgrade the session to an in-memory table.
- Do not enable Studio access to live production data for routine tests.
- Live servers preserve strict load failure behavior rather than granting a fresh profile.

## Economy rules and invariants

### Starting state and income

- Starting weight: `60 kg`.
- Starting Calories: `300`.
- Base tray slots: `6`.
- Starting tray: `{ 1, 1, 1, 2 }`, intentionally containing one full merge group.
- Base recovery income: `15 Cal/s`.
- Recovery floor: `max(15, peak tray Cal/s this run * 0.10)` applied to the tray component only.
- Total income: `max(current tray rate, recovery floor) + digestion rate`.

Multiplier:

```text
milestone metabolism multiplier
* 1.75 ^ diets
* (1 + 0.10 * metabolism-upgrade level)
* incomeMultiplier
```

The same multiplier scales tray Cal/s, digestion Cal/s, and kilograms gained per eat.

### Food tiers

There are `12` food tiers:

1. Small Fries
2. Cheeseburger
3. 6pc Nuggets
4. Chili Dog
5. Bucket Soda
6. Family Pizza
7. Combo Platter
8. Slab of Beef
9. Popcorn Vat
10. The Whole Menu
11. Deep-Fried Everything
12. The McMonstrosity

Generated values:

- tier Cal/s: `1.65 ^ (tier - 1)`;
- tier eat weight: `2.75 * 1.275 ^ (tier - 1)` kg before multipliers;
- preferred merge shape: `3 -> 2`, real merge cost `1.5` input items per output;
- fallback manual merge: `2 -> 1`;
- tier 12 cannot merge.

Key invariants:

- `KgRatio (1.275) < MergeCost (1.5)`, so direct eating beats merging first on weight.
- `CalRatio (1.65) > MergeCost (1.5)`, so a full three-item merge improves income.
- A two-item fallback merge loses some income in exchange for a slot and higher tier.
- Automated/bulk merging only consumes full three-item groups; it never chooses the value-negative pair merge for the player.
- Payback drift at the top tier must remain at least `0.25` of tier-one payback.

### Eating and digestion

- Eating removes one owned tier from the tray and grants its scaled weight.
- `50%` of the food's raw Cal/s is moved into current-run digestion.
- Digestion is stored unmultiplied so later milestone, diet, or upgrade multipliers scale it consistently.
- `EatCalShare` must remain below `1`; at `1` eating would become income-neutral and destroy the merge decision.
- Physical eating is client-presented as plating followed by repeated bite clicks, but the server sees only the final `Eat(tier)` intent.
- Bite count falls with weight and is clamped from `5` down to `1`; the formula uses `EatBitesPerDecade = 1.8` from the `60 kg` baseline.
- A disconnect during a partially eaten client-side plate loses nothing because the server item remains in the tray until the final bite.

### Bags

| ID | Name | Cost | Unlock | Normal tier distribution | Jackpot |
|---|---|---:|---:|---|---:|
| `snack` | Snack Bag | 100 | 0 kg | T1 70%, T2 25%, T3 5% | 0.1% one tier above normal max |
| `big` | Big Bag | 870 | 110 kg | T2 55%, T3 30%, T4 12%, T5 3% | 0.1% |
| `mega` | Mega Bag | 11,250 | 300 kg | T4 45%, T5 30%, T6 18%, T7 7% | 0.1% |
| `feast` | Feast Bag | 165,000 | 1,000 kg | T6 40%, T7 28%, T8 18%, T9 10%, T10 4% | 0.1% |
| `banquet` | Banquet Bag | 1,440,000 | 3,500 kg | T8 40%, T9 30%, T10 20%, T11 10% | 0.1% |
| `monstrosity` | Monstrosity Bag | 10,000,000 | 6,000 kg | T9 35%, T10 30%, T11 25%, T12 10% | 0.1% |

Bag design invariant:

- cheaper bags have better expected Cal/s per Calorie spent;
- pricier bags have better expected Cal/s per tray slot/click.

Bag unlocks use `peakKg`, so they survive a New Diet.

### Upgrades

All upgrade levels are permanent. Cost at the next level is `floor(baseCost * costMultiplier ^ currentLevel)`.

| ID | Effect | Base cost | Multiplier | Max |
|---|---|---:|---:|---:|
| `automerge` | Automatically merge full groups up through the purchased tier. | 20,000 | 6 | 9 |
| `slots` | +1 tray slot per level. | 150,000 | 8 | 5 |
| `fridge` | +1 hour offline cap per level. | 80,000 | 5 | 8 |
| `metabolism` | +10% Calories and weight per bite per level. | 400,000 | 7 | 10 |

### Milestones and character stages

Milestone rewards derive only from permanent `peakKg`. Visual size stage derives from current `kg`, so rebirth shrinks the character.

| kg | Reward/label |
|---:|---|
| 75 | +1 slot — Shirt's getting tight |
| 110 | +10% metabolism and Big Bag unlock |
| 175 | +1 slot — Trousers have given up |
| 300 | +1 slot and Mega Bag unlock |
| 550 | +1 slot and 10-hour offline cap |
| 750 | +1 slot — Two seats, one you |
| 1,000 | +25% metabolism and Feast Bag unlock |
| 2,000 | +2 slots — Booth rebuilt around you |
| 3,500 | +1 slot, 12-hour offline cap, and Banquet Bag unlock |
| 6,000 | +2 slots and Monstrosity Bag unlock |
| 8,000 | +50% metabolism — Half the restaurant |
| 10,000 | Rebirth threshold — You've outgrown the restaurant |

Size thresholds are `60, 75, 110, 175, 300, 550, 750, 1000, 2000, 3500, 6000, 8000, 10000` kg.

`CharacterService` uses matching scale arrays:

- overall scale: `1.0, 1.12, 1.28, 1.45, 1.7, 2.0, 2.2, 2.5, 3.0, 3.7, 4.7, 6.0, 8.0`;
- girth multiplier: `1.0, 1.12, 1.25, 1.38, 1.5, 1.62, 1.72, 1.82, 1.92, 2.02, 2.14, 2.24, 2.35`.

At stage `10` (`3,500 kg`) and above, movement/jumping is disabled and the root is anchored because the character is too large for the room. Earlier stages restore normal movement. R15 humanoid scale values produce extra width/depth; R6 falls back to uniform `ScaleTo`.

### Offline and daily rewards

- Offline earnings grant Calories only, never weight.
- Efficiency: `75%` of current Cal/s.
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

### New Diet / rebirth

Requirement: current `kg >= 10,000`.

Resets:

- current `kg` to `60`;
- Calories to `0`;
- tray to `{ 1, 1, 1, 2 }`;
- current-run peak Cal/s to `0`;
- digestion to `0`.

Preserves:

- `peakKg`, `lifetimeKg`, and completed diet count;
- milestone rewards and bag unlocks derived from peak weight;
- permanent upgrades and monetization-hook fields;
- pending bags and daily metadata.

Each completed diet multiplies metabolism by `1.75`.

## World and presentation

### Place-level settings observed

- `Workspace.StreamingEnabled = true`.
- Gravity approximately `196.2`.
- Fallen-parts destroy height: `-500`.
- Camera mode: Classic.
- Character appearance loading enabled.
- Mouse lock option enabled.
- Lighting: ClockTime `14`, Brightness `1`, warm ambient/outdoor ambient.
- Lighting effects: `Sky`, `SunRays`, `Atmosphere`, `Bloom`, and `DepthOfField`.
- Sound filtering is respected; ambient reverb is NoReverb.

### Restaurant

`ServerStorage.MapBuilder` owns the procedural placeholder restaurant. It is intended to be baked once and then hand-edited. Calling `MapBuilder.build()` destroys and recreates `Workspace.Restaurant`, removes default base parts, and clears Terrain, so it is destructive and must be deliberate.

The room constant is `140` studs and is mirrored in `Shared.Config.Stations`; changes must keep both places aligned.

`Workspace.Restaurant` contains:

- `Floor`: tiled floor;
- `Walls`: walls, trim, windows, ceiling;
- `Counter`: counter base/top and menu board;
- `Booth`: benches, table, table tray, invisible `DropZone`, and `SeatAnchor`;
- `Lights`: nine neon panels with point lights;
- `BoothSpawn`: spawn positioned clear of the booth and facing it.
- `DriveThruArea`: Diet 1-gated asphalt lane, fences, signs, and a collection/server-top-five purpose area inside the existing room.

The physical booth tray mirrors owned food client-side. `DropZone` is the raycast target that makes drag-to-eat usable even when the thin tray is hard to see.

### Stations

`ServerStorage.StationsBuilder` bakes `Workspace.Stations`. NPC art is deliberately block-built and replaceable. Each station model and prompt uses a `StationId` attribute.

| Station ID | Presentation | Opens |
|---|---|---|
| `cashier` | Counter NPC and till | Bag shop |
| `frycook` | Counter NPC and fryer | Upgrades |
| `nutritionist` | Left-wall NPC and scale | Rebirth nudge/action |
| `drivethru` | Right-wall NPC and pickup window | Daily/pending-bag nudge |

`RecordBoard` shows the local player's `lifetimeKg` through a SurfaceGui. `ScaleDial.ScaleDisplay` shows current weight locally. `SizeLeaderboard` shows a server-local, sanitized, deterministic top five of loaded players and coalesces writes to at most once per two seconds.

The left navigation rail remains available even though world stations exist, because very large/anchored players cannot reliably walk to them.

### UI and input

The client creates `McFattyUI` under `PlayerGui`; `StarterGui` itself is empty. The UI is authored around `1280x720`, uses a `0.72` narrow-screen floor, changes the tray to five columns on narrow screens, and lifts the tray above the mobile safe area.

Primary UI:

- three top HUD groups: weight/size stage, Calories/rate/digestion, and Diets;
- a slim next-bag unlock marker;
- visible next-milestone progress card;
- bottom tray grid with persistent Fill Tray and Auto-Merge actions;
- right-side mutually exclusive bag shop and upgrades panel;
- left navigation rail for Shop, Upgrades, Rebirth, and Daily;
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

The counts below describe the Edit DataModel after script additions. The additive Drive-Thru, scale-display, and leaderboard instances are created by boot migration in older baked places and therefore appear in Play even when they are not yet baked into the Edit hierarchy.

### Root counts

| Root | Direct children | Descendants |
|---|---:|---:|
| `Workspace` | 4 | 163 |
| `ReplicatedStorage` | 2 | 13 |
| `ServerScriptService` | 1 | 8 |
| `ServerStorage` | 4 | 4 |
| `StarterPlayer` | 2 | 21 |
| `StarterGui` | 0 | 0 |
| `StarterPack` | 0 | 0 |
| `Lighting` | 5 | 5 |
| `SoundService` | 0 | 0 |
| `Teams` | 0 | 0 |

Top-level game structure:

```text
Workspace
├── Camera
├── Stations
├── Terrain
└── Restaurant

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
│       └── Stations
└── Remotes

ServerScriptService
└── Server
    ├── Main
    └── Services
        ├── DataService
        ├── PlayerService
        ├── CharacterService
        ├── StationService
        └── LeaderboardService

ServerStorage
├── PacingSim
├── EconomyTests
├── MapBuilder
└── StationsBuilder

StarterPlayer
├── StarterPlayerScripts
│   └── Client
│       ├── Main
│       ├── CameraController
│       ├── WorldEffects
│       ├── TableFood
│       ├── CollectionBoard
│       └── UI
│           ├── Theme
│           ├── Tray
│           ├── HUD
│           ├── Shop
│           ├── Panels
│           ├── Tutorial
│           ├── Pending
│           ├── UpgradesPanel
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
- `game.ReplicatedStorage.Shared.Config.Balance` — global tuning and boot-time invariant validator.
- `game.ReplicatedStorage.Shared.Config.Upgrades` — permanent upgrade definitions and cost curve.
- `game.ReplicatedStorage.Shared.Config.Stations` — world station definitions and UI destinations.

Server:

- `game.ServerScriptService.Server.Main` — server entry point and remote creation.
- `game.ServerScriptService.Server.Services.DataService` — session-locked DataStore or memory fallback.
- `game.ServerScriptService.Server.Services.PlayerService` — session ownership, actions, accrual, sync, autosave, progress events.
- `game.ServerScriptService.Server.Services.CharacterService` — stage-to-character rendering and movement lock.
- `game.ServerScriptService.Server.Services.StationService` — validated ProximityPrompt to client station notification.
- `game.ServerScriptService.Server.Services.LeaderboardService` — server-local loaded-player top-five board with sanitized labels and coalesced refresh.

ServerStorage tools:

- `game.ServerStorage.PacingSim` — headless engaged/casual pacing model using shipped economy rules.
- `game.ServerStorage.EconomyTests` — fresh-clone headless regression checks.
- `game.ServerStorage.MapBuilder` — destructive restaurant rebuild tool plus additive TODO-world migration.
- `game.ServerStorage.StationsBuilder` — destructive station rebuild tool plus additive prompt/scale/leaderboard migration.

Client:

- `game.StarterPlayer.StarterPlayerScripts.Client.Main` — client entry point, UI composition, remote wiring, action dispatch.
- `game.StarterPlayer.StarterPlayerScripts.Client.CameraController` — camera framing for growing characters.
- `game.StarterPlayer.StarterPlayerScripts.Client.WorldEffects` — character/world particle and ring effects.
- `game.StarterPlayer.StarterPlayerScripts.Client.TableFood` — table mirror, plate, bite tracking, drag-to-world target.
- `game.StarterPlayer.StarterPlayerScripts.Client.CollectionBoard` — per-player menu collection, scale display, and Diet-gate presentation.
- `game.StarterPlayer.StarterPlayerScripts.Client.UI.Theme` — visual tokens and widget constructors.
- `game.StarterPlayer.StarterPlayerScripts.Client.UI.Tray` — tray rendering and drag/merge gesture.
- `game.StarterPlayer.StarterPlayerScripts.Client.UI.HUD` — stats, local Calories prediction, milestone progress.
- `game.StarterPlayer.StarterPlayerScripts.Client.UI.Shop` — bag purchase/fill panel.
- `game.StarterPlayer.StarterPlayerScripts.Client.UI.Panels` — queued toasts and daily/offline/rebirth modals.
- `game.StarterPlayer.StarterPlayerScripts.Client.UI.Tutorial` — inferred first-run tutorial.
- `game.StarterPlayer.StarterPlayerScripts.Client.UI.Pending` — free pending bags panel.
- `game.StarterPlayer.StarterPlayerScripts.Client.UI.UpgradesPanel` — permanent upgrade panel.
- `game.StarterPlayer.StarterPlayerScripts.Client.UI.Rail` — left navigation and pending-reward pip.
- `game.StarterPlayer.StarterPlayerScripts.Client.UI.RebirthCard` — always-visible New Diet goal/action.
- `game.StarterPlayer.StarterPlayerScripts.Client.UI.Sounds` — pooled engine-built-in UI audio.
- `game.StarterPlayer.StarterPlayerScripts.Client.UI.Effects` — screen pop/float/confetti/flash/shake effects.

## Testing and development tools

### Balance validator

The server runs `Balance.validate()` on boot. It checks bag ordering/value, merge/eat ratios, digestion share, and end-tier payback drift. Economy/config work is incomplete if this fails.

### Economy tests

`ServerStorage.EconomyTests` covers:

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
- meal early completion, success, replay, and stale ownership;
- discovery award, migration sanitization, and New Diet preservation;
- balance validation.

Run it through MCP in Edit mode. It clones Shared to avoid ModuleScript cache staleness. Clean up `ServerStorage.__TestSandbox` after the run if it remains.

### Pacing simulator

`ServerStorage.PacingSim` drives the shipped `Shared.Economy` rules and models a finite action budget, bounded fill-tray purchasing, per-group merges, Auto-Merge, and deterministic hold duration (`eatBites / 4`) rather than fictional bite-button presses.

Use a fresh clone after changing config or economy code because Studio memoizes ModuleScripts. The documented engaged baseline policy is:

```lua
{ name = "engaged", buy = "richest", keepFraction = 0.6, actionsPerSecond = 1.5, upgrades = "smart" }
```

Do not claim pacing numbers from comments as current measurements; run the simulator after relevant changes.

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

Latest observed Play verification on 2026-08-20:

- server boot reached `[McFatty] server ready`, which includes a successful `Balance.validate()`;
- additive migration created `DriveThruArea.Diet1Gate`, `ScaleDial.ScaleDisplay`, and `SizeLeaderboard`;
- a real Fill action filled the tray and advanced tutorial Step 1 only after the authoritative snapshot;
- drag-to-table began a five-bite meal, a timed hold completed it, weight changed from `60` to `62.75 kg`, and the plate cleared after sync;
- the collection revealed Small Fries, the scale displayed live kg, and leaderboard Row 1 updated;
- no project runtime error/warning remained; the expected Studio DataStore API-off fallback warning and a Roblox Assistant-plugin version warning were present.

## Known architectural constraints and change-sensitive couplings

- Studio is currently the only implementation source. Local Luau edits do not affect the game.
- `Remotes` are runtime-created; do not infer a missing contract from an empty Edit-mode folder.
- The room size/counter reference is mirrored between `MapBuilder` and `Config.Stations` because the client cannot require ServerStorage.
- `Progression.SizeStages`, `CharacterService.SCALE`, and `CharacterService.GIRTH` must have identical lengths.
- Food ratios, bag prices, digestion share, bite counts, and action assumptions jointly determine pacing; change them as a system and re-run the simulator.
- Player actions use tray indices for merge but tier identity for `BeginEat`. The reservation must still own that tier at completion.
- Bulk and Auto-Merge intentionally ignore pairs even though manual pair merging is allowed.
- Meal reservation/timing is server-owned while bite visuals are client-only. Food is not removed until timely completion; changing this handshake affects networking, replay safety, disconnects, and tutorial confirmation.
- UI drag hit-testing relies on viewport-space coordinates and unscaled overlay ScreenGuis. Mixing them with inset-inclusive mouse coordinates breaks drops.
- The game is designed for StreamingEnabled; world-facing client modules must tolerate important instances being absent or late.
- `MapBuilder.build()` and `StationsBuilder.build()` destroy/recreate their named world roots. Inspect and confirm the exact target before running them.
- `ensureTodoWorld()` and `ensureTodoStations()` are additive boot migrations and must remain idempotent; do not turn them into hidden destructive rebuilds.
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
