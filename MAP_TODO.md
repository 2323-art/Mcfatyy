# McFatty dynamic restaurant district — implementation prompt

Use this file as the complete prompt for a future implementation task. It records the requirements agreed with the user on 2026-08-21. Roblox Studio is the authoritative implementation source; this file is a TODO, not a claim that the work has shipped.

## Mission

Replace the current compact twenty-restaurant grid with a functional, vehicle-ready district containing twenty permanent player lots in a four-row-by-five-column layout. Three parallel internal roads must separate the four restaurant rows, and connector roads at both ends must join them into one continuous ladder-shaped road network.

Unclaimed plots must be empty grey-asphalt lots. When a player is assigned a plot, the server must place an unchanged copy of the current restaurant in that lot and update a roadside sign with the player's username. When the player leaves, the restaurant must be removed immediately and the lot must return to its unclaimed state.

Do not redesign the restaurant, add doors, implement additional floors, or add restaurant-size progression in this task.

## Mandatory project startup

- [ ] Read `AGENTS.md` and `GAME_CONTEXT.md` completely.
- [ ] List connected Roblox Studio instances and select only **McFatty**, place ID `74686069419969`. If the target is ambiguous, stop and ask before changing anything.
- [ ] Return Studio to Edit mode for authoritative inspection.
- [ ] Inspect the required service roots and enumerate every `Script`, `LocalScript`, and `ModuleScript` as required by `AGENTS.md`.
- [ ] Compare Studio with the `GAME_CONTEXT.md` structure snapshot. Refresh the context before implementation if relevant drift exists.
- [ ] Read the complete source of every affected script before editing it.
- [ ] Search all scripts for consumers of `Restaurants`, `PlotIndex`, `PlotOrigin`, `BoothSpawn`, `Stations`, `Booth`, `Counter`, `RecordBoard`, `SizeLeaderboard`, and `PlotRoot`. Do not rely only on the known consumers listed below.
- [ ] Write a concise implementation brief and design before changing Studio code.

## Current state that must be accounted for

The following was observed in the authoritative place before this TODO was written:

- `Workspace.Restaurants` contains `Ground` and twenty fully built models named `Plot01` through `Plot20`.
- The current plot origins already form five columns by four rows, but the pitch is only about `136` studs. Each restaurant interior is about `120 × 120` studs, leaving nowhere near enough space for functional roads.
- `ServerStorage.MapBuilder.build()` destructively rebuilds the entire restaurant grid and clears the legacy base/terrain. It currently creates every restaurant up front.
- `ServerStorage.StationsBuilder` currently bakes a `Stations` folder directly under every plot and positions its content by adding unrotated offsets to `PlotOrigin`.
- `PlotService` currently assigns the first free plot, expects `BoothSpawn` and `Stations` directly under the plot, and uses the spawn position as the authoritative eating-area center.
- `Client.PlotRoot.get()` currently returns the plot model. `TableFood` and `CollectionBoard` then look for `Booth` and `Counter` directly under that result.
- `LeaderboardService` currently looks for `plot.Stations.SizeLeaderboard`.
- `StationService` listens globally through `ProximityPromptService`, so dynamically cloned prompts can continue to work as long as their `StationId` attributes and validation remain intact.
- The current restaurant has solid walls on all four sides. There is no physical doorway.
- The place is currently configured for up to `60` players although only twenty plots exist.

Re-inspect all of these facts before implementation; do not treat this TODO as a substitute for Studio.

## Confirmed product and layout requirements

### District topology

- [ ] Create exactly twenty permanent plot containers: four rows with five plots in each row.
- [ ] Preserve internal model names `Plot01` through `Plot20` and numeric `PlotIndex` values for compatibility, but never render those identifiers to players.
- [ ] Keep row-major plot identity stable even if assignment priority changes.
- [ ] Place three parallel two-lane roads between the four plot rows.
- [ ] Connect all three roads at both ends with perimeter connector roads. The network must form continuous loops with no isolated road or dead end.
- [ ] Make the middle road the main boulevard.
- [ ] Orient the two middle restaurant rows toward each other across the main boulevard.
- [ ] Orient each outer restaurant row toward its adjacent outer road.
- [ ] Treat the restaurant's local dining/front side as the road-facing side, but do not alter any restaurant geometry to create an entrance.

### Road and pedestrian infrastructure

- [ ] Make road dimensions vehicle-ready even though vehicles are out of scope. Use approximately `32` studs of asphalt for two travel lanes and approximately `8` studs of sidewalk on each side; tune only when a Studio measurement demonstrates a collision or scale problem.
- [ ] Add clear lane markings, curbs, curb ramps, intersections, and crosswalks.
- [ ] Make sidewalks continuous around the network.
- [ ] Ensure every plot frontage can be reached from every other plot through sidewalks and crossings without jumping.
- [ ] Keep paths wide enough for enlarged characters that are still allowed to move. Stage-10-and-larger characters remain governed by the existing movement lock and UI access.
- [ ] Give every plot one simple parking area connected to the road by one driveway. Do not prescribe or enforce individual parking spaces.
- [ ] End the pedestrian path at the center of the road-facing restaurant wall and keep that frontage apron free of decoration for a future entrance.
- [ ] Do not claim that players can walk from the street into the restaurant: doors and wall openings are explicitly deferred, and players continue to spawn inside.

### Lots and occupancy

- [ ] Every unclaimed lot must be one flat grey-asphalt surface.
- [ ] Do not add grass, a distinct foundation pad, construction props, or decorative clutter inside an unclaimed lot.
- [ ] The driveway/parking treatment, sidewalk connection, reserved frontage path, and owner sign remain present when a lot is empty.
- [ ] An unclaimed plot must not contain a restaurant model.
- [ ] An occupied plot must contain exactly one `Restaurant` model using the current restaurant design unchanged.
- [ ] The current floor, walls, trim, windows, ceiling, counter, booth, tray/drop zone, spawn, lights, stations, boards, colors, materials, dimensions, and interior relationships must be preserved. Refactoring their coordinates into a reusable template is allowed; visual redesign is not.
- [ ] Do not add doors, openings, door frames, door scripts, or animations.
- [ ] On player leave, remove the `Restaurant` immediately and restore the sign to its unclaimed state. Permanent lot and district infrastructure must remain.
- [ ] Restaurant occupancy and appearance are server-session state only. Do not persist plot identity or restaurant state.

### Owner sign

- [ ] Add one permanent freestanding sign beside each driveway entrance, outside vehicle and pedestrian clearance zones.
- [ ] Make the text readable from both sides of the sign.
- [ ] Show `AVAILABLE` on an unclaimed plot.
- [ ] On assignment, use the account username from `Player.Name`, not `DisplayName`.
- [ ] Preserve username casing.
- [ ] If the username is longer than ten characters, use only its first ten characters.
- [ ] Format the occupied text exactly as `<truncated username>'s Restaurant`.
- [ ] Update sign text authoritatively on the server so every client sees the same owner.
- [ ] Do not display `Plot01`, plot indexes, user IDs, or other internal identifiers.
- [ ] Fit long allowed text cleanly without overflow or unreadably small type.

### Assignment order and capacity

- [ ] Keep assignment non-yielding between selecting and claiming a free plot.
- [ ] Fill plots nearest the central boulevard first, then expand outward.
- [ ] Within equal-distance groups, use a documented deterministic center-out order so assignment is stable and testable.
- [ ] Do not change the twenty-plot capacity, share plots, or generate overflow lots.
- [ ] Retain the existing safe rejection behavior if no plot is available.
- [ ] Manually set the place's maximum player count to `20` in Game Settings. Verify the setting after saving; this cannot be fixed by assigning `Players.MaxPlayers` at runtime.

### Visual direction and boundary

- [ ] Use a bright, chunky, low-poly fast-food suburb style that matches McFatty's saturated presentation.
- [ ] Build the district from anchored Roblox Parts and built-in materials. Do not use Terrain or import external assets.
- [ ] Add a landscaped green buffer around the outside of the district, with a clean visible boundary such as low fencing or hedges and an unobtrusive collision backstop.
- [ ] Leave deliberate expansion space beyond the visible boundary.
- [ ] Use a small repeated set of streetlights, trees, hedges, markings, and signs. Favor silhouettes and readability over dense decoration.
- [ ] Do not add per-object scripts or continuous polling for scenery.
- [ ] Preserve the current Lighting direction unless a small adjustment is needed to make the new outdoor map readable.

## Required hierarchy and lifecycle boundary

Use this conceptual hierarchy unless inspection proves that a closely equivalent structure is materially safer:

```text
Workspace
└── Restaurants
    ├── RoadNetwork
    ├── Boundary
    └── Plot01 ... Plot20
        ├── Lot
        ├── OwnerSign
        └── Restaurant (present only while claimed)
            ├── Floor
            ├── Walls
            ├── Counter
            ├── Booth
            ├── Lights
            ├── BoothSpawn
            └── Stations
```

Implementation requirements:

- [ ] Keep permanent road, sidewalk, parking, lot surface, boundary, and sign instances outside the optional `Restaurant` model.
- [ ] Make the current restaurant a reusable authoritative template, preferably a `ServerStorage.RestaurantTemplate` model generated/baked from the existing builder and cloned by the server.
- [ ] Give every plot one authoritative local-to-world transform, preferably a `PlotCFrame` `CFrame` attribute, and keep `PlotOrigin` compatible where existing consumers still require it.
- [ ] Use the transform for restaurant placement, spawn orientation, NPC facing, props, SurfaceGui-facing parts, and boards. Do not rotate only the shell while leaving interior/station offsets in world axes.
- [ ] Make the restaurant template pivot explicit and stable. Cloning and pivoting it must preserve all interior relative transforms.
- [ ] Keep `PlotIndex` replicated and stable.
- [ ] Treat a missing `Restaurant` child as normal for an unclaimed or not-yet-streamed plot.

## Server ownership and lifecycle

- [ ] Keep plot selection, ownership, restaurant creation/removal, sign text, spawn selection, board ownership, and eating-area resolution server-authoritative.
- [ ] Populate the `Restaurant` synchronously before setting `Player.PlotIndex`, `RespawnLocation`, or moving the character.
- [ ] If restaurant creation fails, roll back the claim, reset the sign, clean partial content, log a useful error, and do not place the player into a broken plot.
- [ ] On successful assignment, resolve `BoothSpawn`, stations, record board, scale dial, and leaderboard under `plot.Restaurant`.
- [ ] On release, clear ownership maps and player references, remove only the optional restaurant, and reset the permanent sign to `AVAILABLE`.
- [ ] Preserve the existing race-safe join behavior, pending-assignment retry, respawn placement, owner board refresh, capacity rejection, and eating-area injection.
- [ ] Do not introduce any new remote, saved-data field, currency, progression state, or client-authoritative decision.

## Known consumers that must be adapted and verified

At minimum, inspect and update these Studio sources as needed:

- [ ] `game.ServerStorage.MapBuilder` — bake the new permanent district and reusable current restaurant template; keep destructive rebuild explicit.
- [ ] `game.ServerStorage.StationsBuilder` — build stations in template-local coordinates or otherwise support rotated/cloned restaurants without changing their visual design.
- [ ] `game.ServerScriptService.Server.Services.PlotService` — central-first assignment priority, dynamic population/cleanup, signs, nested restaurant lookup, spawn/eating/board lifecycle, and rollback.
- [ ] `game.ServerScriptService.Server.Services.LeaderboardService` — discover `SizeLeaderboard` below the optional `Restaurant` and include newly populated restaurants on refresh.
- [ ] `game.ServerScriptService.Server.Services.StationService` — confirm global prompt handling works for dynamically cloned stations; retain server validation and narrow notifications.
- [ ] `game.StarterPlayer.StarterPlayerScripts.Client.PlotRoot` — return the local player's optional/streamed `Restaurant` model so existing world-facing callers remain narrow and nil-tolerant.
- [ ] `game.StarterPlayer.StarterPlayerScripts.Client.TableFood` — verify booth/tray/drop-zone lookup and rotated CFrames.
- [ ] `game.StarterPlayer.StarterPlayerScripts.Client.CollectionBoard` — verify counter/menu-board lookup after dynamic creation and streaming.
- [ ] `game.ServerScriptService.Server.Main` — ensure infrastructure/template initialization happens before plot assignment without generating all twenty restaurants.

Search may identify additional consumers. Update every actual dependency, not just this list.

## Builder behavior

- [ ] Refactor the builder before rebuilding the world so the Studio result remains reproducible.
- [ ] Keep a clearly named destructive full-build entry point for deliberate Edit-mode use.
- [ ] Make normal server startup idempotent and non-destructive. It may add missing permanent infrastructure, but it must not wipe a valid baked district or an occupied restaurant.
- [ ] Never run the destructive builder against an ambiguous Studio instance.
- [ ] Before the destructive rebuild, inspect and confirm the exact `Workspace.Restaurants`, legacy base, and Terrain targets.
- [ ] Save the selected McFatty place before the broad rebuild and after every internally consistent checkpoint.
- [ ] Use **Save to Roblox**, never **Save to Roblox As**, and do not publish.

## Explicit non-goals

- Additional or accessible restaurant floors.
- `RestaurantLevel`, expansion purchases, or any restaurant-size progression.
- Exterior redesign, doors, wall openings, or entrance animation.
- Vehicles, vehicle spawning, traffic AI, traffic lights, or driving gameplay.
- Parking mechanics or a fixed number of marked parking stalls.
- Shared hubs, parks, communal buildings, or new gameplay landmarks.
- Dynamic roadside plot numbers.
- Persistent plot assignment, restaurant customization, or restaurant save fields.
- Economy, bag, upgrade, eating, rebirth, or UI redesign.
- New remotes or changes to existing remote payloads.
- External assets, Terrain, or production publishing.

## Acceptance criteria

- [ ] Edit mode contains one baked district with exactly twenty permanent plot containers in four rows of five.
- [ ] Three parallel internal roads and two end connectors form one continuous ladder-shaped network.
- [ ] All plots have a driveway, one simple parking area, continuous sidewalk access, and a double-sided roadside sign.
- [ ] An unclaimed plot is visibly empty grey asphalt and reads `AVAILABLE`.
- [ ] No internal plot identifier is visible anywhere in the world.
- [ ] The first joining player receives a deterministic central-boulevard plot.
- [ ] Each subsequent player receives a different free plot according to the documented central-first order.
- [ ] Assignment creates exactly one nested `Restaurant` that visually matches the current restaurant.
- [ ] All rotated restaurants keep correct interior geometry, station positions/facing, board faces, booth spawn facing, and drop-zone alignment.
- [ ] A short username and a username longer than ten characters both produce the exact required sign text.
- [ ] Leaving removes the restaurant immediately and resets the same sign to `AVAILABLE` without damaging lot or road infrastructure.
- [ ] Respawn still places a player at their own `BoothSpawn`.
- [ ] Eating-area validation still resolves only the player's own restaurant.
- [ ] Shop/upgrades/rebirth station prompts still open the intended UI.
- [ ] Table-food mirroring, drag-to-eat, collection board rendering, record board, scale dial, and server top-five board still work.
- [ ] Missing, late, or streamed-out `Restaurant` content is tolerated without infinite waits or client errors.
- [ ] Sidewalk routes connect every frontage without requiring a jump, and curbs/decoration do not block crossings or driveways.
- [ ] A full server never shares plots; overflow retains the existing safe rejection path.
- [ ] Game Settings reports a maximum of twenty players.
- [ ] The server reaches `[McFatty] server ready` with no new project runtime error.

## Verification checklist

- [ ] Re-read every changed script after MCP edits and confirm the applied source.
- [ ] Inspect the final Edit hierarchy, attributes, plot transforms, instance counts, and exact script inventory.
- [ ] Add or run focused Edit-mode validation for:
  - exactly twenty unique `PlotIndex` values;
  - stable row/column placement and central-first priority;
  - expected optional-child hierarchy;
  - no restaurant in unclaimed baked plots;
  - connected road bounds and non-overlap with restaurant/parking footprints;
  - valid signs, transforms, pivots, spawns, stations, boards, trays, and drop zones.
- [ ] Run Studio Play verification for one player: central assignment, restaurant creation, sign, spawn, station prompts, collection board, table food, eating, and respawn.
- [ ] Exercise join/leave/rejoin and confirm immediate cleanup and safe reuse of the same lot.
- [ ] Exercise multiple players and confirm distinct ownership, central-first allocation, public sign/board replication, and no cross-plot eating.
- [ ] Exercise representative restaurants from every facing direction, not only one row.
- [ ] Exercise a missing or temporarily streamed-out `Restaurant`, `Booth`, `Stations`, and `BoothSpawn` where relevant.
- [ ] Inspect server and client console output and report what was actually observed.
- [ ] Record the final anchored-part/descendant count and confirm the streetscape adds no per-frame world loop or per-object script explosion.
- [ ] Economy tests and pacing simulation are not required unless implementation unexpectedly changes economy/progression code. The boot balance validator must still pass.
- [ ] Save to Roblox after verification succeeds. Do not publish.

## Documentation and handoff

- [ ] Update `GAME_CONTEXT.md` in the same task because the Workspace hierarchy, map builder, plot lifecycle, assignment order, optional restaurant structure, sign presentation, startup data flow, script responsibilities, and known couplings will change.
- [ ] Update its verification date, structure snapshot/counts, restaurant-plot documentation, lifecycle, script inventory/responsibilities, runtime verification, and risks.
- [ ] In the final handoff, list every Studio script/instance changed, why `GAME_CONTEXT.md` changed, all verification performed and its result, the manual MaxPlayers result, anything not verified, and remaining risks.

