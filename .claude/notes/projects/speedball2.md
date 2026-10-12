# Speedball 2 on SNES — the first real game on the SDK

Owner's project, local only (`~/workspace/speedball2-snes`; the rights
belong to Rebellion, nothing leaves the machine). Since 2026-10-10 it
builds against THIS working tree (`../opensnes`, branch `develop`): what is
in `bin/` here, committed or not, is what its `make` uses. A session of its
own works there with its own `CLAUDE.md`; from here we read it, and we
build and measure copies of `game/` in the scratchpad
(`make OPENSNES=<this tree> [CC=<another bin>/cc65816]`). Owner,
2026-10-10: it is our first game, not an example — use it to perfect the
SDK and to challenge the port's approach.

## How to measure a compiler or library step on it

Its `game/test/play.toml` input script, then
`luna profile game.sfc --from-frame 0 --until-frame 760 --input <script> --top 3000`
and the sum of master cycles of the symbols of `match.c` (`*.match`,
`match*`). NOT the busy share of a frame window: the game computes a tick
straight through and catches up, so a faster build does more work in a
given window. Reference points (2026-10-10): 99.0 M at `a66a4f53`, 98.6 M
with whole-function inlining, 93.6 M with direct-page temps in functions
that call (92.1 M with `inline` on `vectorLength` and `octantTo`).

Its two tests (`boot`, `play`) hash pictures at fixed frames and press the
pad at fixed frames while the game advances by tick: every step that makes
the code faster changes them. `luna diff a.sfc b.sfc --tolerance 6` with no
input and `testing/frame_sequence.py` tell "earlier" from "wrong".

## What it has shown the SDK

- The time is in the bodies, not in the calls; a leaf poured into a
  function that calls loses; 59 % of the logic was on stack frames.
- `audioLoadSample` raced on its end mark (found the same day by our own
  library test, once the loader's loop got faster).
- The map engine could not host its pitch: `MAP_MAXMTILES` 512 distinct
  tiles where it needs over 1000, `tmx2snes` 64x64 where it has 80x144. It
  wrote its own ring-tilemap module (`pitch.c`). A real gap, not planned yet.
- What a lag frame guarantees was written nowhere (now in the frame-budget
  page).
- `spritesUpdate` is its heaviest function outside the logic: #167
  (`order` in `OamWorldBatch`) would take part of it.
- The SA-1 is ruled out there because C does not run on it. The AI of 18
  players is the textbook load for it: a chantier to weigh.

## What we put to the owner about the port (2026-10-10)

1. The open question of decision 0003 — does the US version keep the PAL
   speed constants? — decides whether 30 ticks a second is the original's
   pace or 20 % above it, and the whole CPU budget hangs on that pace.
2. "Never a three-frame tick" may be a harder criterion than needed: 7 of
   290 today, about 29.6 ticks a second.
3. Tests indexed by frame move with the compiler; inputs indexed by tick
   (the original has a replay) would move only with the logic.
4. `buildColMap` copies the map by columns into 23 KB of RAM at boot, 38
   frames: a table the asset tool can produce, left in ROM.
5. "Kick-off under 40 %" no longer compares two builds (see above).

## Answers and what followed (2026-10-10, relayed by the owner's orchestration session)

1. Checked by the game on the two disassemblies: the US Mega Drive version
   keeps the PAL constants (seven tables identical byte for byte,
   `frames_per_tick` = 2 on both). 30 ticks a second is the US Mega Drive's
   pace, 20 % above the Amiga's. 30 or 25: the owner's decision; the game
   proposes 30.
2. Agreed; proposed criterion "no four-frame tick and at least 29 ticks a
   second". Owner's decision.
3. Agreed; what is missing is on luna's side (inputs indexed by arrival at
   a routine): the game drafts that request.
4. Agreed; the column table should come out of `opensnes-tileset`, after its
   deduplication: an issue for us is drafted, not yet published.
5. Adopted: its `measure.sh` gives the sum of cycles of `match.c`'s symbols.
   Reference with `9ec70672`: 107.24 M over 347 ticks (the game has grown).

The same day it hit bank $00 (issue #168): 3.6 KB free with half the logic
ported. Not a limit: code runs from bank $01 and up, our build refused it
wrongly (`.claude/rules/bank0_budget.md`). And it gave `decide.c`
(`~/workspace/speedball2-snes/tools/workloads/decide.c`, also in #166): the
decision burst, stand-alone, checksum `0x30B3`; 17.03 M master cycles
without the two compiler steps of the day, 14.98 M with — to become the
bench's 21st workload after 0.49.0. Its `rnd` rebuilds a carry by
comparisons (1,445 cycles a call): `s = a + b; c = s < a;` is a pattern the
back end should turn into `adc`.

Its build uses this working tree: between two of our commits it may hold an
unvalidated compiler. Say so when a step is in progress.


## Direct exchanges, 2026-10-10 afternoon (session to session, the owner's new mode)

- **#168 closed on evidence from the game**: on its real sources, bank $00
  at 92 bytes free, 32 functions in bank $01 (its own hot `rnd`,
  `vectorLength`, `predict`, and the library's `bgInitTileSet`, `oamClear`,
  `vramQueueFlush`, `oamSetSize`…), and its reference model agrees at every
  tick: 348/348, 420/420, 420/420. Its remark: a non-static C function that
  nothing calls is not linked (`-d`), so a C filler fills nothing.
- **`inline` on `vectorLength` and `octantTo` at `9ec70672`**: 107.24 M ->
  105.65 M (-1.5 %) for 1,265 bytes; kept. Less than the -4 % of the
  morning: with direct-page temps, `selectActive` gains less from becoming
  a leaf. The advice was right in direction and smaller in size a step
  later: quote such figures with the compiler commit.
- **Column-major map in `opensnes-tileset`: taken**, form agreed before a
  line was written: `column_major = true` in `[convert]`, `<name>.cmap`,
  `<name>_cols[]` / `<name>_cols_end[]`, 2-byte entries, column after
  column, rows top to bottom; entry of column c, row r at byte
  `(c * height + r) * 2`, height in tiles — to be written in the option's
  doc; refused with a message for `lz` and Mode 7.
- **The map engine: noted, not planned — and reformulated by the game.**
  What made it leave was the 512-entry ceiling, but it would have left
  anyway: its scroll must go out at the same VBlank as OAM and never on a
  lag frame (tick computed straight through, scroll set at the end, rows
  and columns through `vramqueue`), and its camera moves up to 16 px a
  tick. What it would have kept of the engine: none of its objects or
  collisions, only "give me row r and column c, ready to push". So the
  chantier, if it is ever opened, is not "raise the ceiling" but **the
  lower half of the map engine exposed alone**: the map in ROM in both
  orders, and nothing else.

## The measurement method changed (2026-10-10, evening)

luna delivered inputs indexed by arrival at a routine (luna `60b05e9`,
after v1.36.0 — NOT in our pinned binary). The game's `play.toml` now has
`input_at = "matchDecide"` and entries per tick: the command above
(`--input <script>` alone) would read those numbers as frames. Use the
game's own script, with the luna of the neighbouring checkout:

    cd <copy of game/> && ../tools/verif/measure.sh x      # last line
    # luna: ~/workspace/luna/target/stable/luna

It profiles from frame 0 to the frame where `matchDecide` is reached for
the 341st time (`luna state --input-at matchDecide --until-pc matchDecide
--hit 341`), with `--input-at matchDecide`: two builds then run exactly the
same logic, whatever their speed. This is the fixed-work measure the frame
windows could not give; it replaces the "760 frames" figures above, which
are not comparable with it.

Reference at `34cf0292`, with `inline` on its two helpers and the map by
columns: **103.47 M master cycles in `match.c` over 340 ticks**, reached at
frame 704, 16 three-frame ticks.

Also from the game, same day: with `--column-major` its first screen comes
at frame 9 instead of 50 and it no longer uses far RAM (23,040 bytes
given back); its three tests keep the same picture hashes once shifted by
41 frames. An unknown key in an asset's `.toml` is refused with a message
(provoked: `column_major` with an underscore).

Decided by the owner on 2026-10-10 (relayed by the orchestration session,
not heard in ours): the game keeps 30 ticks a second, and its criterion is
"no four-frame tick and at least 29 ticks a second". So the frame budget
question of points 1 and 2 above is settled; what remains for the SDK is
the cost of a decision burst (`decide.c`).

2026-10-10, late: goals are ported (the game's `7fd9094`). Its play script
now holds a goal and a full kick-off, a phase where few players decide:
**74.86 M over 340 ticks**, not comparable with the 103.47 M above. The
rule for measuring a step on the game from now on: build BOTH compilers on
the game's sources of the day and compare those two figures; never compare
with a figure of the day before. Bank $00: 1,894 bytes free and nothing in
bank $01 yet (the game first wrote the opposite and corrected itself: the
spill comes with its next step).

## Rights: what was removed on 2026-10-10 (owner decisions, typed here)

- The `dist` workload of `devtools/sdkbench` carried `vectorLength`, the
  game's transcription of an original routine: rewritten in an ordinary
  commit (`de759bd4`), no history rewrite — `a66a4f53` to `54b164a9` still
  show it.
- The comment of issue #166 that held `decide.c` in full (id 6096834930)
  is deleted. `decide.c` stays private in the game's repository; `depot`
  replaces it in the bench.
- Still public, found while doing it and put to the owner: the body of
  #166 (103 lines of code, `place`), and the comments 6087019566
  (`lookupflat`, `near`, `sort`, 31 lines), 6095506664 (`vectorLength` and
  `distances` in full, 31 lines) and 6096440845 (two lines). Not touched
  without his word.
- Later the same day the game's session, on the owner's words typed
  there, deleted 6095506664 and reposted its text without the code as
  6100054429 (deleting leaves no revision behind; editing would have).
  The game re-read the rest: the body of #166 and 6087019566 are code
  written for the port, not transcriptions. Two comments still NAME the
  original routine without copying it (6095577985, 6095597152): put to
  the owner by the game.
- Lesson: an excerpt pasted in a public issue is published. What the game
  shows us comes by path on this machine, and what we publish about it is
  a shape described in words or a stand-alone example of ours.

## Issues closed, and how we talk (2026-10-10, evening)

The game closed #166 (completed: 75.4 M -> 64.2 M master cycles over the
day, three-frame ticks 9 -> 1) and #167 on the owner's request typed in
its session. **#167's need is unchanged and nothing was delivered**: an
optional order array in `OamWorldBatch`. It is owed, followed between the
two sessions, and its API shape goes to the game before a line is written.
#164 (ours, SuperFX) stays open. The game now writes to us in markdown
files (`/tmp/speedball2_pour_opensnes_<subject>.md`) and a one-line
message with the path; we answer the same way
(`/tmp/opensnes_pour_speedball2_<subject>.md`). `/tmp` does not survive a
reboot: what is decided is copied here.

## The order array is delivered (2026-10-10, late)

`OamWorldBatch.order` (the need of #167): see the CHANGELOG. Two things the
game taught us on the way: its `-Werror` build broke on our UNCOMMITTED
header the minute the field was added (a public header is now worked on in
a separate worktree, `exchanges.md`), and a structure of the SDK can only
grow if the build does not make a missing trailing initializer an error
(`-Wno-missing-field-initializers` in `common.mk`, pinned by an old-form
batch in the library test ROM). Waiting for its figure on `spritesUpdate`
once it replaces its sorted copy.

`order` measured by the game on its match screen the same evening: same
rendering, everything but the idle wait 110.70 M -> 107.10 M master cycles
over the 340 ticks (-3.3 %); `spritesUpdate` -9,000 a frame against +2,300
in `oamPlaceWorld`. Its sorted copy of x, y, tile and attr is gone. Thread
closed on both sides.

## 2026-10-11: audio thread closed by the game

With `379a108b` and its workaround removed: 31 samples of 31, no failed
load, 7,585 bytes of sound memory free, first tick at frame 314, voices
1392/1392 against its simulator, 15/15 tests. Upload measured there at
about 670 master cycles a byte (FastROM; 2,270 before), three-frame ticks
21 to 19 with the sound. Its NMI is the SDK handler alone, about 800
master cycles. Still to come from it: the split of `soundTick` between its
interpreter and our commands.

## 2026-10-11: the 32-bit division, measured by the game

`c1e04d00`: `tcc_udivmod32` + `div32_ucore` 4.46 M → 1.16 M over its 340
ticks (about 650 cycles a `u32 / u16` instead of about 2,500). It had
replaced the division by two tables (12 KB of ROM, an approximate pitch
above period 4096) between our two messages and removed them: the exact
division stays. Its load: 340 ticks in 706 frames for a target of 703,
21 three-frame ticks. Heaviest over 340 ticks: `spritesUpdate` 16.7 M
(its C), `oamPlaceWorld` 13.1 M (ours), `stepPlayers` 12.9 M, `soundTick`
11.8 M, `movePlayers` 11.3 M. Nothing asked: it looks at its C first.

## 2026-10-11 — string literals become `const char[]`: shown before landing

Shown to the game before the compiler change landed (a literal handed to a
plain `char *` was read from the wrong bank in silence). Its answer: its
only literals go to `const char *` parameters and it has no plain `char *`;
it prefers the refusal to a silent guard. The host command we gave it to
check (`cc -fsyntax-only -Wwrite-strings`) does not work on a project: the
host compiler stops on `snes.h`. Do not hand that command out again.
Same day: rebuilt on `1c173f87` (text run in assembly, hex escape), 15
tests of 15, load unchanged (it does not call `textPrint`).

## 2026-10-11 — its plain RAM band at 556 bytes free; the index loop

**RAM.** With its manager screen barely begun, the game had 556 bytes free
below `$2000` and asked whether the 8 KB are a firm limit. Answered with a
compile of the day: `FAR` applies to any object, a scalar included (`lda.w
g` becomes `lda.l g`: one byte and one CPU cycle an access; `t[i].f` keeps
the same index code). What stays below is what a plain `T *` points at.
Its rule "anything over 64 bytes goes FAR" was its own, not ours. The free
figure of that band is also the stack's room (it starts at `$1FFF`): asked
to measure, it found 205 bytes at the deepest (`$1F32`, manager screen), so
350 of margin, and keeps the 512 threshold. The 544 bytes at `$0300` are
`oamMemory`; moving it needs a public header change — a lead, not done.
`sample_mirror` (512 bytes of the audio module) went `FAR` the same day
(`8734b1f9`).

**Loops.** `for (k = 0; k < 1920; k++) t[k] = 0;` costs ten instructions a
turn (291 master cycles a word, measured by the game on a stand-alone
workload it wrote for us, `tools/workloads/farfill.c` in its tree, nothing
of the game in it, offered for our bench). What the generated code shows on
five loop shapes of ours: the index is reloaded at the top of the body
although A still holds it from the compare (the A-cache is dropped at every
block entry, single predecessor or not); a `jmp` back to the condition
every turn; a `u8` index compared signed after an `and #$00FF`; and the
index never lives in X. Told the game: the first two first, with a figure
on `farClear`, before any promise on X (stage 2 of the compiler plan, the
one step that opens a new class of silent defects). For the clear itself it
took `memset` on the FAR buffer at once: 1.17 M to 0.49 M cycles.
Closed when `farClear` is under about 120 master cycles a word.

## 2026-10-12 — the loop lot landed; `farClear` 291 → 231

`0202ea6a` (a loop tests at its bottom, no reload of the index at the body's
entry): on its stand-alone workload, built in our scratch space with both
compilers, `farClear` 2.23 M → 1.77 M cycles (291 → 231 master cycles a
word), `screenFar` 21.50 M → 21.31 M, checksum 0x49DB both times. Its
`sample_mirror` ask closed the day before (1,068 bytes free in its plain
band, 16 tests of 16). Still owed: the index kept in X (its target: about
120 a word) — stage 2 of the compiler plan, after the rest of stage 1.
The hunt on this lot found `(c ? 1L : 0L) % d` stopping the compiler
(present before the lot; fixed in it).

The game's own measure of the lot, the same day: `farClear` 231 (291),
checksum unchanged; `match.c` 90.41 M cycles over the 340 ticks of its play
script (92.15 M, −1.9 %), 340 ticks in 703 frames (705), 19 three-frame
ticks (unchanged), ROM and model agreeing tick for tick on three scenarios;
no function slower. One of its tests broke on a buffer overrun of its own
(an OBJ block written with page indices), seen because far RAM moved —
nothing of ours.

## 2026-10-12 — what its match does with `oamPlaceWorld`, for the example

Asked which of its uses `examples/sprites/world_sprites` (`3e8483ba`)
should show: (1) `order` is the essential part — its twenty 32x32 players
and robots are sorted by depth at every tick (an insertion sort on their
depth, slot = 19 − i: the lowest on the screen in the first slot, in
front); (2) eight batches, three of size 32 and five of 16 (ball above,
ball below, objects, lamps, big ball), the size bits set once; (3)
`visible[]` read to choose which frames to stream to VRAM, not to count
(a sprite off screen costs no DMA); (4) a hidden batch is not placed at
all — the call costs the same on hidden sprites (+8,000 master cycles a
frame measured on 2026-10-10), hence a flag per batch. The example was
reshaped the same day (`c0ea2da8`): six 32x32 walkers sorted by y with
`order`, coins of 16x16 in a second batch, the README saying (3) and (4).
Also that day: it arms no HDMA channel (the channel rule noted in its
tooling for its fades), and it uses neither `textPrint` nor the metasprite
draw.

## 2026-10-12 — what the multiply and divide helpers weigh in it

On `2391b0e9` (the helpers waiting 8 and 16 cycles instead of 14 to 22,
`$4202` not rewritten between products that share an operand) the game
rebuilt from scratch: 20 tests of 20, fingerprints unchanged; its match
code 90.41 M → 90.38 M master cycles on its play script (three calls
only), and on the manager's path (710 frames, 17 call sites) `tcc_mul16`
and `tcc_div16` together are 0.81 M of 42 M busy cycles. So the helpers
are 2 % of that path, which bounds what any further work on them can
give it; the loop index in X (stage 2, T3) stays what it asks for.
Its report (`/tmp/speedball2_pour_opensnes_muldiv.md`, luna `793473a`,
ROM against its model 188/188 ticks, 15/15 league days) names what does
weigh on that screen: the C that composes a text layer at the pixel —
`textGlyph` (one 8x8 glyph, two planes, two tiles) 14.9 %, with `textBox`
and `textSlot` 22 % of the busy time; then `soundTick` 9.2 %, `FastNmi`
8.5 %. The same subject as its loops thread, the only one open:
`farfill.c` is its workload for the index kept in X.

## 2026-10-12 — `tileBlitGlyph2bpp` in its manager screen

The shape it asked for (one call per tile row, a far glyph pointer, OR
only, a NULL right tile allowed) shipped as `885fa8f7` the same day. Its
`textGlyph` rewritten over two calls, `LIB_MODULES += tile`, nothing else
changed, a `make`: 21 tests of 21, captures of the manager at frames 600
and 1150 identical to the byte; `textGlyph` on its manager path (710
frames, 181 glyphs) 6.33 M → 1.15 M + 1.93 M in the routine, ≈ 35 000 →
≈ 17 000 master cycles a glyph (10 700 in the routine, 6 400 in its own
envelope: `textSlot` two to four times and the pointers); busy time of the
path 42.49 M → 39.33 M (−7.4 %); the screen lights 26 frames after entry
instead of 30. Our 12 257 a glyph on `farfill.c` and its 10 700 agree
(its font is sparser, its right tile NULL one time in eight). Thread
closed by it; the loops thread (T3, `farClear` per word) is the only one
open.
