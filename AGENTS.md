# McFatty project initialization

This workspace controls a Studio-first Roblox experience through the Roblox Studio MCP connection. Roblox Studio is the authoritative source for the game. The local workspace is used for project guidance and context unless the user explicitly changes that arrangement.

## Required startup sequence

Before investigating, designing, or changing the game:

1. Read `GAME_CONTEXT.md` completely.
2. Use the Roblox Studio MCP connection to list the connected Studio instances.
3. Select the McFatty place with place ID `74686069419969`. If several plausible Studio instances are open, ask the user which one to modify before making any change.
4. Get the Studio state. Prefer the `Edit` DataModel for inspection and editing. Enter Play mode only when runtime verification requires it.
5. Inspect the current structure through MCP and compare it with the **Structure snapshot** in `GAME_CONTEXT.md`. At minimum inspect:
   - `Workspace`
   - `ReplicatedStorage`
   - `ServerScriptService`
   - `ServerStorage`
   - `StarterPlayer`
   - `StarterGui`
   - `StarterPack`
   - `Lighting`
   - `SoundService`
   - `Teams`
6. Enumerate all `Script`, `LocalScript`, and `ModuleScript` instances and compare their exact paths with the script inventory in `GAME_CONTEXT.md`.

Do not assume the structure is unchanged just because a requested feature sounds small.

## Context maintenance

`GAME_CONTEXT.md` is living project documentation and must match Studio.

Update it in the same task, before implementation when practical, whenever MCP inspection shows any of the following:

- an instance, service child, script, module, remote, tag, or major world model was added, removed, moved, or renamed;
- an entry point, module responsibility, ownership boundary, runtime data flow, or lifecycle changed;
- a remote name, action, payload, notification, sync snapshot, player attribute, public module API, or stored-data field changed;
- core gameplay, balance invariants, progression, persistence, UI navigation, input, character growth, map generation, testing, or tooling changed;
- Roblox Studio is no longer the only authoritative implementation source.

After refreshing the file, update its verification date and any affected snapshot, contract, and risk sections. If inspection finds no relevant drift, do not rewrite the context merely to change its date.

## Editing rules

- Perform Roblox game inspection and edits through the Roblox Studio MCP tools. Do not create local Luau files and claim they changed the Studio project.
- Read the complete source of every affected script through MCP before editing it.
- Establish an implementation brief and design before changing implementation code.
- Preserve the current strict Luau, server-authoritative, service/module architecture unless there is a concrete reason to change it.
- Treat every client action as untrusted. The server owns progression, currency, food inventory, rewards, RNG, upgrades, rebirths, and persistence.
- Keep public contracts narrow and typed. Do not silently change remote payloads or save data.
- Use focused MCP edits and re-read changed scripts afterward to confirm the applied source.
- Do not publish, save a new place version, enable production DataStore access in Studio, import assets, or destructively rebuild world content unless that action is explicitly in scope.

## Verification rules

- Run the balance validator after economy or config changes.
- Run `ServerStorage.EconomyTests` after shared economy, progression, bag, upgrade, offline, daily, merge, eat, or rebirth changes.
- Run `ServerStorage.PacingSim` after changing pacing ratios, bag costs or drops, unlock thresholds, starting tray, digestion share, bite counts, upgrade costs, action assumptions, or rebirth tuning.
- Use Studio Play mode for changes involving remotes, client UI, character scaling, camera, prompts, world interaction, respawn, or lifecycle behavior.
- Inspect Studio console output and report what was actually verified. Never claim runtime verification from static inspection alone.
- Exercise malformed/replayed client input, repeated actions, joins/leaves, respawns, full/empty trays, missing world instances, and persistence fallback when relevant.

## Handoff

Every completed game change should identify:

- the Studio scripts/instances changed;
- whether `GAME_CONTEXT.md` changed and why;
- verification performed and its result;
- anything not verified and any remaining risk.
