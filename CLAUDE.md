# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

MidnightMask is a single-file, dependency-free HTML5 dungeon-crawler game called
**MIDNIGHT MASQUE** — a Shin Megami Tensei / Persona-style first-person catacomb
crawl with turn-based battles, aspect "contact"/persuasion, card fusion, a
day/night (moon phase) cycle, and roguelike bones: procedurally generated
floors, randomized loot, and permadeath (see "Exploration / dungeon
generation" below). The entire game — markup, CSS, and JS — lives in one file:
`midnight-masque.html`. There is no other source code in this repo.

## Repo layout

```
midnight-masque.html   the entire game (HTML + <style> + <script>)
LICENSE                GPLv3
```

There is no `package.json`, build tool, bundler, linter, or test suite. Do not
introduce one unless the user explicitly asks — this project is intentionally a
single self-contained artifact.

## Running / testing changes

There is no build step. To verify a change, open the file directly in a browser
or serve it locally, e.g.:

```
python3 -m http.server 8000   # then open http://localhost:8000/midnight-masque.html
```

Since there's no test suite, verify behavior manually in-browser: start a new
game, walk into a wall/door/chest/stairs, trigger a random battle, try a skill
and a CONTACT action, and check the browser console for JS errors.

## Code architecture

Everything is global-scope ES5-style JS in one `<script>` block, organized into
labeled `/* ============ SECTION ============ */` comments in this order: DATA,
STATE, SAVE/LOAD, AUDIO, CANVAS/RENDERING, HUD/LOG/CONTROLS, EXPLORATION,
CONTROL DISPATCH, AZURE ROOM, BATTLE, TITLE/END, KEYBOARD, BOOT. When adding
code, put it in the matching section rather than appending at the end.

### Data tables (top of file) drive all content

Game content is defined as plain object/array literals, not code — extend the
game by editing these, not by writing new logic:
- `SKILLS` — skill name → {cost, elem, pow, targ, type}
- `MASQUES` — playable "persona" name → {glyph, skills, res, weak, b (stat bonuses)}
- `ASPECTS` — enemy/recruitable aspect → {glyph, lv, hp, atk, def, agi, dex, weak, res, pers, rank, elem}
- `POOLS` — per-floor random encounter aspect pools
- `REACT` — aspect personality → CONTACT action → emotion reaction table (persuasion mechanic)
- `ITEMS`, `MOONS`, `FLOOR_NAMES`, `LOOT_TABLE` (weighted chest-loot rolls, see below)

There is no static `MAPS` table anymore — floor layouts are generated fresh
per run (see "Exploration / dungeon generation").

### Global state, not modules

Four globals hold all mutable state: `MODE` (a string state machine: `'intro'`,
`'title'`, `'explore'`, `'battle'`, `'velvet'`, `'masque'`, `'gameover'`,
`'victory'`), `UI` (a stack of
menu-context objects, e.g. `{t:'items',ctx:'battle'}`), `G` (the save-game
object: party, inventory, position, floor, moon phase, etc. — this whole object
is JSON-serialized for save/load), and `B` (transient battle state, `null`
outside battle).

### Rendering loop

A single `requestAnimationFrame` loop (`loop()` → `drawScene(t)`) dispatches on
`MODE` to one `draw*` function per screen (`drawTitle`, `draw3D`/`drawSpecials`/
`drawMinimap`/`drawMoonHud` for exploration, `drawBattle`, `drawVelvet`,
`drawGameover`, `drawVictory`). The dungeon view is a hand-rolled pseudo-3D
raycast-style renderer built from `SCL` (per-depth scale factors) and
`cellOff()`/`fwd()`/`rightv()` (direction-relative cell lookups) — there's no
actual raycasting, just nested-rectangle perspective tricks per depth layer.

Wall look is per-floor: `THEMES[G.floor]` (a data table, alongside `FLOOR_NAMES`)
supplies background-gradient and wall fill/stroke colors, plus a `decor` tag
(`'school'`, `'basement'`, `'psych'`) consumed by `decorFront()`/`decorSide()` to
paint floor-specific detail (school windows, basement grime, psychedelic rings)
clipped to each wall segment. `drawFront`/`drawSide` take the animation
timestamp `t` so decor can animate (e.g. the psychedelic hue rotation);
`draw3D(t)` is what threads `t` down from `drawScene`.

### Pixel art and vector portraits (no image assets)

All character/object art is hand-authored and rendered with plain Canvas 2D
calls — there are no image files, sprite sheets, or data-URI assets anywhere
in the file, keeping the single-file/no-external-assets rule intact. Two
techniques coexist:

Tile icons and the Azure Room watermark are blocky pixel sprites via
`drawPixelArt(rows, pal, x, y, ps)` (CANVAS/RENDERING section): `rows` is an
array of equal-length strings where each character is a key into the `pal`
palette object (`'.'` = transparent), and `ps` is the on-screen size of one
pixel. Sprite data lives in the DATA section (right after `THEMES`) and is
built once at load time: `mirrorH(half)` mirrors a half-row into a symmetric
full row, `pxFillRow`/`pxSetCh` build/tweak individual rows, and
`pxOutline`/`pxShadeLeft`/`finishSprite` add a dark 1px outline pass and a
shadow-left/light-right two-tone shade split. This produced `TILE_ICONS`
(keyed by map tile character `V`/`H`/`T`/`B`/`>` — door/spring/chest/
boss-gate/stairs — each with `rows`, `pal`, and a `glow` backlight color,
drawn by `drawSpecials()`) and `MAESTRO_ROWS`/`MAESTRO_PAL` (the hooded
watermark figure drawn behind the title in `drawVelvet()`).

Party member portraits on the ASPECTS screen are smooth vector illustrations
instead — anime-style academy busts (blazer, collar, tie) drawn with
`CTX.ellipse`/`quadraticCurveTo`/`arc` paths rather than a pixel grid, since a
blocky grid can't produce curved linework at this size. `drawVectorBust(cx,
cy, s, spec)` (CANVAS/RENDERING section, right after the pixel-art helpers)
draws the shared anatomy (hair-back volume, blazer, shirt/tie, neck, face,
eyes, brows, mouth, a clipped shading overlay); `PORTRAIT_SPECS` (DATA
section, keyed by party member name — `AKI`/`BRAM`/`CORA`) supplies each
character's colors and a `bangs(hair, outline)` callback that draws their
distinct hair silhouette on top. `drawMasqueCard()` calls `drawVectorBust`
directly (no intermediate sprite data to precompute). When adding a new tile
type, add pixel sprite data; when adding a new party member, add a
`PORTRAIT_SPECS` entry with a new `bangs()` shape rather than falling back to
emoji/text glyphs — `glyph` fields in `MASQUES`/`ASPECTS` still exist and are
used in compact HTML contexts (STATS screen, battle-token labels) where a
full portrait doesn't fit.

### Exploration / dungeon generation

Floor layouts are procedurally generated, not fixed: `genFloorMap()` carves a
perfect maze on a `MAZE_W`x`MAZE_H` grid with a recursive backtracker, then
knocks down a few extra walls (a "braiding" pass) so the layout isn't pure
dead-ends. `bfsDist()` finds distances from the entrance; the farthest cell
becomes the exit (`>`, or `B` on the last floor for the boss gate), and the
remaining cells (dead ends preferred, but falling back to any open cell so
placement never runs out of room) get the Azure Room door (`V`), the spring
(`H`), and 2-3 chests (`T`). `genDungeon()` builds all `FLOOR_NAMES.length`
floors this way once per new game and stores them as `G.dungeon` — this is
part of the save object precisely so a reloaded run keeps its layout. Nothing
reads the floor-map data table anymore; `mapAt()`/`placeAtStart()`/
`drawMinimap()` all index into `G.dungeon[G.floor]` instead of a static `MAPS`.

Chest loot is rolled fresh at open time from `LOOT_TABLE` (`rollLoot()`
weighted-picks coins/an item/a random aspect card from that floor's `POOLS`,
scaling value with `G.floor`) rather than being pre-assigned per chest —
opening the "same" chest position in two different runs gives different loot.

Death and victory are both permanent: `gameOver()`/`victory()` call
`wipeSave()`, which clears the `window.storage` save slot, so there is no
reloading past a fallen party — a new run starts (and generates a new
`G.dungeon`) from the title screen.

### UI stack pattern

`refreshCtl()` looks at `UI[UI.length-1]` (or `MODE` for full-screen states) and
calls the matching `ctl*` function, which builds buttons via `rows()`/`btnEl()`
and writes them into `#ctl`. Pushing `{t:'...'}` onto `UI` drills into a
submenu; `popUI()` / Escape pops back out. Follow this pattern for any new menu
rather than adding parallel UI state.

Two different patterns exist for "screens," and which one to use depends on
whether the dungeon view should stay visible underneath: a `UI`-stack entry
(e.g. `{t:'stats'}`) renders as buttons/text in `#ctl` while `MODE` stays
`'explore'`, so `draw3D` keeps rendering the hallway behind it — this is what
STATS/CARDS/ITEMS do. A dedicated `MODE` value (e.g. `'velvet'` for the Azure
Room, `'masque'` for the ASPECTS party-portrait screen) instead takes over
`drawScene`'s canvas entirely with its own `draw*` function, with its own
`enter*`/`leave*` pair setting `MODE` back to `'explore'`. Use a `MODE` switch
when the screen should replace the exploration view rather than float above it.

### Battle flow

Turn order is decided once per round: `startRound()` rolls speed
(`agi + rnd(6)`) for every living party member and foe and sorts them into
`B.turnQueue`. `advanceTurn()` walks that queue one actor at a time — an enemy
acts immediately via `doEnemy()`, but a party member's turn just sets
`B.current` to them and waits (`ctlBattle` renders their action menu). Picking
an action calls `queueCmd()`, which resolves it **immediately** via `doPlayer()`
(not queued for later) and then calls `advanceTurn()` again after a short
`setTimeout` pacing delay — so an attack, skill, or move lands the instant it's
chosen rather than waiting for the rest of the party to also pick. `B.running`
is true whenever no one is currently waiting on player input (enemy turn or
between-action pacing), which is what `ctlBattle` checks to hide the controls.
When `B.turnQueue` is exhausted, `startRound()` runs again for a fresh round.
Elemental weak/resist multipliers are resolved in `hitFoe`/`hitChar` against
each `ASPECTS`/`MASQUES` entry's `weak`/`res` arrays.

Battles happen on a shared `GRID_W`x`GRID_H` (5x5) grid: party members and foes
each carry transient `gx`/`gy` cell coordinates. `placeParty()` puts the party
in `GRID_PARTY_COL` (rows from `centeredRows()`); `placeFoes()` scatters foes
across random distinct cells in the remaining columns, so foe starting
distance/row varies battle to battle instead of always facing the party head-on.
`MOVE` is a command like any other action (`cmd.kind==='move'`), resolved in
`doPlayer` same as attacks — positioning is not free — but unlike other
commands it doesn't end the turn: `queueCmd` special-cases `kind==='move'` to
apply it and redisplay the same actor's menu with `B.movesLeft` decremented,
so a character can reposition (possibly more than once — see below) and still
attack/cast/etc. in that same turn; `advanceTurn` resets `B.movesLeft` via
`moveRange()` when a new turn begins.

`dex` is a fifth character/aspect stat (alongside str/mag/agi, shown in the
STATS screen's bar graphs) that governs battle movement: `moveRange(unit)` is
`max(1, floor(dex/5))` — every 5 points of DEX grants one more grid cell of
movement per turn, with a floor of 1 even at low DEX. This applies to both
the player (`B.movesLeft`, decremented per successful MOVE) and enemy AI
(`doEnemy`'s melee branch loops `stepToward()` up to `moveRange(fo)` times,
stopping early once adjacent, so a fast foe can close several cells and still
attack in the same turn).

Every combatant also has a `facing` (0-3, same N/E/S/W convention as `fwd()`),
drawn as a small triangle on their token (`drawFacingArrow`). It's set on
placement (party faces east, foes face west) and updated to match whichever
direction a character last moved (`dirFromDelta`, applied in both the player
`move` handler and `stepToward`'s enemy AI). `facingMultiplier(attacker,
defender)` compares the attacker's position against the defender's facing —
attacking from the direction the defender is looking is a front hit (0.75x),
from the opposite direction is a backstab (1.5x), anything else is a side hit
(1x) — and `hitFoe`/`hitChar` both take the attacker as their first argument
specifically to apply this multiplier alongside the elemental weak/resist one.
It only applies when `elem==='phys'` (basic ATTACK, phys skills, and enemy
melee rolls) — elemental hits ignore facing entirely, so `doEnemy` tags its
melee roll `'phys'` rather than `null` specifically so this check catches it.

Basic ATTACK and any `elem:'phys'` skill are melee-only
(`isMelee()`), requiring `gdist()<=1` (Chebyshev distance) to the target;
`ctlTarget` greys out foes out of reach. Elemental single-target skills are
unlimited range. Skills with `area:'col'`/`'row'` in `SKILLS` hit every living
foe sharing the chosen target's column/row instead of just that one foe —
that's how the old "hits everyone" skills (Flame Lash, Torrent, Stormcall) work
now that positioning exists; picking a target just aims which line gets swept.
Enemy AI (`doEnemy`) is aggressive: every turn it picks `preyTarget()` (the
living party member with the lowest current HP, not necessarily the nearest)
and goes after them, focus-firing a weakened character down rather than
spreading damage randomly. It rolls melee (75%) or ranged (25%); if melee and
not already adjacent to its prey, it takes an opportunistic attack against
whichever party member it IS adjacent to instead of disengaging, and
otherwise spends its move (`stepToward()`, up to `moveRange(fo)` cells) closing
the gap to its prey, attacking immediately if that closes it to melee range
in the same turn. If a foe drops under 30% HP and hasn't already, it goes
berserk automatically ("snarls, cornered and desperate!") for the usual 1.6x
attack/immune-to-CONTACT effects — a last-stand aggression spike, independent
of the CONTACT-anger berserk trigger. The boss additionally has a 55% chance
each turn to skip its normal attack for LUNAR RAY, hitting the whole party
at once.

CONTACT (persuasion) is a distinct SMT-style mechanic: `doContact()` looks up
the aspect's personality (`pers`) and the chosen verb in `REACT` to raise an
emotion counter (`eager`/`happy`/`scared`/`angry`) on the foe; crossing a
threshold ends the fight via recruit-card, coin gift, flee, or berserk — not
damage. This is separate from the ATTACK/SKILL damage path.

Card fusion (`fuseResult`/`doFuse` in the AZURE ROOM section) combines two
aspect cards by summed `rank` into a fixed masque outcome — it's a lookup by
rank-sum thresholds, not a combinatorial recipe table.

### Persistence

`saveGame`/`loadGame` call `window.storage.set/get('midnight-masque-save', ...)`
— **not** `localStorage`. `window.storage` is a host-provided API (present in
whatever environment renders this HTML); it may not exist in a plain browser
tab, so both functions already guard with try/catch and fall back to a log
message. Don't replace this with `localStorage` unless asked — it's likely
relying on a specific host integration.

### Audio

All sound is procedural WebAudio (oscillators via `beep()`/`SFX`/`drone()`) —
there are no audio asset files, and none should be added without discussion.

## Conventions to preserve

- ES5 `var`, no semicolonless style debates — match the existing terse,
  minification-adjacent formatting (multiple statements per line, short
  variable names like `c` for character, `fo` for foe, `G`/`B` for game/battle
  state). This is a deliberate style for this file; don't reformat unrelated
  code while making a change.
- No frameworks, no build step, no external assets — keep the game a single
  loadable HTML file.
- Flavor text (log messages, titles) follows an in-world, gothic/liturgical
  tone ("The ledger will not open", "The Maestro murmurs") — match this voice
  when adding new strings.
