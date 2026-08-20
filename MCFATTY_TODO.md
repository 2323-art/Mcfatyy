# McFatty's — Player-Experience Review + TODO

Reviewed 2026-08-20 against the live place (placeId 74686069419969) in Studio.
Design rationale: `MCFATTY_DESIGN.md`. Shipped work: `MCFATTY_CHANGELOG.md`.
Actionable checklist derived from this review: `ToDo.md`.

## What this review is, and is not

**I did not play the game.** Nothing below is a report of gameplay. Everything is
inferred from things I actually read or looked at:

- live screen captures of the running client (HUD, tray, panels, the room)
- `Config/Food`, `Config/Bags`, `Config/Progression`, `Config/Balance`,
  `Config/Upgrades`, `Config/Stations`
- `Client/Main`, `UI/Tray`, `UI/Shop`, `UI/Theme`, `UI/Tutorial`, `UI/Effects`,
  `Client/WorldEffects`, `Client/CameraController`, `Server/CharacterService`
- **a fresh PacingSim run executed today** — the milestone timings quoted
  throughout are that run's output, not estimates and not copied from the design
  doc

Anything I could not derive from those is marked
**"This should be verified through playtesting."**

## The measurement everything else hangs off

Fresh PacingSim, seed 12345, three policies. This is the single most useful
artefact in the repo and it is what makes most of the criticism below numeric
rather than vibes.

```
kg      engaged   gap     casual    gap      milestone
75      1:16      1:16    1:16      1:16     Shirt's getting tight
110     2:17      1:01    3:27      2:11     BIG BAG (+10% Metabolism)
175     3:07      0:50    6:27      3:00     Trousers have given up
300     4:54      1:47    11:01     4:34     MEGA BAG
550     8:22      3:28    18:49     7:48     XXL chair required
750     10:52     2:30    24:24     5:35     Two seats, one you
1000    13:36     2:44    30:47     6:23     FEAST BAG (+25% Metabolism)
2000    20:43     7:07    42:37     11:50    Booth rebuilt around you
3500    29:15     8:32    56:44     14:07    BANQUET BAG
6000    39:02     9:47    75:34     18:50    MONSTROSITY BAG
8000    44:38     5:36    86:59     11:25    Half the restaurant (+50%)
10000   48:05     3:27    95:01     8:02     YOU'VE OUTGROWN THE RESTAURANT

engaged, no upgrades : 4,327 presses  1,565 bags  216 merges  1,338 eats  2,639 bite clicks
engaged, smart upgr. : 4,431 presses  1,602 bags  248 merges  1,342 eats  2,680 bites  5 upgrades  -> 49:14
```

Two things fall straight out of that table and they drive most of this document:

1. **Minutes 13:36 -> 39:02 contain three consecutive 7-10 minute reward gaps.**
   That is 25 of the 48 minutes — over half the run — delivering three rewards.
   For a casual player the same stretch is 30:47 -> 75:34 with gaps of 11:50,
   14:07 and **18:50**.
2. **Buying upgrades makes run 0 slower.** 49:14 vs 48:05, and every milestone
   from 550 kg onward lands later. The panel that is one of the two largest
   things on screen is, for a first-time player, a net negative.

And one from the press budget:

3. **61% of every press in the game is a chewing click.** 2,639 bite clicks out
   of 4,327 total presses. Merging — the mechanic the entire design document is
   about — is 216 presses, **5%**. The game the player actually performs is
   "buy, then click food 2,639 times".

---

# 1. FIRST IMPRESSION

## What a new player notices first

From the capture, in descending visual weight: the four coloured stat pills
across the top, then the right-hand panel, then the tray strip, then the left
rail. **The character is the smallest meaningful thing on screen** — roughly 40px
tall in a 1920px frame, back turned, dead centre and partly behind the tray.

The room is a 140x140 stud box with a red/white checkerboard floor. The
checkerboard is the loudest thing in the 3D scene and it is the floor.

## Is it obvious what to do?

**No.** Specifically:

- The tutorial's step 1 reads **"Tap FILL TRAY to spend Calories on mystery
  food."** There is no button labelled FILL TRAY anywhere. `UI/Shop` builds a
  **34x34 px button whose entire label is the glyph `↡`**, sitting next to a
  62x34 green button that says **BUY**. A player following the instruction
  literally will not find the thing it names.
- `setPanel("shop")` does open the shop by default, so the row *is* on screen on
  a clean boot — but the capture I took shows UPGRADES open instead (a station
  or the rail can swap it), and in that state the tutorial instructs the player
  to tap a button that is not rendered at all.
- **The player starts with 0 Calories.** `Economy.newState(0)` -> calories = 0,
  income floor 15 Cal/s, Snack Bag = 100. **The first 6.7 seconds of the game
  are unactionable**: the tutorial says buy, and nothing is affordable.

## Do the currencies read?

**No.** Two of the four top pills are **CALORIES / SEC** and **CALORIES**,
adjacent, differing by two words. Worse, the concept is inverted from real life:
in an eating game a player expects calories to be *the thing that makes you fat*.
Here Calories are money, and weight comes from a different action entirely. That
is a genuine conceptual collision, not a labelling nitpick.

## Do I know where upgrades are, and what to work toward?

Upgrades: yes, the rail is legible. But on open, the panel shows **four rows
priced 20.0K / 150K / 80.0K / 400K against a balance of ~0**. The first
impression of the upgrade system is "everything is locked".

Work toward: the NEXT MILESTONE card (75 kg) is good and well placed. The NEW
DIET card in the opposite corner simultaneously announces **10,000 kg** to a
60 kg player. 166x your current weight, in minute one, permanently on screen.

## Visually overwhelming?

Eight separate persistent UI clusters: top pills, left rail, right panel, next
milestone card, tray strip, action bar, new diet card, tutorial card. They
occupy the four corners and all four edges. The 3D game occupies the hole in the
middle.

## Does it look fun?

The *visual language* does — FredokaOne, 3-5px dark outlines, saturated fills,
chunky radii. This is correctly Roblox and it is the strongest thing about the
presentation. The *scene* does not: an empty box with four blocky NPCs.

## Inferred first-session shape

**First 30 seconds** — ~7s of dead time, then Snack Bags become affordable at
one per 6.7s. Tutorial step 1 clears on either BUY or FILL. Step 2 (merge) is
satisfiable immediately because `StartingTray = {1,1,1,2}` contains a complete
3-group. Step 3 (eat) requires discovering a drag onto a table the player spawns
44 studs away from.

**First 2 minutes** — 75 kg at 1:16 (measured). +1 tray slot, size stage 2,
confetti, flash, shake, toast. This beat is good and it lands on time.

**First 5 minutes** — 110 kg at 2:17 (Big Bag), 175 at 3:07, 300 at 4:54 (Mega
Bag). Four milestones, two new bags, five size stages. **This is the best part
of the game and it is correctly tuned.**

**10-15 minutes** — 550 at 8:22, 750 at 10:52, 1000 at 13:36 (Feast Bag). Still
acceptable, but between the Mega Bag at 4:54 and the Feast Bag at 13:36 there is
an **8:42 window with no new food, no new bag and no new mechanic** — only two
+1 slot grants. This is where the first real churn risk sits, and everything past
it is worse.

---

# 2. CORE GAMEPLAY LOOP

## The loop as built

```
FILL TRAY (spend Calories)  ->  tray fills with tiered food
     -> drag tile onto match      = MERGE   (income up, slot freed)
     -> drag tile onto table tray = PLATE, then click 1-5x = EAT (weight up,
                                    half the item's rate kept forever)
     -> weight crosses milestone  = slots / metabolism / next bag
     -> repeat  ->  10,000 kg  ->  NEW DIET
```

The design is genuinely good on paper: two verbs competing for the same scarce
resource (tray slots), with an asymmetry that makes the choice real.

## Where it breaks in practice

**The measured verb mix does not match the design.** Design says the game is
about eat-vs-merge. Measurement says 61% chew clicks, 34% buying, **5% merging**.
Three-of-a-kind is simply rare on a 6-16 slot tray, so the tray leans on
AUTO-MERGE and on eating. The existing TODO already flags this; the fresh sim
confirms it at 216 merges across 48 minutes.

**Friction: eating is 5 clicks on a flat coloured cube.** `EatBitesMax = 5` and
the count only falls below 5 above ~110 kg. So the *first two milestones*, the
part of the game that decides retention, are eaten at 5 clicks per fry for
2.75 kg each. First milestone = +15 kg = ~5.5 items = **~28 clicks plus 6
drags**, all onto a small screen-clamped 2D widget over a booth the player
spawns 44 studs away from.

**Friction: AUTO-MERGE occupies the prime bottom-centre action slot and is dead
for the first ~15 minutes.** Its label reads "Buy in Upgrades" (`UI/Tray:547`).
The single most prominent button in the action bar is an advertisement for a
20,000 Cal purchase.

**Every upgrade meaningfully improves the loop?** No — see §3.

## Ratings

| Axis | Score | Why not 8+ |
|---|---:|---|
| Eating satisfaction | 5 | Crumb particles, pitch-rising bite sound and "+N kg" float all exist and are well built. But the object being eaten is a coloured cube with a 2D button over it, at 5 clicks each, on a table that is often off-camera. |
| Character growth | **4** | `SCALE`/`GIRTH` tables are well designed (width outruns height, head lags). But `CameraController` sets zoom to `max(size) * 2.4`, so the camera retreats in exact proportion to growth — **the character occupies the same fraction of the screen at 10,000 kg as at 60 kg.** The core fantasy is cancelled by the camera. |
| Reward feedback | 7 | Confetti (capped at 90 pieces), flash, HUD shake, float text, ground shockwave ring, milestone sound. Genuinely good. Loses points only because it fires around a screen where the player is looking at a tray, not a character. |
| Progression | **5** | Excellent 0-5 min. Three consecutive 7-10 min gaps from 13:36 to 39:02. |
| Variety | **3** | One room, one verb pair, twelve foods that differ by an emoji and a number. No abilities, no events, no rare items, no quests. |
| Sense of achievement | 6 | Milestone copy is genuinely funny ("Trousers have given up", "Two seats, one you"). Undercut because the body change they announce is not visible on screen. |
| Humor | 6 | The writing is funny. Nothing else is. The board says "i'm eatin' it", which is the best joke in the build, and it is a static decal on a back wall partly hidden by the rail. |
| Replayability | **4** | One rebirth shipped. Size eras past run 0 deferred. Nothing rotates. |
| Clarity | **4** | Tutorial names a non-existent button; two near-identical Calorie pills; tray tiles read "T3 / 2.7". |
| "One more upgrade" | **5** | Four upgrades total, one of which measurably slows run 0. |

---

# 3. PROGRESSION SPEED

## Verdict by phase

| Phase | Engaged | Classification |
|---|---|---|
| 0:00 - 5:00 | 4 milestones, 2 bags, 5 size stages | **GOOD** |
| 5:00 - 13:36 | 2 slot grants, no new bag, no new food | **SLIGHTLY SLOW** |
| 13:36 - 39:02 | 3 milestones in 25 minutes | **TOO SLOW** |
| 39:02 - 48:05 | 2 milestones + rebirth | **GOOD** |
| Casual, 30:47 - 75:34 | 3 milestones in 45 minutes | **EXTREMELY GRINDY** |

Total run length (48:05 engaged / 95:01 casual) is **correct** — do not touch
`KG_SCALE` to fix this. The problem is *distribution*, not length.

## Bag ladder

| Bag | Cost | x prev | Unlock kg | Reached (engaged) | Gap |
|---|---:|---:|---:|---|---|
| Snack | 100 | — | 0 | 0:00 | — |
| Big | 870 | 8.7x | 110 | 2:17 | 2:17 |
| Mega | 11,250 | 12.9x | 300 | 4:54 | 2:37 |
| Feast | 165,000 | 14.7x | 1,000 | 13:36 | **8:42** |
| Banquet | 1,440,000 | 8.7x | 3,500 | 29:15 | **15:39** |
| Monstrosity | 10,000,000 | 6.9x | 6,000 | 39:02 | 9:47 |

The **15:39 between Feast and Banquet is the longest content drought in the
game**, and it overlaps the two worst milestone gaps. Minutes 13-29 are one
uninterrupted stretch of the same bag, the same twelve foods and two slot grants.

## Upgrades

| Upgrade | L1 cost | First reachable | Cost in seconds of total income at that point |
|---|---:|---|---:|
| Auto-Merge | 20,000 | ~5-6 min (134 Cal/s) | ~149 s |
| Walk-In Fridge | 80,000 | ~9-10 min | ~310 s |
| Bigger Table | 150,000 | ~13 min | ~190 s |
| Slow Metabolism | 400,000 | ~13-14 min (787 Cal/s) | **~508 s** |

- **Slow Metabolism is the problem child.** 400,000 Calories for +10%. At the
  moment it first becomes reachable it costs **8.5 minutes of not buying bags**
  to gain 10%. It does eventually return more than it cost over the back half of
  a run, but the measured net effect of the whole smart-upgrade policy is
  **-1:09 on the run and every mid-game milestone landing later**.
- **Walk-In Fridge is the second-cheapest upgrade and is near-useless in
  session 1.** +1h on an 8h offline cap, which milestones already raise to 10h
  and 12h for free. A player in minute 10 has never been offline.
- The panel is not sorted by price (20K, 150K, 80K, 400K), so the cheapest
  meaningful next purchase is not the one nearest the top.

## Numerical recommendations

**R1. Give the player starting Calories.** `Balance.StartingCalories = 300`
(three Snack Bags). Removes 6.7 seconds of dead time at the exact moment the
tutorial is demanding an action. Effect on the run is ~20 seconds of opening
income — negligible; re-run PacingSim to confirm the first milestone moves
1:16 -> ~1:05.

**R2. Add four milestones, and pay for them by SPLITTING existing grants rather
than adding new ones** — so total slots granted stays at 10 and the sim's
pacing is essentially unchanged. Only the *cadence* changes.

| kg | Grant | Taken from | Fills gap |
|---:|---|---|---|
| 1,400 | +1 slot | 2,000's +2 becomes +1 | 13:36 -> 20:43 (7:07) |
| 2,700 | offline cap 13h | 3,500's cap moves here | 20:43 -> 29:15 (8:32) |
| 4,500 | +1 slot | 6,000's +2 becomes +1 | 29:15 -> 39:02 (9:47) |
| 7,000 | offline cap 14h | new (free, sim-neutral) | 39:02 -> 44:38 (5:36) |

Projected engaged gaps after: no gap above ~5 minutes. Casual: no gap above
~10 minutes. **Re-run PacingSim** — moving a slot grant earlier speeds the run
slightly.

**R3. `Upgrades.metabolism.baseCost` 400,000 -> 120,000.** Payback at first
reach falls from ~508 s of total income to ~152 s, which is the same band
Auto-Merge sits in. Re-run the smart-upgrade policy; target is that smart play
is no longer slower than no-upgrade play.

**R4. Sort the upgrade panel by current cost ascending**, and **hide Walk-In
Fridge until the player's second session**. An upgrade whose entire value is
offline earnings is noise to someone who has never been offline.

**R5. Do NOT change `KG_SCALE`, `CAL_RATIO`, `KG_RATIO` or `MERGE_COST`** for
any of the above. Run length is right. See DO NOT BREAK below.

---

# 4. PROGRESSION TIMELINE

| Time | Ideal | Measured / inferred | Verdict |
|---|---|---|---|
| 0-30 s | Understand + immediate reward | 6.7 s dead, then a bag every 6.7 s; tutorial names a button that does not exist | **MISS** |
| 1-2 min | First meaningful upgrade | 75 kg at **1:16** — +1 slot, size stage 2, confetti | **HIT** |
| 3-5 min | Clearly bigger/stronger | 300 kg at **4:54**, size stage 5, Mega Bag. Body scale 1.0 -> 1.7, girth 1.0 -> 1.5. **But the camera pulled back 1.7x at the same time.** | **HIT on paper, MISS on screen** |
| 5-10 min | New mechanic, food or area | Nothing. Next bag at 13:36. No new mechanic ever ships. | **MISS** |
| 10-20 min | Bigger progression goal | Feast Bag 13:36, 2000 kg 20:43 | **HIT** |
| 20-30 min | Major milestone | Banquet Bag 29:15 — but the 8:32 gap before it is the second-worst in the game | **WEAK HIT** |

**Calculable:** every time above, every price, every gap, every press count.

**Needs real playtesting:**
- Whether 5 bite-clicks per item in the opening two minutes reads as satisfying
  chewing or as a chore. **This should be verified through playtesting.**
- Whether the drag-to-table gesture is discoverable without being told twice.
  **This should be verified through playtesting.**
- Whether the 2->1 merge is experienced as a trap.
  **This should be verified through playtesting.**
- Whether the camera pull-back actually reads as "I'm not growing" or whether
  the shrinking room compensates. **This should be verified through playtesting.**

---

# 5. CONSTANT GOALS

## What is on screen at any moment

| Goal type | Exists? | Visible without opening a panel? |
|---|---|---|
| Next milestone | Yes | **Yes** — the card is good |
| Next bag | Yes | Only with the shop panel open |
| Next size stage | Yes, but **unlabelled** | No |
| Next upgrade | 4 total | Only with the upgrades panel open |
| Next zone | **Does not exist** | — |
| Next ability | **Does not exist** | — |
| Collection / cosmetic | **Does not exist** | — |
| Quest / challenge | **Does not exist** | — |
| Next rebirth | Yes | Yes — and it says 10,000 kg to a 60 kg player |
| Rare item | Jackpot, 0.1% | Not as a goal, only as a surprise |

So at minute 10 the player has exactly **two** live goals, one of which is 100x
away. That is the opposite of overlapping.

## How to get overlap without feature bloat

Three additions, all reusing things already built:

**G1. Put the next bag unlock in the always-visible NEXT MILESTONE card.**
The card already renders a progress bar against `Progression.nextMilestone`.
Add a second, thinner bar underneath for the next locked bag's `unlockKg`. Zero
new systems, one new bar, and it makes "80% to a slot / 40% to Feast Bag" true
at a glance.

**G2. Label the size stage and show the next one.** `SizeStage` is already a
replicated attribute and `Progression.SizeStages` already has 13 thresholds.
Show "SIZE 5 / 13" on the weight pill. A player who can see they are 5 of 13
has a goal that does not exist today.

**G3. The Menu Board collection.** The McFatty's board on the back wall already
exists as geometry. Make it a 12-slot menu where each food is greyed out until
first eaten, then stamped in colour permanently (survives rebirth, derived from
a bitmask in save data). This is the cheapest possible third goal track: it uses
an existing prop, needs no economy changes, gives a reason to eat high tiers
rather than only merge them, and gives a returning player something visible that
did not reset. **Recommended.**

Rejected on complexity grounds: quests, achievements, battle pass, challenges
board. Three overlapping tracks is enough; five is clutter.

---

# 6. GETTING FATTER / BIGGER

This is the game's headline fantasy and it is currently **the weakest part of
the build**.

## What is right

`CharacterService` is well designed. Discrete stages, not linear scaling.
`GIRTH` outruns `SCALE` so width/depth climb faster than height (ratio 0.62 ->
0.91 across the ladder). Head lags at 0.85 so the neck disappears into the
shoulders. Anchoring at stage 10 ("too fat to move") is the right call and lands
on the "Booth rebuilt around you" milestone. The `Model:ScaleTo()` bug is
correctly documented and avoided.

## What is wrong — the camera cancels all of it

`CameraController.frameCharacter()`:

```lua
local needed = math.max(DEFAULT_MIN, math.max(size.Y, size.X) * 2.4)
player.CameraMinZoomDistance = needed
```

Distance is **linear in body size**. At stage 1 the body is ~5 studs and the
camera sits at 12; at stage 13 the body is ~40 studs and the camera sits at 96.
The character therefore subtends **the same visual angle at 10,000 kg as at
60 kg**. The only cue that growth happened is that the room got smaller in frame
— and the room is a featureless 140-stud box with a checkerboard floor, which is
a poor reference object.

**Fix (highest-impact single change in this document):** make framing
sub-linear.

```lua
-- was: max(size) * 2.4
local needed = math.max(DEFAULT_MIN, 12 + math.max(size.Y, size.X) * 0.9)
```

At stage 1: body 5, camera 16.5. At stage 13: body 40, camera 48. **The
character ends up ~2.7x larger on screen than at the start**, which is the entire
point of the game. Watch for clipping at the top stages.
**This should be verified through playtesting.**

## Which juice ideas are worth adding, and which are not

**Worth it — they pay for themselves:**

1. **Screen shake scaled by the item's tier on the final bite.** `Effects.shake`
   already exists and is already applied to the HUD only (deliberately, so the
   tray stays tappable). Scale magnitude by `calPerSec(tier)^0.25`. This is
   the cheapest way to make a T12 eat feel different from a T1 eat, and it
   directly addresses the flat weight ladder (§7).
2. **A landing thud + camera kick per size stage.** `WorldEffects.sizeUp`
   already fires a shockwave ring and dust. It has no sound and no camera
   impulse. Adding both makes the milestone a physical event.
3. **Belly bounce on the idle pose above stage ~6.** A single `CFrame` sine on
   the LowerTorso Motor6D. Cheap, reads instantly, and is the funniest possible
   use of one line of code in this game.
4. **A burp on every third eat, pitch falling with tier.** One sound, one
   counter. This is the game's identity in two seconds of audio.
5. **NPCs turning to face the player above stage 8.** The four station NPCs are
   already anchored models with known positions. `CFrame.lookAt` on Heartbeat
   when the player is above a threshold. This is the only "the world reacts to
   how huge I am" beat that is nearly free.

**Not worth it — cost exceeds validation value:**

- Squeezing through doors, breaking objects, chair collapse. The design doc
  already cut these and that was correct: animation/physics cost is high, the
  player is anchored by the time they would trigger, and none of them change a
  decision.
- More crumbs. Already shipped (`WorldEffects.eat`) and correctly scaled to the
  bounding box — but invisible at stage 10+ because the camera scales with the
  body. Fixing the camera makes the existing crumbs work; adding more does not.
- Giant food appearing later. The tray is 2D tiles; a 3D giant burger needs art
  that does not exist. Do it when character art lands, not before.

---

# 7. FOOD PROGRESSION

## What exists

Twelve tiers, generated from ratios. Names are good and escalate correctly:
Small Fries -> Cheeseburger -> 6pc Nuggets -> Chili Dog -> Bucket Soda ->
Family Pizza -> Combo Platter -> Slab of Beef -> Popcorn Vat -> The Whole Menu ->
Deep-Fried Everything -> The McMonstrosity. **The naming ladder is exactly
right for this game and needs no change.**

Each tier has a distinct colour (`Theme.TierColor`) and a distinct emoji
(`Theme.TierIcon`). A tray reads at a glance.

## What is wrong

**1. The player never sees the names.** `UI/Tray:_paint` renders the badge as
`"T" .. tier` and the rate as a bare number. The tile in the capture reads
**"T3 / 2.7"**. Twelve well-written joke names exist in `Config/Food` and
**none of them appear anywhere in the UI**. This is the single cheapest fix in
this document: the game's best comedy asset is invisible.

**2. Better food is barely better at the thing that gates progress.** Cal/s
spans 1.00 -> 246.79 (**247x**). Eat-weight spans 2.75 -> 28.95 (**10.5x**).
Weight is what gates the run. So eating The McMonstrosity feels like eating
fries — because in weight terms it very nearly is.

This is **locked by invariant 2** (`KG_RATIO` must stay below `MERGE_COST` =
1.5) and cannot be fixed by tuning. So fix it *presentationally*:

- Float the **digestion gain** as the headline eat reward for high tiers, not
  the kg. `Client/Main` already floats `"+N Cal/s forever"` — it currently
  renders in the same size and colour as everything else. That number spans
  247x. Make it the big one above tier ~6.
- Scale the eat VFX, shake and chew sound by tier (§6).

**3. New food is not an event.** A Feast Bag pull that lands a Popcorn Vat looks
identical to a Snack Bag pull that lands fries: a tile appears. There is no
first-time-seen moment for any of the twelve foods. **G3 (the Menu Board
collection) fixes this and is recommended for exactly this reason.**

---

# 8. UI / UX REVIEW

## Does it look like a Roblox game?

**Yes, and this is the build's strongest presentational asset.** `UI/Theme`
commits properly: `FredokaOne` display font, `GothamBold` for numbers, 3-5px
dark-brown outlines on every element, 12-16px corner radii, a saturated warm
palette. There is no dashboard-grey anywhere and no thin hairline borders. Do not
change the visual language.

## Visual hierarchy — this is the problem

Priority should be gameplay > main progression > objective > actions > secondary.
Measured from the capture (approx. screen coverage at 1920x790):

| Element | Position | Coverage | Should be |
|---|---|---:|---|
| 4 stat pills | top-left, full-width | ~5% | 2 pills |
| Right panel (shop/upgrades) | right edge | ~6% | on demand only |
| Tray strip | bottom centre | ~5% | correct |
| Left rail (4 icons) | left edge | ~2% | correct |
| Next milestone card | top centre-right | ~1.3% | correct |
| New Diet card | bottom right | ~2% | **hide until 5,000 kg** |
| Tutorial card | bottom left | ~2.5% | correct, temporary |
| Action bar (AUTO-MERGE only) | bottom centre | ~1.6% | **replace contents** |
| **The character** | centre | **~0.1%** | **the biggest thing on screen** |

**Eight persistent clusters on all four edges.** Individually each is fine.
Together they frame the gameplay out of its own screen.

## Specific fixes

**U1. Merge the two Calorie pills into one.** `7.91K` headline with `+15/s` as a
subtitle inside the same pill. Frees a top-bar slot and kills the
CALORIES/CALORIES-PER-SEC collision.

**U2. Rename the spendable currency, or accept the collision.** In an eating
game, "Calories" reads as the fatness resource, not as money. The food on the
tray is being *sold* — call the income **CASH / $** and keep kg as the only
body number. If the parody branding demands Calories, at minimum make the pills
visually opposite (one filled, one outline) rather than adjacent green/orange.

**U3. Give FILL TRAY a real button.** Currently 34x34 px labelled `↡`
(`UI/Shop:128-137`). Make it the primary: at least **120x44 px, text
"FILL TRAY"**, and demote single BUY to the small square marked `x1`. Then the
tutorial's step 1 names something that exists.

**U4. Put FILL TRAY in the bottom action bar.** The bar currently holds one
button (AUTO-MERGE) at `0.3333` width, leaving two thirds of the bar empty, and
that button reads "Buy in Upgrades" for the first ~15 minutes. Put a big
**FILL TRAY** button there with the current bag's cost on it. That makes the
game's primary verb reachable without a panel — which is what the tutorial
assumes.

**U5. Hide the NEW DIET card until ~5,000 kg.** Showing "10,000 kg / Current
60 kg / NOT YET" from second one is a permanent reminder that the goal is 166x
away. The rail already has a rebirth icon for anyone curious.

**U6. Show food names.** Under the icon at TextSize 10, or in the drag hint
line (`Tray.counter` already has 700px of width for exactly this kind of text).

**U7. Fix the record-board occlusion.** The lifetime-mass board on the back
wall sits directly behind the left rail in the default camera framing (visible
in the capture as "...MASS / ...kg / ...ou have ever eaten"). Move the board or
move the spawn.

## Buttons — audit

- Sizes: the shop's FILL at 34x34 is below any reasonable touch target.
  Everything else (BUY 62x34, upgrade BUY, rail icons ~54x54, tray cells 84x84,
  action buttons ~245x62) is fine.
- Icons: emoji throughout. Consistent, readable, zero asset risk. Correct choice.
- Padding/radius/outline: consistent, because everything routes through
  `Theme.card`/`Theme.button`. Good.
- Hover/click feedback: `AutoButtonColor` on buttons, `AutoButtonColor = false`
  on tray slots (correct — they are drag handles). But **there is no press
  animation on the big action buttons**. `Effects.pop` already exists and is
  used on tiles and pills; wire it to button `Activated`.
- Disabled state: shop rows grey out correctly when unaffordable; upgrade rows
  do not visibly distinguish "cannot afford" from "affordable" in the capture.

---

# 9. PLAYER FEEDBACK / GAME JUICE

## What already exists (and is good)

`UI/Effects`: squash-and-stretch pop, rising float text with stroke,
hand-simulated confetti with a 90-piece cap, full-screen colour flash,
positional shake applied to the HUD only. `Client/WorldEffects`: crumb burst on
eat, shockwave ring + dust on size-up, implosion on rebirth, gold burst on
jackpot — **all scaled to the character's bounding box**, which is the right call
and rarely done.

`UI/Sounds`: pitch-rising merge cascades, pitch climbing through a meal's bites
(`1 + 0.06 * left`). Engine built-in assets only, so nothing can be moderated
away. This is careful work.

**The problem is not that juice is missing. It is that the juice fires on a
screen where the player is looking at a 2D tray, and the character it decorates
is 40 pixels tall.**

## The 5 highest-impact feedback improvements

1. **Fix the camera framing (§6).** Every existing world effect immediately
   becomes ~2.7x more visible. This is one line and it is the highest-leverage
   change in the entire review.
2. **Hold-to-chew.** Holding the plated food auto-bites at ~4/sec instead of
   requiring 5 discrete clicks. Removes the single largest friction in the game
   without removing the bite feedback.
3. **Scale eat feedback by tier** — shake magnitude, chew pitch, crumb count.
   Makes twelve foods feel like twelve foods despite a 10.5x weight ladder.
4. **A sound and a camera kick on `sizeUp`.** It currently has a ring and dust
   and is silent. This is the moment the game exists for.
5. **Press feedback on the primary buttons.** `Effects.pop` on `Activated` for
   FILL TRAY / BUY / upgrade BUY. Currently a purchase is confirmed only by a
   number changing somewhere else on screen.

Deliberately not recommended: more confetti, more flashes, screen-wide shake.
The build is already at the right loudness; adding more would cross into
obnoxious and would hurt the 90-piece particle cap on phones.

---

# 10. MOBILE EXPERIENCE

`Client/Main` scales the whole UI: `uiScale.Scale = clamp(min(vpX/1280,
vpY/720), 0.55, 1)`. Sensible approach. The problems are layout, not scaling.

## Likely problems

**M1. The tray sits exactly where the mobile controls live.** `TrayRoot` is
`AnchorPoint (0.5, 1)`, `Position (0.5, 0, 1, -12)`, width
`20 + 8*84 + 7*8 = 748 px`. The action bar sits *below* it at the very bottom
centre. Roblox's mobile thumbstick occupies bottom-left and the jump button
bottom-right. On a phone, the tray strip and the action bar will overlap or
crowd both. **This should be verified through playtesting on a real device.**

**M2. Panel text becomes unreadable.** The shop is 286px wide with TextSize 15
names and TextSize 14 costs. At a 0.55-0.7 scale that is 8-10px text on a phone.
The upgrade panel is worse — TextSize 11 descriptions become ~6-7px.

**M3. The 34x34 FILL button is ~19-24 px on a phone.** Unhittable. (U3 fixes it.)

**M4. Eating requires precise taps on a small screen-clamped widget over a 3D
table.** The widget is already clamped to screen (good — the fourth session
solved the off-camera case). But 5 precise taps per fry on a phone, while the
camera can be moved by the same finger, is the worst interaction in the build
on mobile. **Hold-to-chew fixes this for mobile more than for desktop.**

**M5. Drag-to-merge competes with camera drag.** The tray tile captures the
input so the gesture itself should work, but a drag that starts slightly off a
tile rotates the camera instead of picking up food. `DRAG_THRESHOLD = 8` px is
tuned for a mouse; on touch, 8px of travel is nothing — a tap will frequently
register as a drag. Raise the threshold for touch specifically (e.g. 8 for
mouse, 16 for touch).

## Recommendations

- Move the action bar **above** the tray, not below it, and inset both from the
  bottom by ~90px so the mobile controls have their band back.
- On viewports narrower than ~800px, reduce `COLUMNS` from 8 to 5 and let the
  tray wrap to more rows. It already grows rows correctly
  (`Tray:render` -> `rows = ceil(slots / COLUMNS)`).
- Set a floor on panel text: give the panels their own `UIScale` clamped to a
  higher minimum than the global one, or use `TextScaled` with
  `UITextSizeConstraint`.
- Touch-specific drag threshold.

---

# 11. MAP DESIGN

## What is there

One 140x140 stud room (`ROOM = 140`, `Stations.Room`). Red/white checkerboard
floor across the whole footprint. Counter at the back with the McFatty's board.
Four blocky NPCs with large floating white outlined labels. A fryer, a weigh-in
scale, a drive-thru window, a record board. One booth with a table and a tray.
That is approximately **ten objects in 19,600 square studs**.

## Problems

**P1. It is mostly empty floor.** The player spawns ~44 studs from the booth
where the game's eating verb happens, and the walk between them passes nothing.

**P2. There is nothing you cannot reach.** The prompt asks whether players see
things and think "what is over there?" — **there is no over-there.** The room has
four walls and everything inside them is available in minute one. The game has
zero spatial progression, and rebirth's promise ("new maximum size: CITY CLASS")
has no physical referent anywhere in the world.

**P3. No landmark.** The most memorable object is the floor pattern.

**P4. Flat lighting.** The capture shows an evenly washed white ceiling and no
strong light sources. Nothing draws the eye anywhere.

## Recommendations

**MAP1 — the big one. Shrink the room and spend the reclaimed space on
visible-but-locked content.** Reduce the play area to roughly 70x90 (keep the
20-stud tile multiple; `ROOM` is mirrored in `Config/Stations` so change both and
rebuild). Behind a glass wall on the far side, build three low-detail dioramas:

- **DRIVE-THRU LOT** — "UNLOCKS AT DIET 1"
- **FOOD COURT** — "UNLOCKS AT DIET 2"
- **DOWNTOWN** — "UNLOCKS AT DIET 3"

They do not need to be playable. They need to be *visible*, *labelled*, and
*locked*. This single change gives the game the "what is over there" pull it
completely lacks today, and it makes the rebirth card's promise concrete instead
of abstract. Cost: walls, three signs, some blocky props.

**MAP2. Spawn the player at the booth.** The eating verb lives there, the
tutorial's step 3 requires it, and the existing TODO already flags that the booth
is rarely in frame. Move the SpawnLocation into the booth and give the initial
camera a framing that includes both the character and the table.

**MAP3. Make the McFatty's board a landmark.** It already holds the funniest
line in the build. Light it, raise it, and turn it into the Menu Board
collection (G3). One prop doing three jobs.

**MAP4. Put the weigh-in scale to work.** The Nutritionist already stands at a
scale. Have the scale display the player's current kg on a SurfaceGui in
enormous digits. Free, thematic, and gives the map one interactive readout.

---

# 12. WHAT WOULD MAKE A PLAYER QUIT

## 🔴 Very likely to hurt retention

**Q1. Minutes 13:36 to 39:02 — three reward gaps of 7:07, 8:32 and 9:47.**
Over half the run delivering three rewards, with no new bag between 13:36 and
29:15 (15:39). For a casual player the same stretch has gaps of 11:50, 14:07
and **18:50**. This is the primary churn point.

**Q2. The camera cancels visible growth.** Zoom distance is linear in body size
(`max(size) * 2.4`), so the character subtends the same visual angle at 10,000 kg
as at 60 kg. A player who cannot see themselves getting huge has no reason to
pursue the one thing the game is about.

**Q3. Tutorial step 1 names a button that does not exist.** "Tap FILL TRAY"
against a 34x34 button labelled `↡`, next to a larger button labelled BUY.

**Q4. The first 6.7 seconds are unactionable.** 0 starting Calories, 100-Calorie
first bag, 15 Cal/s floor.

**Q5. 61% of all presses are chewing clicks** (2,639 of 4,327), and the worst
concentration is the opening — 5 bites per item until ~110 kg, when each item is
worth 2.75-3.5 kg against a 15 kg first milestone.

## 🟠 Potential issue

**Q6. Two adjacent pills labelled CALORIES / SEC and CALORIES**, in a game where
"calories" intuitively means the fatness resource but actually means money.

**Q7. Slow Metabolism costs 400,000 for +10%** — roughly 8.5 minutes of not
buying bags at the moment it becomes reachable. The measured smart-upgrade run
is **1:09 slower** than never buying an upgrade at all.

**Q8. Walk-In Fridge is the second-cheapest upgrade and is worthless in session
one** (+1h on an 8h cap, when milestones already grant 10h and 12h free).

**Q9. An empty 140x140 room with nothing locked, nothing distant, no landmark.**

**Q10. The NEW DIET card shows a 10,000 kg target to a 60 kg player, permanently,
from second one.**

**Q11. Twelve foods differ only by an emoji and a number, and their names — the
best comedy in the build — are never shown.**

## 🟡 Needs playtesting

**Q12. Mobile: the tray and action bar occupy the thumbstick/jump band.**
**This should be verified through playtesting on a real device.**

**Q13. Is the 2->1 merge a trap?** It is always a -17% income trade, offered at
exactly the moment a player has two and no third. Correct as a slot-clearing
move; invisible as a cost. **This should be verified through playtesting**, then
fixed by showing the exchange on the drag hint if confirmed.

**Q14. Hand-merging is 5% of presses.** The tray leans on AUTO-MERGE and eating.
Whether that reads as "the merge game is broken" or as "auto-merge is a great
upgrade" is not something a model can answer.
**This should be verified through playtesting.**

**Q15. Nutritionist / Drive-Thru station toasts unconfirmed** (carried over —
see OPEN below). Prompts fire and the branch is verified correct, but no toast
appeared. `StationsBuilder` sets `MaxActivationDistance = 12` and never sets
`RequiresLineOfSight = false`, despite the second session's changelog claiming
both were fixed — check that first.

---

# 13. WHAT WOULD KEEP A PLAYER PLAYING

The target chain:

```
"one more food" -> "one more upgrade" -> "almost at the next zone"
   -> "I want to see the next food" -> "I want to be huge"
      -> "I want the next major milestone"
```

| Link | Supported today? | Where it breaks |
|---|---|---|
| One more food | **Yes** | FILL TRAY is 6.7 s of income early. This link is fine. |
| One more upgrade | **Weak** | Four upgrades exist. One is a trap, one is useless in session 1. |
| Almost at the next zone | **BROKEN** | There are no zones. Nothing in the world is locked or distant. |
| See the next food | **Weak** | New tiers arrive as an unremarked tile. No names shown, no first-seen moment. |
| I want to be huge | **BROKEN** | The camera keeps you the same size on screen. |
| Next major milestone | **Yes, then no** | Excellent to 5 min. Three 7-10 min gaps from 13:36. |

**Three of six links are broken or weak, and the two hard breaks are the two
that carry the fantasy.** Fixing the camera (§6) and adding the locked dioramas
(MAP1) repairs both, and neither touches the economy.

---

# 14. SYSTEMS I SHOULD ADD

Only four, and each has to earn its place against this specific game.

**S1. Menu Board food collection.** ✅ **Add.**
*Why this game needs it:* twelve foods with genuinely funny names that the
player never sees, an existing back-wall prop doing nothing, no permanent
non-economy goal, and no reason to eat a high tier rather than merge it. One
system solves four problems. Stamps survive rebirth, derived from a bitmask in
save data (same "derive everything from stored state" pattern as `peakKg`).

**S2. Server-local size leaderboard.** ✅ **Add — and ship it in v1, not
patch 2.** *Why this game needs it:* the entire premise produces a spectacle
that currently nobody else can see. A board on the back wall listing the top 5
players in the server by current kg costs no DataStore, no OrderedDataStore, no
persistence work — it is a loop over `Players:GetPlayers()` reading an attribute
that already exists. It is the cheapest possible answer to "HOW DID HE GET THAT
BIG". The global lifetime-mass leaderboard can stay in patch 2.

**S3. Golden food.** ✅ **Add.** *Why this game needs it:* the jackpot already
exists at 0.1% and produces a gold particle burst — but it only grants a higher
tier, which is indistinguishable from a lucky roll once it is a tile in the tray.
Make a jackpot pull land as a **golden variant of its tier**: 3x eat-weight, a
gold outline on the tile, and its own sound. That turns a statistical event into
a moment, and it gives the flat weight ladder (§7) an occasional spike.
Reuses `Bags.roll`'s existing jackpot return value.

**S4. Hold-to-chew.** ✅ **Add.** Not a "system" so much as the single largest
friction removal available. See §9.

**Rejected:**

- **Pets/helpers.** ❌ Feature bloat. This game already has an auto-merge
  helper and a passive income floor. A pet would be a fifth number.
- **Quests / NPC challenges.** ❌ The four NPCs are already panel-openers.
  Making them quest givers adds a tracking UI to a screen with eight clusters.
- **Server events / giant shared food.** ❌ Not until there is more than one
  room and more than one reason for two players to be near each other.
- **Destructible objects / size-based abilities.** ❌ The player is anchored
  from stage 10. Abilities for a stationary character are a different game.
- **Temporary multipliers.** ❌ The economy has three ratios and two invariants
  holding it together. A timed multiplier is the easiest way to break invariant
  1 by accident.
- **Eating streaks / combos.** ❌ Would reward spamming the verb that is already
  61% of presses.

---

# 15. SOCIAL EXPERIENCE

Today: **zero social hooks.** The record board on the back wall shows only the
local player's lifetime mass. State is already per-UserId, so a shared
restaurant is placement work rather than an economy rewrite — but it is not
built.

The premise is one of the most naturally social in the simulator genre: a player
who is 40 studs tall is *self-evidently* impressive with no UI at all.

**Strongest ideas, in order:**

1. **Server size leaderboard on the back wall** (S2). Cheap, immediate,
   directly powers "how did he get that big".
2. **A size comparison beat.** When a player crosses a size stage while another
   player is within ~60 studs, both get a brief "X IS NOW BIGGER THAN YOU" /
   "YOU ARE NOW THE BIGGEST IN THE SERVER" toast. Reuses `panels:toast`.
3. **Shared restaurant.** Already architecturally free; it just needs the
   placement work. Without it, ideas 1 and 2 have nobody to compare against.

**Rejected:** group challenges, giant shared food, racing for rare food. All
require coordination systems that a game with one room and no chat hooks cannot
support yet.

---

# 16. RETENTION — why come back tomorrow

Pick **three**, and two are already built:

1. **Daily bag streak.** ✅ Built (`Balance.DailyRewards`, 7 rungs, rewards are
   bags rather than Calories — correct, since a bag is worth roughly the same at
   every stage). **It is under-surfaced.** The rail pip is a small dot. Make the
   DAILY rail icon pulse and show the streak day number when a claim is
   available.
2. **Offline Calories + bonus bags.** ✅ Built
   (`Balance.OfflineBonusBags`: 1h -> 3 snack, 4h -> big, 8h -> mega, capped at
   8h base). The return screen exists. This is the strongest built retention
   hook and needs nothing.
3. **Menu Board collection** (S1). ⬅ **Add.** Daily rewards and offline income
   both give you *resources*; neither gives you a *goal that did not reset*.
   A returning player needs to see something permanent with gaps in it.

**Rejected:** achievements list, rotating rare food, events. Three is enough for
a v1 and all three reuse existing systems.

---

# 17. MONETIZATION

Hooks already exist in player state: `IncomeMultiplier`, `BonusSlots`,
`OfflineCapHours`. Nothing needs re-tuning to add passes.

**Sell:**

| Product | Price band | Why it is fair |
|---|---|---|
| **Auto-Chew** | ~99 R$ | Plated food bites itself at ~2/sec. Sells convenience against the 61%-chew problem. **Only ship this alongside the free hold-to-chew fix** — selling a solution to a problem you left in on purpose is the version of this that players correctly resent. |
| **2x Calories** | ~199 R$ | Income only. Weight still requires eating, so it accelerates the economy without skipping the fantasy. |
| **+4 Tray Slots** | ~149 R$ | `BonusSlots` exists. Slots are the scarce resource the whole design turns on, so this is real value — but it is capped and it does not grant weight. |
| **24h Offline Cap** | ~199 R$ | Pure convenience. |
| **Cosmetic hats/skins** | any | Ideal for this game once character art lands. |

**Keep earnable, never sell:**

- **Weight / kg in any form.** It is the fantasy and the progression. Selling it
  is selling the game's ending.
- **Rebirths / Metabolism multipliers.** Same reason, one level up.
- **Bag unlocks.** They are the weight ladder's payoff.

**Would feel pay-to-win:** anything granting kg, any Metabolism multiplier,
any "skip to Diet 1".

**When monetization should first appear:** not before the **first Mega Bag
(~4:54)**, and ideally not before the **first rejoin**. The first five minutes
are the best-tuned part of this game — do not spend them on a store prompt.

---

# 18. PRIORITY LIST

## 🔴 CRITICAL

1. **Camera framing cancels visible growth.** One line in `CameraController`.
2. **Milestone gaps of 7:07 / 8:32 / 9:47 between 13:36 and 39:02** (casual:
   11:50 / 14:07 / 18:50).
3. **Tutorial step 1 names a button that does not exist** and can be off-screen.
4. **0 starting Calories = 6.7 seconds of unactionable game.**
5. **61% of presses are chewing clicks** — hold-to-chew.

## 🟠 HIGH IMPACT

6. Visible-but-locked areas behind glass (MAP1) — the game has no "over there".
7. Merge the two Calorie pills / clarify the currency.
8. FILL TRAY as a real, large button, in the action bar as well as the shop.
9. Show food names — twelve funny names currently invisible.
10. Slow Metabolism 400,000 -> 120,000; sort the upgrade panel by price; hide
    Walk-In Fridge until session 2.
11. Server-local size leaderboard (S2).
12. Spawn at the booth (MAP2).

## 🟡 MEDIUM

13. Menu Board food collection (S1).
14. Next-bag progress bar on the milestone card (G1); "SIZE 5/13" on the weight
    pill (G2).
15. Golden food on jackpot (S3).
16. Mobile: move the action bar above the tray, inset from the bottom band,
    5 columns under 800px, touch-specific drag threshold.
17. Hide the NEW DIET card until ~5,000 kg.
18. Scale eat feedback (shake / pitch / crumbs) by tier.
19. Resolve the station-toast bug (check `MaxActivationDistance` and
    `RequiresLineOfSight` in `StationsBuilder` first).

## 🟢 POLISH

20. Sound + camera kick on `sizeUp`.
21. Belly bounce above stage 6; burp every third eat.
22. NPCs turn to face very large players.
23. `Effects.pop` on primary button press.
24. Light the McFatty's board; put live kg on the weigh-in scale.
25. Remove the `Debug` action before release (already `IsStudio`-gated).
26. Merge animation on the tray tiles; real plated-food art instead of a cube.

---

# 19. TOP 10 CHANGES

### 1. Make the camera framing sub-linear

- **Problem:** `needed = max(size.Y, size.X) * 2.4` — camera distance is linear
  in body size, so the character occupies the same fraction of the screen at
  10,000 kg as at 60 kg.
- **Why the player cares:** "become ridiculously huge" is the entire game and it
  is currently invisible. Every milestone toast announces a body change the
  player cannot see.
- **Recommended change:** `needed = max(DEFAULT_MIN, 12 + max(size.Y, size.X) * 0.9)`
- **Numbers:** stage 1 body 5 / camera 16.5; stage 13 body 40 / camera 48. The
  character reads **~2.7x larger on screen** at max size than at start, versus
  1.0x today. Watch for wall clipping at the top stages.
- **Difficulty:** Easy (one line) — **Priority: Critical**

### 2. Close the 13-39 minute reward drought

- **Problem:** measured gaps of 7:07, 8:32 and 9:47 back to back; 15:39 between
  the Feast Bag and the Banquet Bag with no new food. Casual: 11:50, 14:07,
  **18:50**.
- **Why the player cares:** 25 of 48 minutes with three rewards is where players
  put the game down, and it is the last thing they will remember.
- **Recommended change:** insert milestones at **1,400 / 2,700 / 4,500 /
  7,000 kg**, paid for by splitting the existing `+2` slot grants at 2,000 and
  6,000 into `+1`s, and moving the 12h offline cap from 3,500 to 2,700.
- **Numbers:** total slots granted stays at **10** (tray 6 -> 16), so pacing is
  near-unchanged. Projected engaged gaps: **no gap above ~5:00**. Casual: no gap
  above ~10:00. **Re-run PacingSim** — an earlier slot grant speeds the run
  slightly.
- **Difficulty:** Easy (data only) — **Priority: Critical**

### 3. Hold-to-chew

- **Problem:** 2,639 bite clicks out of 4,327 total presses. 5 bites per item
  until ~110 kg, when an item is worth 2.75 kg against a 15 kg first milestone.
- **Why the player cares:** it is 61% of everything they do, it is heaviest in
  the opening two minutes, and on a phone it is 5 precise taps on a small widget
  over a 3D table.
- **Recommended change:** holding the plated food auto-bites at **4 bites/sec**.
  Keep the per-bite sound, pitch and crumbs — this removes the clicking, not the
  feedback. If holding is not wanted, the alternative is `EatBitesMax` 5 -> 3 and
  `EatBitesPerDecade` 1.8 -> 1.1 (bites become 3/3/2/2/1 instead of 5/4/3/2/1),
  cutting ~35% of bite presses.
- **Numbers:** either path removes roughly 900-1,300 presses from a run. That
  makes the run **faster**, so `KG_SCALE` will need to come **down** from 2.75
  (expect ~2.2-2.4). **Must be re-measured with PacingSim** — this touches the
  pacing dial.
- **Difficulty:** Medium (input handling + a re-tune) — **Priority: Critical**

### 4. Give the player 300 starting Calories

- **Problem:** `Economy.newState(0)` starts at 0 Calories. Snack Bag is 100,
  income floor is 15 Cal/s, so the first 6.7 seconds are unactionable — while
  the tutorial is demanding an action.
- **Why the player cares:** the first thing a new player does is fail to do the
  thing they were just told to do.
- **Recommended change:** `Balance.StartingCalories = 300` (three Snack Bags
  immediately).
- **Numbers:** 300 Cal is about 20 seconds of opening income. Expect the first
  milestone to move **1:16 -> ~1:05**. Confirm with PacingSim.
- **Difficulty:** Easy — **Priority: Critical**

### 5. Make FILL TRAY a real button, in two places

- **Problem:** the tutorial says "Tap FILL TRAY". `UI/Shop:128-137` builds a
  **34x34 px** button whose label is the glyph `↡`, beside a larger green BUY.
  And if the UPGRADES panel is the open one, neither is on screen.
- **Why the player cares:** step 1 of the tutorial is unfollowable as written.
- **Recommended change:** (a) in the shop row, make FILL the primary at
  **at least 120x44 px** with the text "FILL TRAY", and demote single-buy to a
  small `x1` square. (b) put a large **FILL TRAY** button in the bottom action
  bar next to AUTO-MERGE, showing the selected bag's cost.
- **Numbers:** 34x34 -> 120x44 is about 4.6x the hit area; roughly 19-24 px ->
  66-84 px on a scaled-down phone viewport. The action bar currently uses 1 of
  its 3 slots.
- **Difficulty:** Easy — **Priority: Critical**

### 6. Build something the player can see but cannot reach

- **Problem:** the map is one 140x140 box with ~10 objects and four walls.
  Nothing is distant, nothing is locked, nothing is unexplained. There is no
  "what is over there".
- **Why the player cares:** it is one of the two strongest pulls in any Roblox
  simulator, and this game has none of it. It also makes rebirth's "new maximum
  size: CITY CLASS" concrete rather than abstract.
- **Recommended change:** shrink the play area to ~**70x90** (keep the 20-stud
  tile multiple; `ROOM` is mirrored in `Config/Stations`, change both and
  rebuild). Put a glass wall along the far side with three low-detail dioramas
  behind it: **DRIVE-THRU LOT / UNLOCKS AT DIET 1**, **FOOD COURT / DIET 2**,
  **DOWNTOWN / DIET 3**. Not playable — visible, labelled, locked.
- **Numbers:** floor area 19,600 -> 6,300 studs squared, reclaiming ~13,000 for
  locked content and shortening every walk in the game.
- **Difficulty:** Medium — **Priority: High**

### 7. Fix the currency read

- **Problem:** two adjacent top pills read **CALORIES / SEC** and **CALORIES**,
  and in an eating game "calories" intuitively means the fatness resource while
  here it means money.
- **Why the player cares:** they cannot tell which number is which, and the
  mental model they arrive with is wrong.
- **Recommended change:** merge them into one pill — `7.91K` headline with
  `+15/s` beneath, inside the same card. Preferred, if branding allows: rename
  the spendable currency to **CASH / $** and leave kg as the only body number.
- **Numbers:** four top pills -> three; one fewer thing competing at the top of
  the hierarchy.
- **Difficulty:** Easy (merge) / Medium (rename touches UI copy throughout) —
  **Priority: High**

### 8. Show the food names

- **Problem:** `Config/Food` holds twelve well-written joke names. `UI/Tray`
  renders `"T" .. tier` and a bare rate. The tile reads **"T3 / 2.7"**. The
  names appear **nowhere in the UI**.
- **Why the player cares:** "The McMonstrosity" is funnier and more motivating
  than "T12", and the food ladder is one of the game's best assets.
- **Recommended change:** name under the icon at TextSize 10 (or in
  `Tray.counter`, which already reserves 700px for hint text). Pair with the Menu
  Board collection so each name has a first-time moment.
- **Numbers:** 12 names, currently 0 surfaced.
- **Difficulty:** Easy — **Priority: High**

### 9. Repair the upgrade panel

- **Problem:** Slow Metabolism costs **400,000** for +10% — about **8.5 minutes
  of not buying bags** when it first becomes reachable. Measured: the
  smart-upgrade run finishes **49:14 vs 48:05** and every milestone from 550 kg
  lands later. Walk-In Fridge is the second-cheapest at 80,000 and grants +1h on
  an 8h cap that milestones already raise for free. The panel is not sorted by
  price (20K, 150K, 80K, 400K).
- **Why the player cares:** the second-largest UI element on screen is, for a
  first-time player, a menu of ways to slow yourself down.
- **Recommended change:** `metabolism.baseCost` **400,000 -> 120,000**; sort rows
  by current cost ascending; hide Walk-In Fridge until the player's second
  session.
- **Numbers:** metabolism payback at first reach ~508 s of total income -> ~152 s
  (the same band as Auto-Merge). Target: smart-upgrade play measures **faster**
  than no-upgrade play, not slower. Re-run PacingSim.
- **Difficulty:** Easy — **Priority: High**

### 10. Server-local size leaderboard

- **Problem:** the game's whole premise produces a spectacle nobody else can see.
  The only board in the room shows the local player their own lifetime mass, and
  it is occluded by the left rail in the default framing.
- **Why the player cares:** "HOW DID HE GET THAT BIG" is the free marketing this
  concept generates, and today there is no surface for it.
- **Recommended change:** a back-wall SurfaceGui listing the top 5 players in the
  server by current kg, updating every few seconds. No DataStore, no
  OrderedDataStore — a loop over `Players:GetPlayers()` reading state the client
  already receives. Global lifetime-mass leaderboards stay in patch 2.
- **Numbers:** 5 rows, ~2s refresh, zero persistence cost.
- **Difficulty:** Easy — **Priority: High**

---

# 20. FINAL VERDICT

**Understandable within 30 seconds?** — **No.** The tutorial names a button that
does not exist under that name, and the first 6.7 seconds are unactionable.

**Does the core loop look satisfying?** — **Somewhat.** The design is sound and
the first five minutes are genuinely well tuned. But the measured verb mix (61%
chewing, 5% merging) is not the loop the design describes.

**Is progression too slow?** — **Overall, no — 48:05 engaged / 95:01 casual is
right. But minutes 13:36-39:02 are too slow**, with three consecutive gaps of
7:07, 8:32 and 9:47 (casual: 11:50, 14:07, 18:50). Fix the distribution, not the
length.

**Are upgrades spaced well?** — **No.** Four exist. One measurably slows run 0
(smart-upgrade play: 49:14 vs 48:05). One is useless in session 1. The panel is
not sorted by price and opens showing four unaffordable rows.

**Does becoming bigger look rewarding enough?** — **No.** The body scale tables
are good; the camera cancels them by retreating in exact proportion to growth.

**Is the UI good?** — **Structurally over-dense, visually strong.** Eight
persistent clusters on four edges around a character that occupies ~0.1% of the
screen.

**Does the UI feel Roblox-friendly?** — **Yes.** FredokaOne, thick dark
outlines, chunky radii, saturated palette, emoji icons, no dashboard grey. This
is the best-executed part of the presentation and should not change.

**Is there enough variety?** — **No.** One room, one verb pair, twelve foods that
differ by an emoji and a number, four upgrades, zero events, zero abilities,
zero collectibles.

**Is there always something to work toward?** — **No.** At minute 10 the player
has two live goals and one of them is 100x away. Minutes 13-39 have three.

**Biggest retention risk** — the **13:36-39:02 reward drought**.

**Strongest part of the game** — the **integrity of the economy**. Two invariants
asserted on server boot that refuse to start a broken build, a headless pacing
simulator that drives the shipped rules rather than a copy, 46/46 rules tests,
and every balance decision in the design doc measured rather than guessed. Almost
no Roblox simulator has this, and it is why every recommendation above could be
made numeric.

**Weakest part of the game** — the **character does not visibly get bigger**,
which is the one thing the entire game is about.

**Current estimated quality: 5.5 / 10**
**Potential quality: 8.5 / 10**

The gap between those two numbers is unusually cheap to close. Four of the five
CRITICAL items are one-line or data-only changes, and none of them touch the
balance invariants.

---
---

# OPEN — needs a human or a playtest

1. **Nutritionist / Drive-thru station toasts unconfirmed.** Prompts fire, the
   `Station` notify reaches the client, the branch code in `Client/Main` is
   verified correct, no error is logged — yet no toast appeared in testing.
   Cashier and Fry Cook (which open panels, not toasts) work end-to-end. A
   debug print is sitting in the `Station` branch of `Client/Main`
   (`[McFatty DEBUG] station notify: ...`): play, talk to the Nutritionist, read
   the output, then delete the print. **Check `StationsBuilder` first** — it
   sets `MaxActivationDistance = 12` and never sets `RequiresLineOfSight =
   false`, though the second session's changelog says both were fixed (16 studs,
   no line of sight). That is a plausible cause.
2. **Is the 2->1 merge a trap?** Always a -17% income trade, offered at exactly
   the moment a player has two of something and no third. Correct as a
   slot-clearing move, invisible as a cost. Options if a playtest confirms it:
   show the exchange on the drag hint, give it a different sound from a
   3-merge, or pay a small Calorie rebate. Measure before touching it.
3. **Hand-merging is a minor verb.** Fresh sim: 216 merges vs 1,338 eats vs
   2,639 bite clicks. The tray leans on AUTO-MERGE and eating rather than on the
   player pairing things up. The sim never pair-merges (a losing trade), so a
   real player who does will merge more. A model cannot answer whether it feels
   right.
4. **Auto-Merge / upgrade pricing.** Fresh measurement: smart-upgrade play is
   **49:14 vs 48:05** — still slower, and every milestone from 550 kg lands
   later. See Top-10 #9. Measure with PacingSim, never by eye.
5. **Playtest a full run by hand.** Every pacing number in this document comes
   from the sim. It is a model, not a player.
6. **Rebirth and Jackpot effects** are verified as functions but never seen
   live. Force them in Studio: `Debug setKg 10000`, then Rebirth.
7. **Enable Studio API access** (Game Settings -> Security) for real DataStore
   testing. Without it saves are in-memory (by design), but persistence across
   rejoins is untested against a real DataStore.
8. **Mobile layout** — the tray and action bar sit in the thumbstick/jump band.
   Needs a real device.

# LATER (unchanged by this review)

- Global leaderboards (Lifetime Mass via OrderedDataStore) — patch 2. The
  **server-local** board is promoted to v1 (Top-10 #10).
- Random events (McHappy Hour, food truck, fry rain) — patch 2 by decision.
- Shared restaurant. State is already per-UserId so this is placement work, not
  an economy rewrite. Stations already replicate per-player prompts for free.
- Size eras past run 0 (building -> city -> planet). MAP1's locked dioramas are
  the cheap preview of this and should land first.
- Real character/NPC/food art. Character reads one integer (`SizeStage`); NPCs
  and props carry `StationId` attributes; food blocks are one pooled loop in
  `Client/TableFood`. All three swap in without touching game logic.
- Monetization. `IncomeMultiplier`, `BonusSlots`, `OfflineCapHours` already
  exist in player state so passes drop in without re-tuning. See section 17.

# KNOWN ISSUES

- Character is a stretched Roblox avatar. It reads as fat (width/height ratio
  climbs 0.62 -> 0.91) but it is not bespoke art.
- Twelve tiers span only 10.5x in weight against 247x in Cal/s, because
  KG_RATIO is capped below MERGE_COST. `KG_SCALE` 2.75 absorbs it for pacing,
  but the weight ladder is flat: high-tier food is an income play far more than
  a weight play. See section 7 for the presentational fix.
- The plated food is a coloured cube with a 2D button over it.
- The booth table is rarely in frame (spawn is ~44 studs away and the character
  is in the way). The 2D screen-clamped widgets mean eating always WORKS, but
  "drag it onto the thing you cannot see" is a weak read. MAP2 fixes this.
- Studio + MCP was unstable mid-session (plugin reload, dropped play sessions).
  If play sessions die within seconds of starting, restart Studio first.
- Consider removing the `Debug` action before any real release. It is gated
  behind `RunService:IsStudio()`, so live servers reject it, but it is there.

---

# DO NOT BREAK

Two invariants hold the whole economy up. `Balance.validate()` checks both on
server boot and refuses to start if either fails. **None of the recommendations
above require touching them.**

**Invariants 1 and 2 are measured against `Food.MergeCost` (MergeInputs /
MergeOutputs = 1.5), NEVER against `MergeInputs` (3), and never against the
2->1 fallback (2.0).** MergeCost is what a tier-(n+1) item actually costs in
tier-n items at the BEST rate — which is the rate players converge on, and the
worse path is automatically safe once it checks out. Checking MergeInputs
instead is a silent pass: 3 looks safely above KG_RATIO, but the real cost is
1.5.

**A 2->1 merge loses income and that is intended.** Two tier-n earn 2x; the one
tier-(n+1) they become earns CAL_RATIO (1.65x). It buys a tray slot and a tier.
It cannot be made positive without CAL_RATIO > 2, which invariant 1 forbids. No
script may make that trade — `mergeAll` and `runAutoMerge` take full groups
only, or Auto-Merge would cut the income of whoever bought it. Three tests pin
all of this.

1. **`CAL_RATIO` (1.65) must not exceed `MERGE_COST` (1.5) by much.** Payback
   per tier scales by `(MERGE_COST / CAL_RATIO)^(n-1)`. The original 2:1 with x3
   gave a tier-12 payback of 1.2 seconds and a full run in 2 minutes 54 seconds.
2. **`KG_RATIO` (1.275) must stay strictly below `MERGE_COST` (1.5).** Eating
   three tier-n gives `3 x kg(n)`; merging first and eating both results gives
   `2 x KG_RATIO x kg(n)`. At 1.5 they tie, and since merging also grants income
   it would strictly dominate — the core decision silently disappears.
3. **`EatCalShare` (0.5) must stay strictly below 1.** A bite keeps that
   fraction of the item's Cal/s forever (`state.digestion`). At 1.0 eating is
   income-NEUTRAL: the rate simply moves from tray to digestion, the slot comes
   back free, and the weight is a gift — so tray slots stop mattering, merging
   stops mattering, and buy-cheapest-bag-and-eat-it becomes optimal.
   `Balance.validate()` now asserts this, and EconomyTests covers both "holding
   out-earns eating" and "merging out-earns holding".

**Not an invariant, but the same class of trap: bag costs are coupled to
`MERGE_COST`.** A bag competes with building the same tier by hand out of cheap
food. When MergeCost fell 2 -> 1.5, every rung above Snack Bag was suddenly
2-12x overpriced and the engaged run went 48:48 -> 81:30. `Balance.validate()`
did NOT catch it — it only compares bags to each other, and they were all wrong
by a smooth curve. If MergeCost moves again, rescale every cost by
`(new/old)^(expectedTier - 1.35)` and re-run PacingSim.

To retune run length, use **`KG_SCALE`** in `Config/Food` (currently 2.75). It
moves total run time a great deal while barely touching the opening. Measured at
the current ratios: 1.0 -> 81:30, 2.0 -> 48:34, 4.0 -> 29:36, 8.0 -> 19:03,
16.0 -> 12:52, while the first milestone only moves 2:10 -> 1:23 -> 1:06 ->
0:53 -> 0:47. Do not reach for bag prices to change PACE: scaling them uniformly
slows the opening by the same factor as the endgame (a x4 pass buried the first
milestone 15 minutes deep), and steepening only the upper bags does nothing at
all because players just buy a cheaper rung. Bag prices exist to re-anchor the
ladder against MERGE_COST, which is a different job.

**Run length is currently correct. Do not use `KG_SCALE` to fix the 13-39
minute drought — that is a distribution problem, not a length problem.** The one
change above that legitimately requires re-tuning `KG_SCALE` is hold-to-chew
(Top-10 #3), because it removes presses from a press-bound run.

# HOW TO RUN THE TOOLS

Studio memoizes ModuleScripts, so editing config and re-running silently uses the
OLD numbers. Both tools must load a fresh clone. A helper module cannot fix this
— the helper gets cached too.

Balance sim (full snippet is in `ServerStorage.PacingSim`'s header comment):

```lua
local sb = game.ServerStorage:FindFirstChild("__SimSandbox")
if sb then sb:Destroy() end
sb = Instance.new("Folder"); sb.Name = "__SimSandbox"; sb.Parent = game.ServerStorage
local shared = game.ReplicatedStorage.Shared:Clone(); shared.Parent = sb
local sim = game.ServerStorage.PacingSim:Clone()
sim.Source = sim.Source:gsub("game%.ReplicatedStorage%.Shared", "game.ServerStorage.__SimSandbox.Shared")
sim.Parent = sb
print(require(sim).report({
	{ name = "engaged", buy = "richest", keepFraction = 0.6, actionsPerSecond = 1.5, upgrades = "smart" },
}, 12345))
sb:Destroy()
```

`upgrades` is `"smart"` (saves for Auto-Merge -> Slow Metabolism -> slots when
within 90s of income) or `"none"`. One press = one group-merge (3 items -> 2),
one FILL TRAY, or one drag-to-plate. An EAT costs `Balance.eatBites(kg)` presses,
not one — the sim charges them explicitly, because it is a client-side cost
nothing else in the codebase can see. `report` prints per-milestone timings,
which is what makes the gap analysis in this document possible.

Rules tests (46/46 currently passing):

```lua
local t = game.ServerStorage.EconomyTests:Clone()
t.Parent = game.ServerStorage
print(require(t).run())
t:Destroy()
local sb = game.ServerStorage:FindFirstChild("__TestSandbox")
if sb then sb:Destroy() end
```

Rebuild the map after editing `ServerStorage.MapBuilder` (this also re-applies
the dimmed lighting values — they live in MapBuilder, not the Lighting service).
`ROOM` is 140 and must stay a multiple of the 20-stud floor tile; it is mirrored
in `Shared.Config.Stations`, so change both and rebuild the stations too:

```lua
local m = game.ServerStorage.MapBuilder:Clone()
m.Parent = game.ServerStorage
require(m).build()
m:Destroy()
```

Rebuild the stations after editing `ServerStorage.StationsBuilder` (same
pattern; the server also builds them on boot if `Workspace.Stations` is missing):

```lua
local b = game.ServerStorage.StationsBuilder:Clone()
b.Parent = game.ServerStorage
require(b).build()
b:Destroy()
```
