# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

MidnightMask is a single-file, dependency-free HTML5 dungeon-crawler game called
**MIDNIGHT MASQUE** — a Shin Megami Tensei / Persona-style first-person catacomb
crawl with turn-based battles, aspect "contact"/persuasion, card fusion, and a
day/night (moon phase) cycle. The entire game — markup, CSS, and JS — lives in
one file: `midnight-masque.html`. There is no other source code in this repo.

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
- `ASPECTS` — enemy/recruitable aspect → {glyph, lv, hp, atk, def, agi, weak, res, pers, rank, elem}
- `POOLS` — per-floor random encounter aspect pools
- `REACT` — aspect personality → CONTACT action → emotion reaction table (persuasion mechanic)
- `ITEMS`, `MOONS`, `MAPS` (ASCII dungeon layouts per floor), `FLOOR_NAMES`, `CHEST_LOOT`

### Global state, not modules

Four globals hold all mutable state: `MODE` (a string state machine: `'intro'`,
`'title'`, `'explore'`, `'battle'`, `'velvet'`, `'gameover'`, `'victory'`), `UI` (a stack of
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

### UI stack pattern

`refreshCtl()` looks at `UI[UI.length-1]` (or `MODE` for full-screen states) and
calls the matching `ctl*` function, which builds buttons via `rows()`/`btnEl()`
and writes them into `#ctl`. Pushing `{t:'...'}` onto `UI` drills into a
submenu; `popUI()` / Escape pops back out. Follow this pattern for any new menu
rather than adding parallel UI state.

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
each carry transient `gx`/`gy` cell coordinates (assigned by `placeParty()`/
`placeFoes()` at battle start, column `GRID_PARTY_COL`/`GRID_FOE_COL` with rows
from `centeredRows()`). `MOVE` is a command like any other action
(`cmd.kind==='move'`), resolved in `doPlayer` same as attacks — positioning is
not free. Basic ATTACK and any `elem:'phys'` skill are melee-only
(`isMelee()`), requiring `gdist()<=1` (Chebyshev distance) to the target;
`ctlTarget` greys out foes out of reach. Elemental single-target skills are
unlimited range. Skills with `area:'col'`/`'row'` in `SKILLS` hit every living
foe sharing the chosen target's column/row instead of just that one foe —
that's how the old "hits everyone" skills (Flame Lash, Torrent, Stormcall) work
now that positioning exists; picking a target just aims which line gets swept.
Enemy AI (`doEnemy`) picks melee (60%) or ranged (40%) each turn; if melee and
not adjacent to its target, it calls `stepToward()` to close in one cell
instead of attacking that turn, rather than attacking regardless of position.

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
