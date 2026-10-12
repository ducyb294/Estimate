# Level JSON Analysis — LoopHole (Level_21 / Level_23 / Level_54)

Investigation verb: **map + measure** (schema, entity vocabulary, data volume).
Read-only. No source file was modified. Scratch scripts under
`…/scratchpad/lh/` (outside any repo).

Source files (all under `C:/WORK/VIDEO/LoopHole/Assets/LevelRunner/Levels/`):

| file | raw bytes | minified | gzip(minified) |
|---|---|---|---|
| `Level_21.json` | 1,089,748 | 506,992 | 152,961 |
| `Level_23.json` | 3,173,209 | 1,467,696 | 312,172 |
| `Level_54.json` | 285,633 | 112,122 | 15,135 |

Consumer: `C:/WORK/VIDEO/LoopHole/Assets/LevelRunner/LoopHoleLevelRunner.cs`
(307,321 bytes; read only via `Grep`/`sed`, never whole).

---

## Scope

- **System entry point:** `LoopHoleLevelRunner.TryLoadLevel` — `LoopHoleLevelRunner.cs:960`,
  deserialisation at `:965` `JsonUtility.FromJson<LevelJson>(json)`.
- **Swept:** the 3 JSON files above (full parse); `LoopHoleLevelRunner.cs` via `grep`/`sed` —
  `[Serializable]` class defs (`:83–149`), load/validate (`:960–1055`), ball creation
  (`:4462–4485`), pipe creation (`:4655–4695`, `:4810–5035`), cell→hole/wall (`:1340–1372`,
  `:1430–1445`, `:1686–1710`), `BallType` enum.
- **Not swept:** the shipped game at tag `lh-0.0.3` (a64c854) — this runner is an extraction of it;
  prefabs/materials under `Resources/`; any other level in `Levels/`. Whether the *editor* that
  wrote these files still exists was not established.
- **Independent cross-check:** every headline count re-derived with `python -I` (see Measurement
  table) — numbers agreed.

---

## 1. Schema

### Top-level keys

| key | type | L21 | L23 | L54 |
|---|---|---|---|---|
| `levelId` | int | 21 | 23 | 54 |
| `isHardLevel` | bool | false | false | false |
| `ballCount` | int | 5000 | 15000 | 350 |
| `holeCount` | int | 56 | 24 | 25 |
| `coloredHoleCaptureLimit` | int | 100 | 200 | 100 |
| `overrideColoredHoleCaptureLimit` | bool | true | true | true |
| `overrideReferenceHoleCaptureRadialHalfThickness` | bool | true | true | true |
| `referenceHoleCaptureRadialHalfThickness` | float | 1.46 | 1.40 | 1.0 |
| `sourceImageId` | string | `""` | `"Screenshot 2026-07-22 at 16.42.38"` | `""` |
| `arenaRadius` | float | 7.0 | 13.0 | 7.0 |
| `ballRadius` | float | 0.07730838656425476 | 0.09091007709503174 | 0.28572142124176028 |
| `ballEntries` | array | 5000 | 15000 | 350 |
| `pipes` | array | **0 (empty)** | **0 (empty)** | 4 |
| `board` | object | ✓ | ✓ | ✓ |
| `paletteOverrides` | array | 14 | 2 | **key absent** |

### `board` (object)

| key | type | L21 | L23 | L54 |
|---|---|---|---|---|
| `rows` | int | 6 | 9 | 8 |
| `columns` | int | 16 | 8 | 7 |
| `scale` | float | **1.6** | 0 | 0 |
| `cells` | array | 96 (=6×16) | 72 (=9×8) | 56 (=8×7) |
| `spawners` | array | 0 | 0 | 0 |
| `mode` | int | 0 | 0 | 0 |
| `columnSlideColumns` | array | 0 | 0 | 0 |

`cells` length **always equals** `rows × columns` (verified in all three).

### `board.cells[]` (`CellJson`)

Every cell in all three files has the identical 6-key shape:

```json
{"cellType":2,"unitType":0,"isMystery":false,"isFreeze":false,"freezeCounter":0,"freezeOrder":0}
```

`cellType` is `1` (hole) or `2` (wall/empty) — the runner branches on exactly those two
(`:1353`, `:1366`, `:1690`). `unitType` is a `BallType` code (`0` for non-holes).

### `ballEntries[]` (`BallJson`)

Every entry in all three files is the identical shape:

```json
{"ballType":1,"position":{"x":-3.008599281311035,"y":-0.44678640365600586,"z":-1.7248622179031372}}
```

~98 bytes/entry. Positions carry **15–20 significant decimals** (measured max 20, mean 15.9 L21 /
15.6 L23 over 6000 axes).

### `pipes[]` (`PipeJson`) — L54 only

| key | type | value (all 4 identical apart from ballType) |
|---|---|---|
| `configIndex` | int | 1, 2, 3, 4 |
| `ballCount` | int | 400 each |
| `width` | int/float | 2 |
| `flowTargetDensity` | float | 0.8 |
| `segments` | array[{ballType,count}] | `[{2:400},{3:400},{4:400},{1:400}]` |
| `ballLocalPositions` | array[{x,y}] | 400 each |

### `paletteOverrides[]` (`PaletteJson`)

```json
{"ballType":1,"ball":{"r":…,"g":…,"b":…,"a":1},"inside":{…},"border":{…}}
```

L21 covers ballTypes `1…14`; L23 covers `[1, 2]`; L54 has no such key.

### Schema differences between the three levels

1. **`pipes`**: 4 populated (L54) vs 0 (L21, L23). Only L54 exercises the `PipeJson` sub-schema.
2. **`paletteOverrides`**: 14 entries (L21), 2 (L23), **key entirely absent** (L54).
3. **`board.scale`**: 1.6 (L21) vs 0 (L23, L54). `0` means "fall back to default"
   (`LoopHoleLevelRunner.cs:1331` `board.scale > 0f`).
4. **`sourceImageId`**: non-empty only in L23.
5. No difference in cell shape, ballEntry shape, or top-level key set between L21 and L23.

---

## 2. Entity vocabulary and counts

`BallType` enum (`LoopHoleLevelRunner.cs`): `None=0, Red=1, Green=2, Blue=3, Yellow=4, Purple=5,
Orange=6, Cyan=7, White=8, Black=9, Magenta=10, Pink=11, Lavender=12, Skin=13, DarkTeal=14,
Brown=15`. `ToBallType` accepts `0..15`.

| entity | L21 | L23 | L54 |
|---|---|---|---|
| cells total | 96 | 72 | 56 |
| **holes** (`cellType==1`) | **56** | **24** | **25** |
| walls/empty (`cellType==2`) | 40 | 48 | 31 |
| balls (`ballEntries`) | 5000 | 15000 | 350 |
| pipes | 0 | 0 | 4 (400 balls each) |
| spawners | 0 | 0 | 0 |
| blockerPanels / elevators / columnSlideColumns | key absent / empty | absent / empty | absent / empty |
| mystery / freeze / bomb / pair flags | **all false / 0** | all false / 0 | all false / 0 |
| `freezeCounter` / `freezeOrder` | 0 in every cell | 0 | 0 |

Grid: L21 `6×16`, L23 `9×8`, L54 `8×7`. `board.mode = 0` in all three.

### Flagged — entity present in one level but not the others

| entity | only in | evidence |
|---|---|---|
| **populated `pipes[]`** | **L54** | L54 `pipes` = 4 entries; L21/L23 `"pipes": []` |
| **`paletteOverrides` key** | **L21 + L23** | L54 has no `paletteOverrides` key at all (recursive key census) |
| `board.scale` non-default | **L21** | 1.6 vs 0 / 0 |
| `sourceImageId` non-empty | **L23** | `"Screenshot 2026-07-22 at 16.42.38"` |
| 14-colour palette in use | **L21** | balls in colours 1–14 |
| 5th hole colour with no matching ball | **L54** | hole colour `6` (Orange) exists; no Orange ball |
| 2-colour level | **L23** | balls only in colours 2 and 14 |

**No entity appears in L23 alone.** Mystery / freeze / bomb / pair / key / lock / spawner / panel /
elevator are **absent from all three files** — the C# declares `blockerPanels` and `elevators`
(`:133–134`) and reads them (`:1027–1028`), but not one of the three JSON files carries the key.

---

## 3. Ball counts per colour, holes, capacities, pipe order

### Balls per colour

| ballType | name | L21 | L23 | L54 |
|---|---|---|---|---|
| 1 | Red | 841 | – | 89 |
| 2 | Green | 476 | 7434 | 84 |
| 3 | Blue | 558 | – | 91 |
| 4 | Yellow | 301 | – | 86 |
| 5 | Purple | 541 | – | – |
| 6 | Orange | 185 | – | – |
| 7 | Cyan | 237 | – | – |
| 8 | White | 453 | – | – |
| 9 | Black | 214 | – | – |
| 10 | Magenta | 492 | – | – |
| 11 | Pink | 418 | – | – |
| 12 | Lavender | 258 | – | – |
| 13 | Skin | 20 | – | – |
| 14 | DarkTeal | 6 | 7566 | – |
| **total** | | **5000** | **15000** | **350** |

`ballEntries.length == ballCount` field in all three (no mismatch).

### Holes and their colours (`cellType==1`, grouped by `unitType`)

| level | holes | colours (unitType: count) |
|---|---|---|
| L21 | 56 | 1:4, 2:4, 3:4, 4:4, 5:4, 6:4, 7:4, 8:4, 9:4, 10:4, 11:4, 12:4, 13:4, 14:4 |
| L23 | 24 | 2:12, 14:12 |
| L54 | 25 | 1:5, 2:5, 3:5, 4:5, 6:5 |

`holes == holeCount` field in all three. Hole colour set == ball colour set in L21 and L23;
**L54 has a hole colour (6) with no ball of that colour, and hole colour 5 is absent**.

### Capacities / limits

| quantity | L21 | L23 | L54 |
|---|---|---|---|
| `coloredHoleCaptureLimit` | 100 | 200 | 100 |
| `override…CaptureLimit` | true | true | true |
| `referenceHoleCaptureRadialHalfThickness` | 1.46 | 1.40 | 1.0 |
| `arenaRadius` | 7.0 | 13.0 | 7.0 |

No move-limit field, no timer field, no countdown field exists in any of the three files.

### Pipe output ordering (L54, present)

`pipes[].segments` carries the ordered output per pipe:

| configIndex | segments | output colour |
|---|---|---|
| 1 | `[{ballType:2,count:400}]` | Green |
| 2 | `[{ballType:3,count:400}]` | Blue |
| 3 | `[{ballType:4,count:400}]` | Yellow |
| 4 | `[{ballType:1,count:400}]` | Red |

Each pipe is single-segment (no interleaving). The runner resolves this via `SegmentBallTypeAt(…)`
(`:4873`). Pipe order in the array is configIndex 1,2,3,4.

---

## 4. Dead / editor-only vs gameplay-bearing

Determined by: (a) presence of a matching field on the `[Serializable]` classes
(`LoopHoleLevelRunner.cs:83–149`), and (b) reference count of the field name in the `.cs`.

### Gameplay-bearing — declared on `LevelJson`/`BoardJson`/`CellJson`/`PipeJson`/`PaletteJson` AND read

`levelId`, `coloredHoleCaptureLimit`, `overrideColoredHoleCaptureLimit`,
`overrideReferenceHoleCaptureRadialHalfThickness`, `referenceHoleCaptureRadialHalfThickness`,
`arenaRadius`, `ballRadius`, `ballEntries` (`ballType`, `position`), `pipes.*`
(`configIndex`, `ballCount`, `width`, `flowTargetDensity`, `segments`, `ballLocalPositions`),
`board.rows`, `board.columns`, `board.scale`, `board.cells` (`cellType`, `unitType`, `objectKind`),
`board.mode`, `board.spawners`, `board.blockerPanels`, `board.elevators`, `paletteOverrides.*`.

### Dead / editor-only

| field | evidence |
|---|---|
| `isHardLevel` | present in all 3 JSON; **no `isHardLevel` member** on `LevelJson`; `grep -c` in `.cs` = **0** |
| `ballCount` | present in all 3 JSON; not a `LevelJson` field; count derived from `ballEntries.Count` (`:973`, `:4462`) |
| `holeCount` | present in all 3 JSON; not a `LevelJson` field; holes derived from `cells` with `cellType==1` |
| `sourceImageId` | present in all 3 JSON; not a `LevelJson` field; `grep -c` = **0** |
| `freezeCounter` | present in every cell of all 3 files; **not declared on `CellJson`** (`:137–146`) → dropped by `JsonUtility` |
| `freezeOrder` | same as above |
| `board.columnSlideColumns` | present (empty) in all 3; not declared on `BoardJson` (`:125–135`); `grep -c` = **0** |

### Declared but absent from the data (default-valued)

`CellJson.objectKind`, `CellJson.isBomb`, `CellJson.pairId`, `BoardJson.blockerPanels`,
`BoardJson.elevators` — keys not present in any of the three files, so they resolve to
`0`/`false`/empty. `isMystery`/`isFreeze` are present but `false` in every cell of all three.
`objectKind` is read in the hole loop (`:1366` `cell.objectKind != 0` → skip), so it is live code
resting on an all-zero default.

### Consumed but inert in a specific level

- `paletteOverrides` in **L23** overrides ballTypes `[1, 2]`; L23 has **no ball of type 1**, so only
  the type-2 override can affect gameplay there. The real L23 colours (2, 14) are not both covered.
- `board.scale` is `0` in L23/L54 → the default is used (`:1331`). Consumed, currently defaulted.
- `pipes` is an empty array in L21/L23 — the schema is present, no data.

### Confirmed non-dead by the runner's own header

`LoopHoleLevelRunner.cs:10–12` states the supported set: *"plain holes, walls, bowl balls, pipes and
per-level palette overrides"*, and that blockers *(mystery, freeze, bomb, pairs, keys/locks,
spawners, panels, elevators)* **"are not part of this build; a level that carries them still loads,
they are just ignored."** `WarnAboutUnsupportedContent` (`:1022–1055`) is the enforcer.

---

## 5. Data volume — is the 1–3 MB mostly ball positions?

**Yes for L21 and L23; no for L54.**

Byte cost per subtree, as a share of minified JSON:

| subtree | L21 | L23 | L54 |
|---|---|---|---|
| `ballEntries` | 493,343 (97.3%) | 1,459,660 (99.5%) | 34,338 (30.6%) |
| `pipes` | 2 (0.0%) | 2 (0.0%) | **71,929 (64.2%)** |
| `board` | 9,438 (1.9%) | 7,085 (0.5%) | 5,521 (4.9%) |
| `paletteOverrides` | 3,837 (0.8%) | 543 (0.0%) | — (absent) |
| all scalars | 372 | 406 | 334 |

Two independent inflations stack:

1. **Pretty-printing** — raw is ~2.15× the minified size (1,089,748 vs 506,992 for L21).
2. **Per-ball verbosity** — ~98 bytes/entry (`{"ballType":N,"position":{"x":…,"y":…,"z":…}}`), of
   which the key names are ~34 bytes and the values ~60 bytes at 15–20 significant decimals.

### Evidence on procedural generation from counts

**(a) L54 pipe positions are provably redundant — the runner already regenerates them.**
`BuildPipeLocalPositions` (`:4818`) uses `ballLocalPositions` when
`Count >= BallCount`, otherwise calls `GenerateBakedPipeBallPositions(width, ballCount, ballRadius)`
(`:4840`). I re-implemented that function faithfully and diffed the output against the shipped
arrays:

| pipe | columns×rows generated | count | max abs error vs shipped, over all 400 |
|---|---|---|---|
| cfg1–4 | 3 × 134 | 400 | **0.0** (first 5 entries byte-identical) |

So `ballLocalPositions` (71,929 bytes = 64.2% of L54) carries **zero information** beyond
`width`, `ballCount`, `ballRadius` and `segments`. Dropping the field while keeping the pipe
schema takes L54 from 285,633 raw → **15,703 bytes minified (5.5% of raw, 14.0% of its own
minified size)** — evidence, not an estimate.

**(b) L23 bowl balls are shaped like generative output.**
`y` is a **constant** `0.110910` (`ballRadius + 0.02`) for all 15,000 entries (distinctY = 1).
The population is a dense near-hex fill of the bowl footprint: nearest-neighbour distance
p50 = 0.088953, mean = 0.099166, `ballRadius` = 0.090910 (mean/radius = **1.09**); min 0.0448
(overlapping), max 0.4532 (sparse edge), sd 0.0292. A counts-only representation
`{"2":7434,"14":7566}` is **20 bytes** against 1,459,660 bytes of positions.

**(c) But exact per-ball reproduction is NOT demonstrated.**
The JSON carries no seed, no generator parameters, and no algorithm revision — only the output.
L23's NN-distance spread (0.045–0.453, sd 0.029) is inconsistent with a regular lattice, so
reproducing the *exact* layout needs a deterministic packer plus a recorded seed, not just counts.
For L21/L54 the bowl `y` is **not** a function of `(x,z)`: fitting `y = ballRadius + b + a·r²`
leaves a max residual of **0.531** (L21) / 0.495 (L54) over 2471 / 313 radial buckets — so "drop y
and recompute from radius" does not hold. `y` alone is **25.0% / 23.9% / 24.9%** of `ballEntries`.

**(d) Cheap wins that do not require a generator** (minified bytes, whole file):

| variant | L21 | L23 | L54 |
|---|---|---|---|
| baseline minified | 506,992 | 1,467,696 | 112,122 |
| drop `y`, round x/z to 2 dp, short keys | 148,546 (29.3%) | 416,468 (28.4%) | 87,143 (77.7%) |
| \+ drop `ballLocalPositions` | 148,546 | 416,468 | **15,703 (14.0%)** |

Against the **raw** file sizes that is 13.6% / 13.1% / 5.5% respectively — i.e. L21 and L23 still
ship ~148 KB / ~416 KB of per-ball coordinates after every lossless-ish trick short of a generator.

**Verdict on the question asked:** the 1–3 MB *is* mostly ball positions for L21/L23 (97.3% / 99.5%
of minified bytes). Balls could be generated from counts **only** with an added deterministic
packer + recorded seed and a bowl-surface function; that is a fidelity decision, not a mechanical
compaction — the shipped data does not contain the parameters that would reproduce it. L54's pipes
are the one case where generation is already proven byte-identical and the field is pure waste.

---

## Measurements

| quantity | L21 | L23 | L54 | unit | command |
|---|---|---|---|---|---|
| raw / minified / gzip | 1,089,748 / 506,992 / 152,961 | 3,173,209 / 1,467,696 / 312,172 | 285,633 / 112,122 / 15,135 | bytes | `node lh/volume.cjs <files>` |
| `ballEntries` share of minified | 97.3 | 99.5 | 30.6 | % | `node lh/analyse.cjs <files>` |
| avg bytes per ballEntry | 98.7 | 97.3 | 98.1 | bytes | `node lh/volume.cjs` |
| ball colours distinct | 14 | 2 | 4 | count | `node lh/vocab.cjs` |
| holes (`cellType==1`) | 56 | 24 | 25 | count | `node lh/analyse.cjs` |
| `y` distinct values | 4999 / 5000 | **1** | 350 / 350 | count | `node lh/analyse.cjs` |
| `y` range | 1.069994 | 0.000000 | 1.065563 | units | `node lh/analyse.cjs` |
| parabolic-fit max residual of `y(r)` | 0.531262 | 0.000000 | 0.495265 | units | `node lh/derived.cjs` |
| xz-radius range vs `arenaRadius` | 2.8773–6.2119 (7.0) | 2.8909–8.9486 (13.0) | 3.0857–6.7143 (7.0) | units | `node lh/analyse.cjs` |
| L23 nearest-neighbour mean | — | 0.099166 | — | units | `node lh/regen.cjs` |
| L23 NN mean / `ballRadius` | — | 1.0908 | — | ratio | `node lh/regen.cjs` |
| regenerated pipe-pos max error | — | — | **0.0** | units | `node lh/regen.cjs` |
| pipe subtree share of minified | 0.0 | 0.0 | 64.2 | % | `node lh/analyse.cjs` |
| variant-A minified size | 148,546 | 416,468 | 87,143 | bytes | `node lh/variant.cjs` |
| variant-B minified size | 148,546 | 416,468 | 15,703 | bytes | `node lh/variant.cjs` |

Independent cross-check of headline scalars, `python -I`:

```
python -I -c "import json,os; d=json.load(open('Level_54.json')); print(len(d['ballEntries']), d['ballCount'], d['holeCount'], len(d['pipes']), 'paletteOverrides' in d)"
→ 350 350 25 4 False
```
(agreed with the node pass on L54, and on L21 = `5000 5000 56 0 True`, L23 = `15000 15000 24 0 True`.)

---

## Could not measure

- **Whether the level editor that authored these files still exists**, or whether it can re-bake a
  generated (seed-based) layout. `ls` of `Assets/LevelRunner/` returned only
  `LoopHoleLevelRunner.cs`, `Levels/`, and a `.meta` file per level; no editor source was present at
  that path. A wider search of `C:/WORK/VIDEO/LoopHole/` was **not** run (out of the named brief).
- **Why L54 has an Orange (6) hole with no Orange ball**, and why hole colour 5 (Purple) is absent.
  Nothing in the JSON or runner explains per-level colour-set selection; `TryLoadLevel` accepts any
  `unitType` in `0..15`.
- **The generator/seed behind bowl-ball positions** — not present in the data (see §5c). Stated as
  "not demonstrated", not as "generated" or "hand-placed".
- **Runtime cost of `ballEntries`** — a per-frame sync exists (`:4475` comment, "the live height
  takes over from the next sync") but no frame-time measurement was attempted.

---

## UNVERIFIED LEADS — not findings

- `_proceduralWalls` (`LoopHoleLevelRunner.cs:872`, built at `:1623–1660`) implies walls are
  derived from `cellType==2` cells (`:1690`), not stored — consistent with the absent `wall` key
  (zero `wall` matches across all three files). Check: read `SpawnProceduralWalls` `:1623–1712`.
- `BallShadeVariants` (`:4536`) applies per-ball random shade variation on top of
  `paletteOverrides` — a second colour source not visible in the JSON. Check: read `:4530–4540`
  and `GetBallColor`.
- The runner's `UnitState` enum has a `Hidden` state (`:88`) that nothing in these three levels
  produces. Check: `grep -n 'UnitState.Hidden' LoopHoleLevelRunner.cs`.
- `LoopHoleLevelRunner.cs:3505` seeds `CaptureVariance` from `UnityEngine.Random` — capture visuals
  are non-deterministic per run regardless of the JSON. Check: read `:3500–3510`.
- The header (`:5`) says this class was extracted at tag `lh-0.0.3` (a64c854); the shipped game's
  own level loader may consume more of these fields than this runner does. Check:
  `git -C C:/WORK/VIDEO/LoopHole log --oneline -1 lh-0.0.3`.

---

## Report meta

- Wrote exactly one file: this report. No source, config or data-store file was edited; no
  repository is present at the primary working directory (`Is a git repository: false`), so no
  `git status` in-repo sweep applied. Scratch scripts live under
  `…/scratchpad/lh/` and were never placed inside `C:/WORK/VIDEO/LoopHole/`.
- Not committed, per the read-only brief.
