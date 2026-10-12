# Loop Hole (Hole-Sort) Playable — System Design & Development Estimate

**Date:** 2026-10-12   **Engine:** Cocos Creator 3.8.7 (3D)   **Built on:** CocosPlayableAdsTemplate
**Source spec:** `brief.txt`, `LOHO091026_P1_Con1.mp4`, `LOHO091026_P2_Con1.mp4` (1080x1920, 37.2 s each). Client level files: promised, **sample not yet received**.
**Basis:** 1 dev · ideal hours · 8h = dev-day · 5 dev-days = week
**Estimate points:** O = L = **Base** (the plan number) · P = **Worst-case** (base + risk on the downside only).

This report IS the system design and the estimate, so `/t1k:cook` can build straight from §§5-8 with no separate planning pass.

## 2. Project understanding & gameplay

**Core-loop confirmation (verbatim, confirmed by the client in the clarification gate on 2026-10-12):**
> The player taps any hole in the bottom grid that has a free route to the arena. That hole travels out of the grid and is deployed into the arena, where it follows its scripted path, eating balls of its own colour while its % rises. It leaves the arena only at 100%, and the grid's queued holes advance. Win = every hole has been used and removed, then the end card with store redirect. There is no lose state. P1 and P2 are one build with different JSON params (grid size, colour set, arena layout). Balls are hybrid: real physics near holes, an instanced field elsewhere.

**Genre:** 3D top-down hole-sort / collect, portrait only, one build, two level configs (P1 priority, P2 follows).

| Question | P1 (`LOHO091026_P1_Con1`) | P2 (`LOHO091026_P2_Con1`) |
|---|---|---|
| On-screen elements | Arena: huge ball field in 2 colours (blue, dark green) with a flat clearing. Grid 6 cols x 4 rows of holes/queued tiles. % label per active hole. Tutorial hand. | Round plate with decorative holes and a ring of balls, 3 colours (red/green/yellow), 3-col grid, close-up camera beats |
| Primary interaction | Tap a reachable hole in the grid | same |
| What a tap does | Hole leaves grid, deploys at arena entry, runs a scripted path, eats matching balls, fill % rises | same |
| Win | All holes used and removed | same |
| Lose | none | none |
| Redirect | End card CTA; in-game CTA/click triggers via template RedirectService | same |

**Not visible in the clips (our call under reference authority A, listed for the client to confirm):** exit edge used for route reachability (top edge facing the arena), feedback on a blocked tap, a single hole capacity for all holes, how a path is assigned to a hole, P2 close-up beats being scripted.

## 3. Scoping assumptions

| Driver | Decision | Effect |
|---|---|---|
| Starting point | Reuse framework (CocosPlayableAdsTemplate + `PLAGameFoundation`), not greenfield | `●`/`◐` discounts, §4 |
| Reference authority | **A: the clips exactly.** No original-game features imported | Unseen rules are ours; listed above |
| Engine | 3D, perspective/orthographic selectable by param | WP1/2/8.2 3D block |
| Balls | **Hybrid physics**: real rigid bodies only near active holes, instanced static field elsewhere, activated on approach | Largest risk, `ball-field` |
| Level data | **Client supplies level files** (format TBD, sample requested) | `level-data` decoder, P widened |
| Orientation | Portrait only | single framing pass |
| P2 | Same build, different JSON params; new arena layout type + close-up beats billed as the P2 delta | §9b column |
| Art | Final, game-ready, rigid/static meshes (no skinned-mesh penalty) | no WP2 skinned surcharge |
| Dynamic colour | Yes: colours/materials customisable at runtime (brief) | shading surcharge, §9 |
| Grayscale | Full-scene post pass, not a value toggle | billed custom pass |
| Parameters | Standard dashboard, **not estimated**. COMPLEX slots billed (material swap, custom font, GIF, gradient, grayscale) | §9 `appearance`, `hud`, `intro` |
| Reference build | Clips only, no playable reference build | worst-case widened on bespoke rows |
| Team | Single dev, serial | totals are wall-clock |
| Visual level editor | **Not requested** (client left it unticked) | WP10 excluded |

**Out of scope:** visual level editor, landscape, lose/fail state, original-game meta features, localization, remote config, analytics beyond hooks, steering/drag input on holes, full tens-of-thousands rigid-body physics.

## 4. Reuse basis

`discovery: live-corpus` (Knowledge MCP `doc_search`, engine=cocos, 15 capability queries, 2026-10-12). The corpus slice swept: Cocos assemblies only. Project assemblies (`Cocos*PLA`) are **port-the-pattern, nothing to install**.

| Need | Member(s) found | Tag | Feeds |
|---|---|---|---|
| Flow states, win/end-card routing | `GameFlow`/`GameFlowState` (H5CocosTemplate), `GameView.onWin` / `EndCardView` (PlayableAdsTemplate) | ● | `GameplayController`, `end-cards` |
| Parameters / dashboard | `ParameterController`, `PlayableParameterUpdates`, `validateConfig` (PlayableParamterTool) | ● | all modules |
| Camera params | `MainCameraParameter`, `CameraConfig` (projection, fov/orthoHeight), `applyCameraComponentParams` | ● | `camera-rig` |
| Background image / colour | `BackgroundImageParameter`, `BackgroundColorParameter` | ● | `appearance` |
| Store redirect / CTA | `RedirectService`, `ClickToStoreHandler`, `EndCardView.actionButton` | ● | `end-cards` |
| Tutorial hand | `FingerTutorial`, `FingerAction.TAP`, `TutorialStepConfig.finger` (assembly `tutorials`) | ● | `tutorial` |
| Pooling | `ObjectPoolManager.Spawn` (assembly `objectPool`) | ● | `ball-field`, `vfx` |
| Audio | `AudioService.playMusic`, `onUpdateVolume` (assembly `gameControl`) | ● | `audio` |
| Hole absorb mechanic | `CocosAllInHolePLA`: `HoleCollectionModule.suction/consume/consumeSunk`, label reveal, progress ring | ◐ port | `hole-agent`, `ball-field`, `hud` |
| Grid route / path clear | `CocosHexaBugFlowPLA.planReturnPath`, `smoothCellPath`; `CocosBeePartyPLA.isSitPathClear` | ◐ port | `hole-grid` |
| Entrance video | `IntroSequence`, `EntranceVideoView` (TimelineUpPLA / BounceDefendersPLA / BlastPartyPLA) | ◐ port | `intro` |
| Top banner + custom fonts | `RollicTopBannerView.CUSTOM_FONTS`, `RollicTopBannerParameter` | ◐ port | `hud` |
| Static-then-dynamic bodies | `CocosBlastPartyPLA` Duck body (spawned STATIC, flipped DYNAMIC on activation wavefront) | ◐ pattern | `ball-field` |
| 3D material slot swap | param tool supports sprite/texture replace only | ○ | `appearance` |
| GIF playback | no animated-GIF hit (only sprite-sheet slicer) | ○ | `intro` |
| Grayscale full-scene pass | no hit (`Sprite.grayscale` is 2D-sprite only, `ScreenEffects` is overlays) | ○ | `appearance` |
| Gradient background | no hit | ○ | `appearance` |
| Tens-of-thousands ball field with activation zone | no hit at this density | ○ | `ball-field` |

**Stays custom:** the grid reachability + hole deployment, scripted-path hole agent, the hybrid ball field, level decode for the client files, grayscale pass, GIF player, material slots, 3D shading polish.

**Capability caveat:** `AllInHolePLA` holes are **stick-steered** (`stepHoleMotion`). Only suction, consume and label logic are reusable; the scripted path follower is new.

## 5. Module architecture (system design)

`[Name-Project]` is a placeholder. **Rename it to the real project folder before implementing.** The doc `state:` is `to-be`.

```
assets/scripts/[Name-Project]/
├── GameplayController.ts
├── level-data/   LevelConfig, LevelDecoder, LevelSelector
├── hole-grid/    GridModel, RouteResolver, GridBuilder, GridView, HoleToken, GridController
├── hole-agent/   HoleAgent, HolePathFollower, HoleSpawner, HoleController
├── ball-field/   BallFieldBuilder, BallFieldLayoutRegions, BallFieldLayoutRing, ActivationZone,
│                 SuctionSolver, BallPool, BallFieldController
├── objective/    ObjectiveController
├── hud/          HoleLabel, TopBannerView, HudController
├── camera-rig/   CameraRig, CameraBeats
├── intro/        IntroSequence, EntranceVideoView, GifPlayerView
├── tutorial/     TutorialController
├── end-cards/    WinEndCardView
├── audio/        AudioHooks
├── appearance/   BackgroundGradient, MaterialSlots, GrayscalePass, AppearanceController
└── vfx/          VFXController
```

**Root:** `assets/scripts/[Name-Project]/`

| Module | Files | Responsibility | Upstream | Status |
|---|---|---|---|---|
| `level-data` | `level-data/LevelConfig.ts`, `level-data/LevelDecoder.ts`, `level-data/LevelSelector.ts` | What does this level contain, decoded from the client file? |  | TBD |
| `hole-grid` | `hole-grid/GridModel.ts`, `hole-grid/RouteResolver.ts`, `hole-grid/GridBuilder.ts`, `hole-grid/GridView.ts`, `hole-grid/HoleToken.ts`, `hole-grid/GridController.ts` | Which holes can the player launch, and in what order? |  | TBD |
| `hole-agent` | `hole-agent/HoleAgent.ts`, `hole-agent/HolePathFollower.ts`, `hole-agent/HoleSpawner.ts`, `hole-agent/HoleController.ts` | Where is each deployed hole and how full is it? |  | TBD |
| `ball-field` | `ball-field/BallFieldBuilder.ts`, `ball-field/BallFieldLayoutRegions.ts`, `ball-field/BallFieldLayoutRing.ts`, `ball-field/ActivationZone.ts`, `ball-field/SuctionSolver.ts`, `ball-field/BallPool.ts`, `ball-field/BallFieldController.ts` | Which balls exist, which are live physics, and which got eaten? |  | TBD |
| `objective` | `objective/ObjectiveController.ts` | Are all holes used and gone? |  | TBD |
| `hud` | `hud/HoleLabel.ts`, `hud/TopBannerView.ts`, `hud/HudController.ts` | What text and % does the player read? |  | TBD |
| `camera-rig` | `camera-rig/CameraRig.ts`, `camera-rig/CameraBeats.ts` | How is the scene framed, and when does it cut in? |  | TBD |
| `intro` | `intro/IntroSequence.ts`, `intro/EntranceVideoView.ts`, `intro/GifPlayerView.ts` | What plays before gameplay? |  | TBD |
| `tutorial` | `tutorial/TutorialController.ts` | Which hole does the hand point at? | `@playablelabs/game-foundation` | TBD |
| `end-cards` | `end-cards/WinEndCardView.ts` | What shows after the win and where does the CTA go? |  | TBD |
| `audio` | `audio/AudioHooks.ts` | Which sound plays on which event? | `@playablelabs/game-foundation` | TBD |
| `appearance` | `appearance/BackgroundGradient.ts`, `appearance/MaterialSlots.ts`, `appearance/GrayscalePass.ts`, `appearance/AppearanceController.ts` | What does the scene look like given the params? |  | TBD |
| `vfx` | `vfx/VFXController.ts` | How do events look visually? |  | TBD |
| `GameplayController` | `GameplayController.ts` | Orchestrator: wires signals to GameView/CTA |  | TBD |

Rules: modules never import each other (only `SignalBus`); only `GameplayController` calls `GameView`/`CTAService`. Every runtime visual is a **prefab** authored via the Cocos MCP (`prefab-only-construction-cocos.md`): `Arena`, `ArenaPlate`, `GridPanel`, `HoleToken`, `Hole`, `Ball`, `HoleLabel`, `WinEndCard`, plus pooled VFX prefabs.

**Upstream column:** `@playablelabs/game-foundation` is the package that ships the `tutorials` and `gameControl` subsystems per the knowledge-sweep rule. Confirm the installed version at `cpm.playablelabs.ai`; no other upstream is claimed.

## 6. Communication flow

```
-- STARTUP --------------------------------------------------------------
  LoadingView  (wait ParametersReady + AllAsyncParametersReady)
  GameplayController.startLevel()
    ├─> LevelSelector.select(param.levelId)  -> LevelDecoder.decode()    # config first, everything reads it
    ├─> AppearanceController.apply()         # bg, materials, grayscale before first frame
    ├─> CameraRig.apply()                    # persp/ortho, pos/rot/zoom
    ├─> BallFieldController.build(config)    # field must exist before holes can eat
    ├─> GridController.build(config)         # grid + queues
    ├─> HoleController.init(config)          # paths registered
    ├─> HudController.init()                 # banner text/font
    └─> IntroSequence.play()                 # video or GIF or none -> IntroFinishedSignal

-- PLAYER INPUT ---------------------------------------------------------
  GridView.onTouchEnd(cell)
    └─> SignalBus.fire(TapCellSignal{cellId})
          └─> GridController.onTapCell()
                ├─ RouteResolver.isReachable(cell)? no -> fire(TapBlockedSignal) -> VFX/Audio shake
                └─ yes: GridModel.vacate(cell); HoleToken travels route to arena entry
                      └─> fire(HoleLaunchedSignal{holeId,color,cell})
                            ├─> HoleController.onLaunched -> HoleSpawner.spawn -> fire(HoleDeployedSignal)
                            ├─> TutorialController.onFirstInteraction (hide hand)
                            └─> VFXController / AudioHooks

-- GAME LOOP (per frame) ------------------------------------------------
  HoleController.tick(dt)
    ├─ HolePathFollower.step(dt)           # scripted waypoints, speed from config
    └─> fire(HoleMovedSignal{holeId,pos,radius})
  BallFieldController.tick(dt)
    ├─ ActivationZone.update(holes)        # static -> dynamic near each hole
    ├─ SuctionSolver.step(dt)              # pull/swallow balls of the hole's colour
    └─> fire(BallsConsumedSignal{holeId,count})
  HoleController.onBallsConsumed -> fill += count/capacity
    ├─> fire(HoleFillChangedSignal{holeId,fill})   # HudController updates label
    └─ if fill >= 1: fire(HoleCompletedSignal{holeId}) -> recycle hole

-- WIN ------------------------------------------------------------------
  ObjectiveController: queued=0 AND active=0
    └─> fire(WinSignal) -> GameplayController.onWin()
          └─> GameView.instance.onWin() -> WinEndCardView -> CTA -> CTAService
```

## 7. Signal contracts

| Signal | Payload | Fired by | Consumed by | Trigger |
|---|---|---|---|---|
| `TapCellSignal` | `cellId` | `GridView` | `GridController` | Tap on a grid cell |
| `TapBlockedSignal` | `cellId` | `GridController` | `VFXController`, `AudioHooks` | Tapped hole has no route |
| `HoleLaunchedSignal` | `holeId, color, cell` | `GridController` | `HoleController`, `TutorialController`, `VFXController`, `AudioHooks` | Token reached arena entry |
| `QueueAdvancedSignal` | `column` | `GridController` | `TutorialController`, `VFXController` | Column queue shifted up |
| `HoleDeployedSignal` | `holeId, color, position` | `HoleController` | `BallFieldController`, `HudController`, `VFXController`, `AudioHooks` | Hole spawned in arena |
| `HoleMovedSignal` | `holeId, position, radius` | `HoleController` | `BallFieldController`, `HudController` | Per frame along path |
| `BallsConsumedSignal` | `holeId, count, color` | `BallFieldController` | `HoleController`, `VFXController`, `AudioHooks` | Balls swallowed this step |
| `HoleFillChangedSignal` | `holeId, fill` | `HoleController` | `HudController` | Fill % changed |
| `HoleCompletedSignal` | `holeId` | `HoleController` | `ObjectiveController`, `BallFieldController`, `CameraRig`, `VFXController`, `AudioHooks` | Fill reached 100% |
| `IntroFinishedSignal` | none | `IntroSequence` | `GameplayController` | Video/GIF done or skipped |
| `WinSignal` | none | `ObjectiveController` | `GameplayController` | Queues empty, no active holes |

No module imports another; all coupling is via `SignalBus`.

## 8. Class summary, level-data schema, class reference

### Class summary

| Class | Module | Key properties | Key methods |
|---|---|---|---|
| `LevelConfig` | `level-data` | grid, palette, paths, arena, ballField | (data) |
| `LevelDecoder` | `level-data` | schema version | `decode(raw)`, `validate()` |
| `LevelSelector` | `level-data` | levelId param | `select()` |
| `GridModel` | `hole-grid` | cells, queues | `vacate()`, `advance(col)`, `isEmpty()` |
| `RouteResolver` | `hole-grid` | exit edge | `isReachable(cell)`, `route(cell)` |
| `GridController` | `hole-grid` | model, view, token prefab | `build()`, `onTapCell()` |
| `GridView` / `HoleToken` | `hole-grid` | cell nodes | `renderCell()`, `travel(route)` |
| `HoleAgent` | `hole-agent` | color, fill, capacity, pathId | `addBalls(n)`, `isFull()` |
| `HolePathFollower` | `hole-agent` | waypoints, speed | `step(dt)` |
| `HoleController` | `hole-agent` | active agents | `onLaunched()`, `tick(dt)` |
| `BallFieldController` | `ball-field` | static layer, live pool | `build()`, `tick(dt)` |
| `ActivationZone` | `ball-field` | radius per hole | `update(holes)` |
| `SuctionSolver` | `ball-field` | pull strength | `step(dt)` |
| `ObjectiveController` | `objective` | queuedCount, activeCount | `onLaunched()`, `onCompleted()` |
| `VFXController` | `vfx` | pooled particle refs | `on[Signal]()` |
| `GameplayController` | root | all module refs | `startLevel()`, `onWin()` |

### Level-data schema (PlayableConfig JSON param, decoded from the client's files)

**Flagged assumption:** the client file format is unknown. The shape below is OUR target schema; the decoder maps the client's files onto it.

```json
{
  "levelId": "p1",
  "palette": ["#2E9BFF", "#1F6B4F"],
  "grid": { "cols": 6, "rows": 4,
            "cells": [["g","b","g","b","g","b"], ["b","g","b",null,"b","g"]] },
  "holeCapacity": 100,
  "paths": [ { "id": "A", "waypoints": [[2.0,-3.0],[1.0,0.0],[-1.5,2.5]], "speed": 1.6 } ],
  "pathAssign": "cycle",
  "arena": { "layout": "regions|ring", "clearing": { "shape": "rounded", "size": [4.0,4.0] },
             "plate": { "radius": 3.0, "holes": 6 } },
  "ballField": { "radius": 0.11, "regions": [ { "color": 0, "polygon": [[...]] } ],
                 "activeBudget": 400, "ring": { "inner": 3.1, "outer": 5.0, "sectors": 3 } },
  "camera": { "projection": "perspective", "position": [0,12,0], "euler": [-90,0,0], "fov": 40 },
  "tutorial": { "enabled": true }
}
```

| Property | Type | Default | Description |
|---|---|---|---|
| `palette` | `string[]` | 2 colours (P1), 3 (P2) | Colour id -> display colour; ids index `cells`/`regions` |
| `grid.cells` | `(string|null)[][]` | none | Hole colour per cell; `null` = empty cell (route space) |
| `holeCapacity` | `number` | 100 | Balls a hole needs to reach 100% |
| `paths[]` | `Path[]` | none | Scripted waypoints per hole, speed |
| `pathAssign` | `cycle|byCell` | `cycle` | How a launched hole picks a path |
| `arena.layout` | `regions|ring` | `regions` | P1 free field vs P2 ring around a plate |
| `ballField.activeBudget` | `number` | 400 | Max live rigid bodies at once |

### Class reference (bespoke modules)

#### `RouteResolver` - `hole-grid/RouteResolver.ts`
Static helper. Decides whether a cell has a free route to the arena.

| Type | Name | Description |
|---|---|---|
| prop | `exitEdge: 'top'` | Edge of the grid facing the arena (assumption) |
| fn | `isReachable(model, cell): boolean` | BFS over empty cells to the exit edge |
| fn | `route(model, cell): Cell[]` | Shortest cell path, smoothed to corners (pattern: `HexaBugFlowPLA.planReturnPath`) |

#### `GridController` - `hole-grid/GridController.ts`
Cocos Component. Owns grid state and deployment.

| Type | Name | Description |
|---|---|---|
| prop | `model: GridModel` | Cells + per-column queues |
| fn | `build(config)` | Spawns `GridPanel` prefab cells via `GridBuilder` |
| fn | `onTapCell(sig)` | Route check, vacate, token travel, queue advance |
| signal | fires `HoleLaunchedSignal`, `QueueAdvancedSignal`, `TapBlockedSignal` |  |

#### `HoleController` / `HolePathFollower` - `hole-agent/`
Cocos Component + ITickable.

| Type | Name | Description |
|---|---|---|
| prop | `agents: HoleAgent[]` | Active holes |
| fn | `onLaunched(sig)` | Spawns a `Hole` prefab at the arena entry |
| fn | `tick(dt)` | Steps each follower; fires `HoleMovedSignal` |
| fn | `onBallsConsumed(sig)` | `fill += count/capacity`; fires fill/complete signals |
| signal | fires `HoleDeployedSignal`, `HoleMovedSignal`, `HoleFillChangedSignal`, `HoleCompletedSignal` |  |

#### `BallFieldController` - `ball-field/BallFieldController.ts`
Cocos Component + ITickable. Hybrid physics, highest risk.

| Type | Name | Description |
|---|---|---|
| prop | `staticLayer` | Instanced, non-simulated balls (the vast majority) |
| prop | `livePool: BallPool` | Pooled rigid-body balls, capped by `activeBudget` |
| fn | `build(config)` | Fills field via `BallFieldLayoutRegions` or `BallFieldLayoutRing` |
| fn | `tick(dt)` | `ActivationZone.update`, `SuctionSolver.step`, fires consume signals |
| signal | fires `BallsConsumedSignal` |  |

Framework-inherited classes (`GameView`, `EndCardView`, `AudioService`, `FingerTutorial`, `RedirectService`) are one-liners and are not re-specified.

## 9. Estimate — full line-item breakdown

> `Reuse`: ●=inherited (near-zero) · ◐=framework-assisted · ○=custom. **Base = O = L; P = worst-case.** `Type`: code = gameplay module (maps to a §5 module) · prod = cross-cutting production package. Hours are ideal engineering hours, rounded to 0.5.

| # | Module / Package (classes) | Type | Reuse | Scope billed | O | **L** | P |
|---|---|---|---|---|---:|---:|---:|
| | **— Gameplay code modules —** | | | | | | |
| 1 | `GameplayController` + flow | code | ◐ | Add game-specific states (intro, play, win) on template `GameFlow`. ~60 lines, typing-bound. | 0.5 | **0.5** | 0.5 |
| 2 | `level-data` (LevelConfig, LevelDecoder, LevelSelector) | code | ○ | Decode the **client's level files** into `LevelConfig`; select P1/P2 by param; hand-author both level configs. ~250 lines → 1.25 h authoring + 1.25 h allowance (part debug-bound, unknown format) + 0.5 h P1/P2 config authoring. Part of the §3 level-generator anchor (see reconcile). | 3.0 | **3.0** | 5.0 |
| 3 | `hole-grid` (GridModel, RouteResolver, GridBuilder, GridView, HoleToken, GridController) | code | ○ | Grid build from config (1.5 h, generator half), route-reachability BFS, token travel along route, queue advance, blocked-tap. ~400 lines → 2.0 h authoring + 2.0 h allowance (part debug-bound) + 0.5 h blocked-tap feedback. Pattern port from `HexaBugFlowPLA`/`BeePartyPLA` (not installable). | 4.5 | **4.5** | 7.0 |
| 4 | `hole-agent` (HoleAgent, HolePathFollower, HoleSpawner, HoleController) | code | ○ | Scripted-path follower, deploy at arena entry, capacity/fill, finish-at-100% lifecycle, concurrency. ~350 lines → 1.75 h authoring + 3.5 h allowance (tuned-by-eye: pacing, path shapes). Consume rules ported from `AllInHolePLA`; its stick steering is NOT reused. | 5.5 | **5.5** | 8.5 |
| 5 | `ball-field` (BallFieldBuilder, 2 layouts, ActivationZone, SuctionSolver, BallPool, BallFieldController) | code | ○ | Hybrid physics: instanced static field, static→dynamic activation near holes, colour-matched suction, count/recycle, active budget; builder (3.5 h, generator half) + P2 ring layout (1.5 h). ~750 lines → 3.75 h authoring + 7.5 h allowance (tuned-by-eye + debug-bound: perf and stability). Pattern from `BlastPartyPLA` static→dynamic bodies. | 11.5 | **11.5** | 18.5 |
| 6 | `objective` (ObjectiveController) | code | ◐ | Win when queued=0 and active=0 across concurrent holes. ~100 lines → 0.5 h + 0.5 h for the concurrency/edge cases. | 1.0 | **1.0** | 1.5 |
| 7 | `hud` (HoleLabel, TopBannerView, HudController) | code | ◐ | Per-hole % billboard following the hole (pattern `AllInHolePLA.sizeBillboard`), top-banner text + **custom font upload** (port `RollicTopBannerView`). ~200 lines → 1.0 h + 1.0 h allowance (font load, billboard) + 0.5 h. HUD binding itself at anchor, not hours. | 2.5 | **2.5** | 3.0 |
| 8 | `camera-rig` (CameraRig, CameraBeats) | code | ◐ | Param wiring of `MainCameraParameter` ≈ 0.5 h (●); portrait framing for both arenas, tuned-by-eye 1.0 h (○); P2 scripted close-up beats 1.0 h (◐). | 2.5 | **2.5** | 3.5 |
| 9 | `intro` (IntroSequence, EntranceVideoView, GifPlayerView) | code | ◐/○ | Entrance video ≈ 1.5 h (◐, port of `IntroSequence`); **GIF player** with free resize/position/dimensions 2.5 h (○, ~200 lines, part debug-bound: frame decode/atlas). | 4.0 | **4.0** | 5.5 |
| 10 | `tutorial` (TutorialController) | code | ● | `FingerTutorial` wiring ≈ 1.0 h (anchor) + 0.5 h retarget the hand to the next reachable hole. Enable/disable toggle is a basic param ($0). | 1.5 | **1.5** | 1.5 |
| 11 | `end-cards` (WinEndCardView) | code | ● | End-card subclass wire-only 0.5 h + store-redirect hook ≈ 1.0 h (anchor). | 1.5 | **1.5** | 1.5 |
| 12 | `audio` (AudioHooks) | code | ● | ~5 hooks (music, deploy, eat loop, complete, win) ≈ 0.5 h (5 min each) + replaceable BGM/SFX params 0.5 h. | 1.0 | **1.0** | 1.0 |
| 13 | `appearance` (BackgroundGradient, MaterialSlots, GrayscalePass, AppearanceController) | code | ○ | Gradient background 1.0 h; **COMPLEX material-slot swap** for arena/ball/hole/tile 2.5 h; **grayscale full-scene pass** with toggle 3.0 h (debug-bound: compile read-back). Image/solid bg are ● and $0. | 6.5 | **6.5** | 9.5 |
| 14 | `vfx` (VFXController) | code | ◐ | 5 signal-to-effect hooks + pooled bursts, zero game logic. ~100 lines typing-bound 0.5 h + 0.5 h wiring. Effect content is §9 row 18. | 1.0 | **1.0** | 1.5 |
| | **Subtotal — code** | | | | **46.5** | **46.5** | **68.0** |
| | **— Production packages —** | | | | | | |
| 15 | Art production (WP1): 8 prefabs via Cocos MCP, scene, lighting | prod | ◐ | Prefabs: Arena, ArenaPlate (P2), GridPanel, HoleToken, Hole, Ball, HoleLabel, WinEndCard = 8 x 0.5 h; default lighting 1.0 h; art final. | 6.0 | **6.0** | 7.5 |
| 16 | 3D optimization (WP2, asset side) | prod | ◐ | Ball/hole/plate static meshes: poly trim, texture atlas, instancing material. 3 meshes. No skinned meshes. | 1.5 | **1.5** | 2.0 |
| 17 | Polish (WP8.1 + WP8.2) | prod | ○ | Juice authored once 2.5 h (deploy bounce, eat pop, complete flash, win). Shading: **4 distinct models x 2 h = 8.0 h** (ball, hole, tile token, arena ground/plate) + **1.0 h dynamic-colour surcharge**. | 11.5 | **11.5** | 17.0 |
| 18 | VFX assets (WP9) | prod | ◐ | **4 effects x 1 h:** eat burst, hole-complete ring, deploy puff, win confetti. | 4.0 | **4.0** | 8.0 |
| 19 | QA (WP8.3) | prod | ◐ | P1/P2 x tutorial on/off x video/GIF x bg modes x grayscale x persp/ortho x end card + device checks. | 4.0 | **4.0** | 5.0 |
| 20 | Iteration buffer (WP8.4) | prod | ○ | 0.1 x total implementation (`estimation-rules.md` § 1a). | 7.5 | **7.5** | 11.0 |
| | **Subtotal — production** | | | | **34.5** | **34.5** | **50.5** |
| | **TOTAL — hours** | | | | **81.0** | **81.0** | **118.5** |
| | **TOTAL — dev-days** (÷8) | | | | **10.25** | **10.25** | **14.75** |
| | **TOTAL — weeks** (÷40) | | | | **2.0** | **2.0** | **3.0** |

**Excluded defaults ($0 line items):** parameter dashboard (all BASIC params: value/colour/text/timing/toggle, Gameplay Parameters, text content/font size/position, tutorial toggle) · all-network SDK/CTA adapters + build export · image/solid background parameters · draw-call/batching · size-budget compliance (**4.8 MB hard cap recorded as the delivery-risk driver**, esp. GIF/entrance video) · low-end device profiling.

**VFX not double-counted:** `VFXController` (row 14, wiring) vs Polish feel (row 17, timing/juice) vs VFX asset content (row 18).

**Reconcile:** code 46.5 + production 34.5 = **81.0** = the WP-sizing total (WP0 0.5 · level-data/generator 8.0+1.5 P2 layout · gameplay · WP1 art 6.0 · WP2 1.5 · WP3 tutorial/intro · WP5 1.5 · WP7 1.0 · WP8 polish/QA/buffer). No slice missing or double-counted.

**Reuse-tag reconcile:**
- `●` rows above 0.5 h justify the excess: tutorial 1.5 h = 1.0 h `FingerTutorial` wiring (anchor) + 0.5 h dynamic retarget the framework does not supply; end-cards 1.5 h = 0.5 h subclass + 1.0 h redirect hook (anchor); audio 1.0 h = 0.5 h hooks (anchor) + 0.5 h asset-replace params.
- `◐` rows name their delta (rows 1, 6, 7, 8, 9, 14, 15, 16, 18, 19).
- Per-object-scaled: shading **4 models x 2 h**; VFX **4 effects x 1 h**; prefabs **8 x 0.5 h**.
- Level-generator anchor (`estimation-rules.md` § 3, placement + runtime colour ≈ 8 h): `level-data` 3.0 + grid builder 1.5 + ball-field builder 3.5 = **8.0 h**. The extra 1.5 h on row 5 is P2's second layout type (ring), billed once.
- Sizing basis: every code row states volume and driver above. Rows 4 and 5 are deliberately high against line count; named risk: tuned-by-eye pacing and physics stability/perf (`estimation-rules.md` § 1b allowance).

## 9b. Client summary — module overview

Testing and client-iteration effort is **included** — distributed into the modules that carry it rather than listed as its own line.

| Module | Playable 1 | Playable 2 (delta) | Both |
|---|---:|---:|---:|
| Core framework, flow, win & end card | 4.0 h | 0.0 h | 4.0 h |
| Level data & loading | 2.5 h | 0.5 h | 3.0 h |
| Hole grid, tap & route logic | 4.5 h | 0.0 h | 4.5 h |
| Hole movement & collection | 6.5 h | 0.0 h | 6.5 h |
| Ball field & physics | 14.0 h | 2.0 h | 16.0 h |
| Camera, intro video/GIF & tutorial | 8.5 h | 1.0 h | 9.5 h |
| UI, text, banner & appearance options | 10.5 h | 0.0 h | 10.5 h |
| Art, optimization, effects & polish | 25.5 h | 1.5 h | 27.0 h |
| **Total** | **76.0 h** | **5.0 h** | **81.0 h** |

> **Both playables: 81 h ≈ 10.25 dev-days ≈ 2.0 weeks** (single developer, ideal engineering time). P1 alone: 76 h ≈ 9.5 dev-days.
> Worst case if the §11 risk items land badly: ~118.5 h ≈ 3 weeks.
> The estimate depends on receiving the client level-file sample; an unreadable format widens the level-data and ball-field rows.

## 10. Delivery summary

### 10a. Timeline (single dev, serial)
- **Base:** ~10.25 dev-days ≈ ~2.0 weeks · **Worst:** ~14.75 dev-days ≈ ~3.0 weeks. P1 can ship first at ~9.5 dev-days; P2 adds ~0.75 dev-day.
- Ideal engineering time; calendar excludes client asset delivery, review cycles, holidays. A second dev could parallelise `ball-field` against `hole-grid`/`intro`/`appearance`.

### 10b. Included at no charge
Parameter dashboard · all-network SDK/CTA adapters and build export · image/solid background parameters · draw-call/batching · size-budget compliance · low-end profiling.

### 10c. Headline
> **Base plan: ~81 h ≈ ~10.25 dev-days ≈ ~2.0 weeks.** Worst-case ~118.5 h ≈ ~3.0 weeks.
> Held down by the template, parameter dashboard and the AllInHole absorb logic; pushed up by the hybrid ball field, the unseen client level format, the grayscale pass and 3D shading polish with runtime colours.

## 11. Cost concentration & risk hotspots

1. **`ball-field` 11.5 h (P 18.5 h):** hybrid physics at clip density. Risks: stability of static→dynamic hand-off, phone frame rate, 4.8 MB cap. The client's "real physics" is implemented as live bodies in the activation zone only; a full-density rigid-body field is out of scope.
2. **Polish 11.5 h (P 17.0 h):** 4 models x 2 h shading, dynamic-colour surcharge (look must read across the customisable palette).
3. **`appearance` 6.5 h (P 9.5 h):** material-slot swap and grayscale pass are both custom and debug-bound.
4. **`level-data` 3.0 h (P 5.0 h):** the client's level format is unseen. A binary/custom format would need a converter and moves the rest of the schedule.
No pre-rigged/animated models, so no skinned-mesh optimization penalty.

## 12. Gut-check

The base (81 h) was checked against a gut figure of about 70-90 h for a 3D tap-to-deploy sort playable with a high-count physics field and a full parameter list. Re-slicing WPs into modules did not move the number (reconcile above). The worst/base ratio is 1.46; risk sits only on the bespoke rows.

## 13. Next step

Run **`/t1k:cook`** from §§5-8. Implementation order from the §6 STARTUP trace: `level-data` → `ball-field` (spike first, it carries the risk) → `hole-grid` → `hole-agent` → `objective`/`GameplayController` → `hud` → `appearance`/`camera-rig` → `intro`/`tutorial`/`end-cards`/`audio` → polish/VFX/QA.
Before building: obtain the client **level-file sample**, confirm the five unseen-rule assumptions in §2, and promote §§5-8 to `docs/system-design.md` (`state: to-be`, flip to `converged` when built).
