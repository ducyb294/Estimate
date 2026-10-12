# Fill Color Blocks Playable — System Design & Development Estimate

**Date:** 2026-09-25   **Engine:** Cocos Creator 3.8.7 (3D)   **Built on:** CocosPlayableAdsTemplate
**Source spec:** `Fill Color Blocks/requirement.txt`, `Fill Color Blocks/P1.mp4`, `Fill Color Blocks/P2.mp4` (60 s each, 1080×1920)
**Basis:** 1 dev · ideal hours · 8h = dev-day · 5 dev-days = week
**Estimate points:** O = L = **Base** (the plan number) · P = **Worst-case** (base + risk on the downside only). The gap is driven by risk, not a uniform band.

This report is both the system design and the estimate: `/t1k:cook` can build straight from §§5–8 without a separate planning pass.

---

## 2. Project understanding & gameplay

**Core loop (read back at the clarification gate, confirmed by the estimator on 2026-09-25):**
A portrait/landscape 3D board of ~750 cube cells forms a pixel picture; unfilled cells are grayscale (param) or dim with a colour hint dot. A 4-column bottom queue holds numbered colour blocks (front + next visible). Tapping a front block: if its colour has frontier cells adjacent to the filled area, it flies in and fills N cells of that colour in a fast wave; otherwise it moves to the 5-slot top tray. Tapping a tray block uses it once usable. Tray full = lose → fail/store. Picture 100% = level complete; P1→P2 continues with no endcard (order param); second completion → endcard/CTA. The 7 store triggers hook these events.

**Genre / structure:** hyper-casual "fill the picture" puzzle, 2 fixed levels in one build (P1 red-flower/vine picture ~22×34; P2 trophy with headline "Only %1 can reach the cup" ~22×32). 3D bevelled cubes, rigid meshes (no rigs/animation clips). Client supplies art later and supplies level JSON in our schema.

| Question | Answer |
|---|---|
| On-screen elements | Picture board (cells: unfilled / hint dot / frontier outline / filled), 4-column block queue (front + next row), 5-slot tray, headline text, BG, tutorial hand |
| The ONE interaction | Tap a block (queue front or tray) |
| What it does | Usable → block flies to the board, N cells of its colour pop in as a wave from the frontier. Not usable → block moves to the first free tray slot |
| Win | Picture 100% filled. Level 1 → seamless next level; level 2 → endcard |
| Lose | Tray full (5/5) with no usable move → fail card / store |
| Redirect triggers | X taps · X seconds · X correct blocks · X blocks in tray · X% picture revealed · 1st level complete · 2nd level complete (+ standard CTA/endcard) |

## 3. Scoping assumptions

| Driver | Decision | Effect |
|---|---|---|
| Starting point | Reuse CocosPlayableAdsTemplate + studio prior art (§4) | Flow, signals, params, CTA, audio, endcards, tutorial all `●`/`◐` |
| Engine | **3D** cubes, 3D camera (client choice) | 3D block on: art import, shading polish, FPS (≈750 cells per level) |
| Art | Client delivers 2D/3D assets later; **placeholder-first** (rigid meshes) | Art production includes a placeholder pass and one swap to final |
| Level data | **Client provides ready JSON in our schema** (§8); 2 fixed levels | Loader + validator only, no converter |
| Reference authority | **A, reference-exact**: the P1/P2 clips are the spec | Nothing imported beyond what the clips show |
| Block usability rule | Usable iff its colour has frontier cells adjacent to filled area | Drives `FillResolver` (§8) |
| Tray | 5 slots, tap to return, full = lose | `TrayController` + lose branch |
| Grayscale | Param: unfilled cells render as grayscale picture vs dim + hint dot | Material variant, not a post-process pass |
| Orientation | Portrait + landscape | Camera fit + layout QA × 2 |
| Parameters | **Standard dashboard, NOT estimated** (BG, text, SFX, colours, camera, tray capacity, fill speed, hand timing, level order, store thresholds). Billed exceptions: runtime palette recolour + grayscale material (row 16), the two non-template store triggers (row 11) | |
| Level editor | **Not in scope** (client said not required); priced separately as an option in §10d | |
| Team | Single dev, serial | Totals are serial hours |
| Networks / size | Standing default: all networks, 4.8 MB cap | Adapters/export not billed; cap sizes Build-size row |

**Out of scope:** visual level editor (optional add-on §10d), localization, multiple aspect-art sets, remote config, analytics beyond standard hooks, haptics (waived, see §8 feedback matrix).

## 4. Reuse basis ⭐

`discovery: live-corpus` (knowledge-retrieval `doc_search`, keyword across the whole corpus, 8 queries, 2026-09-25).

| Need | Framework / prior art found | Feeds module | Tag |
|---|---|---|---|
| Store redirect + 7 triggers | `RedirectService`, `MarketOptionsArbiter` (N seconds, N completes) — CocosPlayableAdsTemplate; `ClickToStoreHandler` (N taps) — CocosBeePartyPLA; seven "Store" triggers — CocosBounceDefendersPLA | `store-triggers` | ●/◐ |
| Tutorial hand | `FingerTutorial` + `FingerAction.TAP` — CocosPlayableAdsTemplate; `TutorialHandView` — CocosBeePartyPLA | `tutorial` | ◐ |
| Grayscale look | `SoldierBase._applyGrayscale` (gray effect material swap) — CocosLordMobileFuntapPLA | `palette` | ◐ |
| Queue + slot tray | `TrayBuilder`, `BoardModel.trayEmpty` — CocosHexaBugFlowPLA | `tray`, `block-queue` | ◐ (tray) |
| Cell fill/tint | `IBoardCell.fill(Color)` — CocosBlockanzaJourneyPLA | `board` | reference only |
| 3D cube grid | `VoxelIntroCube`, `cellLocalPositionCocos` — CocosBeePartyPLA | `board` | reference only |
| Juice presets | `Motion.punch` — CocosHexaBugFlowPLA; `FlyingAnimation` (template) | polish, `block-queue` fly | ◐ |
| Camera fit / orientation | `EmitterQueueView.applyCameraFit` — CocosBeePartyPLA; `OrientationLayoutSwitch` — CocosBackpackLegendsPLA; `AspectRatioFitter` (layout) | `camera-rig` | ◐ |
| Flow FSM, SignalBus, params, AudioService, endcards, ObjectPool, screen adapt | CocosPlayableAdsTemplate | bootstrap, `audio`, `end-cards`, `vfx` | ● |

**Stays custom (`○`):** frontier-based fill resolver and wave ordering, the 3D cell board with four visual states, the block queue columns, multi-level flow P1↔P2 (no prior art found for a level-order controller; scope: whole corpus), 3D camera framing, VFX content, shading polish.

## 5. Module architecture (system design, `state: to-be`)

**Root:** `assets/scripts/[Name-Project]/` — `[Name-Project]` is a placeholder; rename it to the real project folder before building.

| Module | Files | Responsibility | Upstream | Status |
|---|---|---|---|---|
| `level-data` | `level-data/LevelDataProvider.ts`, `level-data/LevelSchema.ts`, `level-data/LevelValidator.ts` | Which levels exist, in what order, with what cells and blocks? | | TBD |
| `board` | `board/BoardBuilder.ts`, `board/BoardModel.ts`, `board/CellView.ts` | Where is each cell and what state is it in? | | TBD |
| `fill` | `fill/FillResolver.ts`, `fill/FillAnimator.ts` | Can this colour be placed, and which N cells fill, in what order? | | TBD |
| `block-queue` | `block-queue/BlockQueueController.ts`, `block-queue/BlockView.ts` | Which blocks wait in which column, and what happens on tap? | | TBD |
| `tray` | `tray/TrayController.ts`, `tray/TraySlotView.ts` | Which rejected blocks are parked, and is the tray full? | | TBD |
| `objective` | `objective/ObjectiveController.ts` | How much is revealed; is the level won or lost? | | TBD |
| `level-flow` | `level-flow/LevelFlowController.ts` | Which level is next, and how do we move to it? | | TBD |
| `palette` | `palette/PaletteService.ts`, `palette/GrayscaleMaterial.ts` | What colour does each palette key render as, and is grayscale on? | | TBD |
| `camera-rig` | `camera-rig/CameraRig.ts` | How is the board framed in this orientation? | | TBD |
| `tutorial` | `tutorial/TutorialController.ts` | Where should the hand point, and when does it hide? | | TBD |
| `store-triggers` | `store-triggers/StoreTriggerController.ts` | Has any store threshold been reached? | | TBD |
| `hud` | `hud/HeadlineView.ts` | What headline text shows for this level? | | TBD |
| `vfx` | `vfx/VFXController.ts` | How do events look visually? | | TBD |
| `GameplayController` | `GameplayController.ts` | Orchestrator: wires signals to GameView/CTA | | TBD |

Framework-inherited (template, not in table): `AudioService`, `EndCardView`, `RedirectService`, `MarketOptionsArbiter`, `FingerTutorial`, `ObjectPoolManager`, `SignalBus`, parameter system.

**Rules:** modules never import each other, only `SignalBus`; only `GameplayController` touches `GameView`/`CTAService`. Every cell, flying block and VFX burst is pooled (`ObjectPoolManager`); no per-event `instantiate()`. Board and UI are prefab-authored; code only instantiates the cell prefab and sets data.

## 6. Communication flow

```
STARTUP
  GameplayController.onLoad
   → LevelDataProvider.load(levelOrder param)          [P1,P2] or [P2,P1]
   → PaletteService.apply(params)                       colours, grayscale flag
   → LevelFlowController.startLevel(0)  ── fires LevelStarted{index,level}
        BoardBuilder.build(level)        (pooled cells, frontier seeded)
        BlockQueueController.build(level.queue)
        TrayController.reset(capacity param)
        CameraRig.fit(board bounds, orientation)
        HeadlineView.set(level.headline)
        TutorialController.show(first usable block)

PLAYER INPUT
  BlockView tap → BlockTapped{blockId, source}
   → FillResolver.canPlace(colour)?
       yes → BlockAccepted{blockId, colour, count}
             FillResolver.pick(colour, n) → FillAnimator.play(cells) → CellsFilled{colour, cells}
       no  → BlockRejected{blockId}  → TrayController.add → TrayChanged{count, capacity}

GAME LOOP
  CellsFilled → BoardModel.markFilled → frontier recomputed → FrontierChanged
             → ObjectiveController → FillProgress{ratio}
             → TrayController marks usable slots (FrontierChanged)
  StoreTriggerController listens to BlockTapped / BlockAccepted / TrayChanged / FillProgress / LevelCompleted / timer

WIN
  FillProgress.ratio == 1 → LevelCompleted{index}
   → LevelFlowController: more levels? → transition → startLevel(next)
                          else → AllLevelsCompleted → GameplayController → EndCard / CTA

LOSE
  TrayChanged.count == capacity AND no usable move → TrayOverflow
   → GameplayController → fail card / RedirectService.fire()
```

## 7. Signal contracts

| Signal | Payload | Fired by | Consumed by | Trigger |
|---|---|---|---|---|
| `LevelStarted` | `{index, levelId}` | level-flow | board, block-queue, tray, camera-rig, hud, tutorial, objective | level begins |
| `BlockTapped` | `{blockId, source: 'queue'\|'tray'}` | block-queue, tray | GameplayController (routing), store-triggers, tutorial | any block tap |
| `BlockAccepted` | `{blockId, colour, count}` | GameplayController (after `FillResolver.canPlace`) | fill, block-queue/tray (remove), store-triggers, vfx | usable tap |
| `BlockRejected` | `{blockId}` | GameplayController | tray, block-queue, vfx | unusable tap |
| `CellsFilled` | `{colour, cells: CellId[]}` | fill | board, objective, vfx | fill wave done |
| `FrontierChanged` | `{usableColours: string[]}` | board | tray, tutorial | after each fill |
| `FillProgress` | `{filled, total, ratio}` | objective | store-triggers | after each fill |
| `TrayChanged` | `{count, capacity}` | tray | objective, store-triggers | add/remove |
| `TrayOverflow` | `{}` | objective | GameplayController | tray full, no move |
| `LevelCompleted` | `{index}` | objective | level-flow, store-triggers, vfx | ratio == 1 |
| `AllLevelsCompleted` | `{}` | level-flow | GameplayController | last level done |

No module imports another; all edges above go through `SignalBus`.

## 8. Class summary, level-data schema, class reference

### Class summary

| Class | Module | Key properties | Key methods |
|---|---|---|---|
| `LevelDataProvider` | level-data | `levels`, `order` | `load(order)`, `get(i)` |
| `LevelValidator` | level-data | — | `validate(level)` (colour counts in queue = cells per colour) |
| `BoardModel` | board | `cells[]`, `filled`, `frontier` | `markFilled()`, `frontierOf(colour)` |
| `BoardBuilder` | board | `cellPrefab`, `cellSize` | `build(level)`, `teardown()` |
| `CellView` | board | `state`, `colour` | `setState()`, `popFill(delay)` |
| `FillResolver` | fill | — | `canPlace(colour)`, `pick(colour, n)` (BFS from frontier) |
| `FillAnimator` | fill | `fillSpeed` param | `play(cells)` |
| `BlockQueueController` | block-queue | `columns[4]` | `build(queue)`, `take(blockId)`, `shift(col)` |
| `BlockView` | block-queue | `colour`, `count` | `flyTo(target)` |
| `TrayController` | tray | `slots[]`, `capacity` | `add()`, `remove()`, `refreshUsable()` |
| `ObjectiveController` | objective | `filled`, `total` | `onCellsFilled()`, `onTrayChanged()` |
| `LevelFlowController` | level-flow | `index`, `order` | `startLevel(i)`, `next()` |
| `PaletteService` | palette | `map`, `grayscale` | `colourOf(key)`, `apply(params)` |
| `CameraRig` | camera-rig | angle/zoom/offset params | `fit(bounds, orientation)` |
| `TutorialController` | tutorial | `FingerTutorial`, hand timing | `show(target)`, `hide()` |
| `StoreTriggerController` | store-triggers | 7 thresholds | `on*()` → `RedirectService.fire()` |
| `HeadlineView` | hud | label | `set(text)` |
| `VFXController` | vfx | pooled effect refs | `on[Signal]()` |
| `GameplayController` | root | module refs | `startGame()`, `onWin()`, `onLose()` |

### Level-data schema (client-authored JSON)

```json
{
  "id": "P1",
  "width": 22, "height": 34,
  "headline": "",
  "palette": { "G": "#4CD964", "R": "#E8453C", "D": "#8B1A1A", "Y": "#FFD23F", "B": "#3CC8F0", "O": "#FF8A2A" },
  "cells": ["GGGGRRDD...", "..."],
  "seedFilled": ["0,33", "1,33"],
  "queue": [
    [ { "c": "R", "n": 35 }, { "c": "G", "n": 20 } ],
    [ { "c": "R", "n": 35 }, { "c": "B", "n": 6 } ],
    [ { "c": "G", "n": 25 }, { "c": "D", "n": 30 } ],
    [ { "c": "G", "n": 30 }, { "c": "Y", "n": 8 } ]
  ]
}
```

- `cells`: one string per row, top to bottom, one palette key per cell; `.` = no cell.
- `seedFilled` (**assumption, confirm with client**): the reference starts with a few filled cells at the bottom edge that seed the frontier. Alternative: "bottom row is always frontier".
- `queue`: 4 columns, index 0 = front. The validator checks that the sum of `n` per colour equals that colour's cell count.
- Palette colours, tray capacity, fill speed, hand timing and level order are dashboard params (default, not billed). The decode and validation are billed (row 2).

### Class reference (bespoke modules)

- **`FillResolver`** (plain TS, no `cc`): `canPlace(c)` = any unfilled cell of colour `c` 4-adjacent to a filled cell. `pick(c, n)` = BFS over unfilled cells of colour `c` starting from the frontier, ordered by distance (this produces the "grow from edge" wave in the reference), taking the first `n`. It is unit-tested (vitest) against both level JSONs.
- **`BoardModel`** (plain TS): cell grid, filled set, frontier cache; `markFilled(cells)` updates the frontier incrementally and emits `FrontierChanged`.
- **`BoardBuilder`** (`cc.Component`): instantiates pooled cell prefabs in a grid, applies the palette/grayscale material via `PaletteService`, uses GPU instancing (one shared mesh and material per state) so ~750 cells stay in a few draw calls. `teardown()` returns cells to the pool for the level switch.
- **`CellView`** (`cc.Component`): 4 states (unfilled-gray / hint-dot / frontier-outline / filled). `popFill(delay)` does a scale-from-0 punch, colour set on land.
- **`BlockQueueController`** (`cc.Component`): 4 column stacks; `take()` removes the front block and slides the next forward; tap input goes through `InputService` (per `code-quality-cocos.md`).
- **`TrayController`** (`cc.Component`): N slots (capacity param, default 5); `add()` flies the block into the first free slot; on `FrontierChanged`, slots whose colour became usable glow and become tappable.
- **`LevelFlowController`**: builds the order from the param, runs `startLevel(i)`; between levels it tears the board down and rebuilds it with a short camera/board transition and no endcard.
- **`StoreTriggerController`**: extends the template's `MarketOptionsArbiter` pattern with two triggers the template lacks: *X blocks in tray* (`TrayChanged`) and *X% revealed* (`FillProgress`).

### Feedback matrix (per `game-feedback-mandate.md`)

| Action | Class | visual | vfx | sfx | haptic | ui | Variants |
|---|---|---|---|---|---|---|---|
| Tap block | input-ack | squash on block <100 ms | — | tap click (−18 LUFS) | waiver¹ | — | — |
| Block fills picture | valid-action | fly arc + cell wave pop | sparkle burst | fill whoosh (−18 LUFS) | — | — | ≥3 pitch variants (fires >3×/level) |
| Block to tray | invalid-action | shake + fly to slot | — | soft "thunk" (−18 LUFS) | waiver¹ | slot highlight | — |
| Picture reveal step | progress | cells settle | — | covered by fill sfx | — | reveal % (store trigger only, no HUD in ref) | — |
| Level 1 complete | milestone | picture bounce + transition | confetti | jingle (−10 LUFS) | — | headline swap | — |
| Level 2 complete / tray full | terminal | celebration / gray-out | confetti / — | win / lose sting (−6 LUFS) | — | endcard / fail card | — |

¹ `feedback-waiver: no haptic mechanism in kit; target is H5/web where navigator.vibrate is unsupported (iOS Safari/WebKit) or gesture-gated`; covered by a stronger visual beat.

## 9. Estimate — full line-item breakdown & totals

> `Reuse`: ●=inherited (near-zero) · ◐=framework-assisted · ○=custom. **Base = O = L; P = worst-case.** `Type`: code = gameplay module (maps to a §5 class) · prod = cross-cutting production package.

| # | Module / Package (classes) | Type | Reuse | Scope billed | O | **L** | P |
|---|---|---|---|---|---|---|---|
| | **— Gameplay code modules —** | | | | | | |
| 1 | bootstrap + `GameplayController` | code | ◐ | Template bootstrap ≈ 0.5 h (anchor); flow states load/play/transition/win/lose + module init order, ~150 lines, part debug-bound (0.75 h authoring + 0.75 h allowance) | 2 | **2** | 2.5 |
| 2 | `level-data` (`LevelDataProvider`, `LevelSchema`, `LevelValidator`) | code | ◐ | Schema types, loader, order param, count validator; ~150 lines, typing-bound; + test JSON stubs until the client's arrive | 1 | **1** | 1.5 |
| 3 | `board` (`BoardBuilder`, `BoardModel`, `CellView`) | code | ○ | Grid→world, pooled instanced cubes, 4 cell states, frontier cache, teardown; ~300 lines, part debug-bound (1.5 h + 1.5 h). Risk: instancing + per-cell state materials | 3 | **3** | 4.5 |
| 4 | `fill` (`FillResolver`, `FillAnimator`) | code | ○ | Frontier BFS, usability test, wave ordering + vitest; ~250 lines, part debug-bound (1.25 + 1.25) + 0.5 h matching the reference fill order by eye | 3 | **3** | 4.5 |
| 5 | `block-queue` (`BlockQueueController`, `BlockView`) | code | ○ | 4 column stacks, front/next display, number label, tap input, fly-to-board/tray path; ~250 lines, part debug-bound | 2.5 | **2.5** | 4 |
| 6 | `tray` (`TrayController`, `TraySlotView`) | code | ◐ | Slot row pattern from HexaBugFlow `TrayBuilder`; delta = add/compact/remove, usable highlight, tap-to-return; ~150 lines, part debug-bound | 1.5 | **1.5** | 2 |
| 7 | `objective` (`ObjectiveController`) | code | ◐ | Reveal ratio, correct count, win at 100%, lose on full tray; ~100 lines, typing-bound | 0.5 | **0.5** | 0.5 |
| 8 | `level-flow` (`LevelFlowController`) | code | ○ | P1→P2 seamless / order param, board swap + transition; ~120 lines, part debug-bound. No prior art in corpus | 1.25 | **1.25** | 2 |
| 9 | `palette` (`PaletteService`, `GrayscaleMaterial`) | code | ◐ | Grayscale via material swap (LordMobile `_applyGrayscale` pattern); runtime palette recolour for the Color Change param (COMPLEX param); ~80 lines + material variant | 1 | **1** | 1.5 |
| 10 | `camera-rig` (`CameraRig`) | code | ◐ | Camera-fit pattern from BeeParty; delta = 3D framing for 2 grid sizes × 2 orientations + angle/zoom params; ~120 lines, tuned-by-eye (0.6 + 1.2) | 2 | **2** | 3 |
| 11 | `tutorial` (`TutorialController`) | code | ◐ | incl. FingerTutorial wiring ≈ 1 h (anchor) + 0.5 h deviation: target = first currently usable block, re-shown after idle (hand timing param) | 1.5 | **1.5** | 2 |
| 12 | `store-triggers` (`StoreTriggerController`) | code | ◐ | incl. RedirectService/MarketOptionsArbiter hooks ≈ 1 h (anchor: taps, seconds, correct, level 1/2 complete) + 0.5 h for 2 non-template triggers (blocks in tray, % reveal) | 1.5 | **1.5** | 2 |
| 13 | `hud` (`HeadlineView`) | code | ● | Headline text per level (P2 "%1" text); one binding | 0.25 | **0.25** | 0.25 |
| 14 | `end-cards` | code | ● | Wire endcard + fail card | 0.5 | **0.5** | 0.5 |
| 15 | `audio` | code | ● | ~6 hooks (music, tap, fill, to-tray, level complete, lose) at ~5 min each | 0.5 | **0.5** | 0.5 |
| 16 | `vfx` (`VFXController`) | code | ◐ | Signal → pooled effect spawn for 3 effects; ~80 lines | 0.5 | **0.5** | 0.5 |
| | **Subtotal — code** | | | | 22.5 | **22.5** | 31.75 |
| | **— Production packages —** | | | | | | |
| 17 | Art production (3D) | prod | ◐ | Placeholder-first: cell cube, block cube + number label, tray slot, board frame (4 models) import + materials; default lighting 1 h; one swap to the client's final art 1 h | 4 | **4** | 5 |
| 18 | 3D optimization (asset-side) | prod | ◐ | Low-poly rigid meshes: mesh/texture compression | 1 | **1** | 1 |
| 19 | FPS optimization | prod | ○ | ~750 cells/level: instancing/batching verification, per-cell tween budget during fill waves, low-end profiling, 2 orientations | 4 | **4** | 6 |
| 20 | Build size optimization | prod | ○ | 3D + ~6 SFX + music under the 4.8 MB cap; engine module pruning, compression, single HTML | 2.5 | **2.5** | 4 |
| 21 | VFX assets | prod | ○ | 3 effects × 1 h: fill sparkle, level confetti, tray puff | 3 | **3** | 6 |
| 22 | Polish (juice + shading) | prod | ○ | Juice 5 h: tap squash, fly arc, fill wave pop, tray in/out, invalid shake, level celebration, V1 fill variants. Shading 3 distinct models (cell, block, tray/frame) × 2 h = 6 h + 1 h dynamic-colour surcharge (palette recolour + grayscale must read across all colours) | 12 | **12** | 18 |
| 23 | QA | prod | ◐ | 2 orientations × 2 level orders × 7 store triggers × win/lose × key params | 3.5 | **3.5** | 4.5 |
| 24 | Iteration buffer | prod | — | 0.1 × implementation (52.5 h / 76.25 h) | 5.5 | **5.5** | 7.5 |
| | **Subtotal — production** | | | | 35.5 | **35.5** | 52 |
| | **TOTAL — hours** | | | | 58 | **58** | 83.75 |
| | **TOTAL — dev-days** | | | | 7.25 | **7.25** | 10.5 |
| | **TOTAL — weeks** | | | | 1.5 | **1.5** | 2 |

- **Excluded defaults ($0):** parameter dashboard (BG change, text, SFX toggle, colours, camera values, tray capacity, fill speed, hand timing, level order, store thresholds) · all-network SDK/CTA adapters + build export.
- **VFX not double-counted:** code `VFXController` (row 16) · juice feel (row 22) · effect content (row 21) are three separate rows.
- **Reconcile:** code 22.5 + production 35.5 = **58 h** = WP total (WP0 2 · WP4 15.5 · WP1 6 · WP3 1.5 · WP5 0.5 · WP7 0.5 · WP9 3.5 · WP2 1 · FPS 4 · size 2.5 · WP8.1/8.2 12 · WP8.3 3.5 · WP8.4 5.5). Nothing missing or double-counted.
- **Reuse-tag reconcile:** `●` rows are all ≤ 0.5 h. Each `◐` row names its delta. Rows 11 and 12 show their anchors separately. Per-object rows state their counts (row 21: 3 effects; row 22: 3 models). Every code row states volume and cost driver.

## 9b. Client summary — module overview

Testing and client-iteration effort is **included**, distributed into the modules that carry it rather than listed as its own line.

| Module | Hours |
|---|---:|
| Project setup, level flow & win/lose rules | 5.5 h |
| Picture board & colour-fill mechanic | 11 h |
| Block queue & tray | 6 h |
| Camera & portrait/landscape framing | 2 h |
| Tutorial, store triggers & audio | 3.5 h |
| 3D art integration & performance/size optimization | 11.5 h |
| Polish, game feel & visual effects | 18.5 h |
| **Total** | **58 h** |

> **58 h ≈ 7.25 dev-days ≈ 1.5 weeks** (single developer, ideal engineering time).
> Worst case, if the §11 risk items all land badly: ~84 h (≈ 10.5 dev-days).

## 10. Delivery summary

### 10a. Timeline (single dev, serial)

- **Base:** ~7.25 dev-days ≈ 1.5 weeks · **Worst:** ~10.5 dev-days ≈ 2 weeks.
- Ideal engineering time; the calendar below adds one client review round.

**Proposed due dates (working days from kickoff; the client asked to fix the final date up front):**

| Milestone | Content | Due |
|---|---|---|
| **First build** | Both levels fully playable (fill, queue, tray, lose, P1→P2, level-order param, tutorial, store triggers), placeholder art | **Day 4** |
| Client feedback | One consolidated round on feel + parameters | Days 5–6 (client side) |
| **Final delivery** | Final client art swapped in, polish/VFX, optimization under 4.8 MB, QA across all networks | **Day 10** (worst case Day 12) |

Dependency: the final date holds if the client's 2D/3D assets and both level JSONs arrive **by Day 4**. Each day of delay moves the final date by the same amount.

### 10b. Included at no charge

Parameter dashboard (all the requested params above) · all-network SDK/CTA adapters + build export. FPS and build-size optimization are billed (rows 19–20).

### 10c. Headline

> **Base plan: ~58 h ≈ 7.25 dev-days ≈ 1.5 weeks.** Worst-case ~84 h ≈ 2 weeks.
> Kept down by template reuse (flow, CTA, 5 of 7 store triggers, tutorial, audio), studio prior art (tray, grayscale, camera fit) and client-provided level JSON. Pushed up by the bespoke fill mechanic, 3D cube board and ~750-cell performance, and the heavy feel/polish emphasis in the brief.

### 10d. Optional: visual level editor (not included above)

A standalone HTML tool: paint the pixel picture on a grid with a palette, edit the 4 block columns (colour + count), validate (block counts match cells per colour, simulated solvability), import/export JSON in our schema, optional PNG → grid import.

- **+12 h base (≈ 1.5 dev-days) / +18 h worst**, +2 h with PNG import.
- Impact on dates: final delivery moves from **Day 10 → Day 12** (worst Day 14). It can be built in parallel with the client's feedback round if ordered at kickoff.

## 11. Cost concentration & risk hotspots

1. **Polish (12 h → 18 h):** the brief insists on "fast, smooth, responsive and satisfying" fills and tray moves. Feel tuning is by eye, and the palette recolour plus grayscale must read across every colour.
2. **Board + fill (6 h → 9 h):** frontier/wave order must match the reference; instanced cubes with per-state materials in 3D.
3. **FPS optimization (4 h → 6 h):** ~750 3D cells with per-cell pop tweens during fill waves on low-end devices.
4. **Art arriving late:** placeholder-first protects the first build, but a late or non-game-ready delivery repeats row 17 and threatens the Day 10 date.

## 12. Gut-check

Base 58 h (~7 dev-days) for a 2-level 3D fill-puzzle with heavy juice sits in line with an experienced estimator's number for the genre (1–1.5 weeks). Re-slicing WPs into modules did not change the total (58 h both ways).

## 13. Next step

`/t1k:cook` straight from §§5–8. Order (from the §6 STARTUP trace): level-data → board → fill (with vitest) → block-queue → tray → objective → level-flow → camera-rig → palette → tutorial → store-triggers → hud/end-cards/audio/vfx → art swap → polish → FPS/size optimization → QA. When the repo is scaffolded, promote §§5–8 to `docs/system-design.md` with `state: to-be`, and flip it to `converged` when the build lands.

**Open confirmation items for the client:** frontier seed rule (`seedFilled` vs "bottom row"); lose presentation (fail card vs direct store); final art delivery date.
