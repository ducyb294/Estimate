# LoopHole Playables: System Design & Development Estimate

**Part A: `LOHO091026_P1_Con1` · Part B: `LOHO091026_P2_Con1`.** Combined totals in §1.

**Date:** 2026-10-12   **Engine:** Cocos Creator 3.8.7 (3D, portrait)   **Built on:** CocosPlayableAdsTemplate
**Source spec:** `LoopHole/Client_Requirement.txt`, the two creative clips in `LoopHole/` (P1 clip is primary; P2 clip for differences), Unity reference runner `C:\WORK\VIDEO\LoopHole\Assets\LevelRunner\LoopHoleLevelRunner.cs` (7,783 lines), level data `Level_21.json`, `Level_23.json`, `Level_54.json`. Evidence reports: `LoopHole/_scout/{runner-analysis,level-json-analysis,video-analysis,reuse-sweep}.md`.
**Basis:** 1 dev · ideal hours · 8h = dev-day · 5 dev-days = week
**Estimate points:** O = L = **Base** (plan number) · P = **Worst-case** (base + risk on the downside only).

This report is both the system design and the estimate: a dev (or `/t1k:cook`) builds straight off §§5-8 with no separate planning pass. **Part A covers `LOHO091026_P1_Con1` in full. Part B (after §13) prices `LOHO091026_P2_Con1` as a delta on top of Part A, under stated assumptions because its JSON has not been supplied.**

> **Due date flag (read first):** client wants `LOHO091026_P1_Con1` by **2026-10-16**. That is 5 working days from today. Base is ~14.5 dev-days for one dev. See §10a and §11 for what that means.

---

## 1. Headline (details in §9, §9b, §10; Part B for P2)

| | P1 `LOHO091026_P1_Con1` | P2 `LOHO091026_P2_Con1` (delta, assumptions in Part B) | Both |
|---|---:|---:|---:|
| **Base** | **116.5 h · 14.5 d · 3.0 w** | **21.5 h · 2.75 d · 0.5 w** | **138.0 h · 17.25 d · 3.5 w** |
| Worst-case | 169.5 h · 21.25 d · 4.25 w | 33.5 h · 4.25 d · 0.75 w | 203.0 h · 25.5 d · 5.0 w |

P2 is cheap only because it reuses P1's code and spikes; it is **not** a standalone price. See Part B conditions.

---

## 2. Project understanding & gameplay

**Core-loop confirmation line:** *"Board grid of closed holes; a hole opens when a 4-way path of free tiles reaches row 0 (BFS). Tap an open hole: it flies onto one of 5 slots of a spinning disk inside the bowl; the slot pulls balls of its own colour until its % fills, then closes and frees. Win = no hole left on the board and no slot occupied. Lose = disk full with no collectable ball, 5.5 s countdown. Tap a closed hole = wobble + red flash; tap with disk full = refuse animation."* **confirmed by the requester on 2026-10-12** (clarification gate read-back).

**Genre / structure:** 3D ball-sorting "hole.io" style puzzle, one level, no meta. P1 clip matches **Level_23** (2 colours Green/DarkTeal, 24 holes 12/12, 15,000 balls, 9x8 board with 6 hole columns visible, capture limit 200, no pipes). Models: static meshes, one blend-shape lid mesh; no skeletal characters.

**Six gameplay questions:**

1. **On screen:** top-down-ish portrait camera; bowl of ~15,000 tiny balls (2 colours); flat indigo disk with 5 slots rotating ~65 deg/s; board of 24 coloured holes (closed square lids, round when open); `NN%` world-space labels above occupied slots; tutorial hand. Nothing else (no banner/HUD/counter in the clip).
2. **Primary interaction:** tap a hole on the board.
3. **Each tap:** open hole flies (parabolic arc) to the nearest free slot and lands; closed hole wobbles + flashes red; full disk triggers refuse animation.
4. **Win:** no hole on board and no occupied slot.
5. **Lose:** 4-5 of 5 slots filled, no collectable ball for any assigned colour; 5.5 s countdown with red disk pulse; or board deadlocked with no reachable free slot.
6. **Store redirect:** CTA tap, end card, and a store-only route with no end card (client option).

**Behaviours that the video does not show (taken from the runner, per the authority decision):** win, lose, countdown, end card. The P1 clip is a 21 s loop that hard-cuts back to a fresh board.

---

## 3. Scoping assumptions

| Driver | Decision | Effect |
|---|---|---|
| Starting point | Reuse studio framework + `@playablelabs/*` packages (not greenfield) | Discounts per §4 |
| Reference authority | **Runner is authoritative for rules; P1 clip for look** (camera, colours, HUD); end card = template default | Lose/countdown included though not in clip |
| Level scope | P1 = **Level_23 only**, single level, no level switching | No multi-level controller |
| Simulation | **Sleep/wake position-based solver** on typed arrays, same formulas as runner (centre pull, damping, speed caps, substeps), no PhysX. Balls at rest cost nothing; a captured ball wakes neighbours so the pile collapses as a wave | Matches the video's balls-pile-forward effect; new design, not in runner |
| ECS meaning | Cocos has no built-in ECS. "ECS" here = struct-of-arrays typed arrays + system functions, engine-free (`cc` only in adapters under `engine/cocos/`), per `engine-decouple-cocos.md` | Testable sim, ships `.tests/` |
| Ball rendering | One instanced mesh, per-instance colour/shade, full toon look (outline, gloss, shadow tint) per user decision | Custom `.effect`; heaviest shader risk |
| Hole lid | Keep blend-shape/morph target from the FBX (user decision) | Needs early spike on Cocos web build |
| Art | Reuse Unity FBX/textures; **rebuild shaders as `.effect`** (ShaderGraph has no portable source); VFX rebuilt with Cocos particles | Production packages §9 |
| Audio | Studio/client supplies files; dev only wires | ~8 hooks |
| Orientation | Portrait only (1080x1920 clip) | One layout |
| Parameters | Dashboard + basic params are **default, not estimated**. Client's list (bg modes, camera, tutorial toggle, text/font, entrance video/GIF, grayscale, store redirect) is billed only as implementation wiring in this game | Row P |
| Optimisation | FPS and build-size packages are always billed (`estimation-rules.md` §3b) | Rows 24-25 |

**Out of scope (P1):** pipes + entrance pour (Level_54 only), blockers (mystery/freeze/bomb/pair/key/lock/spawner/panel/elevator, absent from all three levels), standalone level editor (WP10, client supplies JSON), landscape, level switching, ad-network adapters, Unity runner's `R`-restart key. P2 is priced in Part B.

---

## 4. Reuse basis

`discovery: live-corpus` · scope: 47 Cocos assemblies of 108 indexed repos, swept 2026-10-12 · **grep: not run (no local Cocos checkout; every hit is corpus-grounded, not disk-grounded)** · graph: unavailable. The sweep ran on a cheap routed model; package names were resolved with `resolve-stdlib-package.cjs`, but **API fit has not been read on disk**. Re-verify when the project is scaffolded.

| Need | Tag | Found | Feeds module | Delta (what is NOT covered) |
|---|---|---|---|---|
| Board lid-open BFS | ● | `@playablelabs/core-grid-flood` `floodFrom/shortestPath/exposed` (precedent HexaBugFlowPLA `computeAttackable`) | board | wiring: occupancy as passable, row 0 as goals |
| Disk spin + slot occupancy | ● | `@playablelabs/cc-node-spin` `NodeSpinner`, `core-footprint-grid`; `Cocos3DSpinReady` `landingRotation` | disk | none for the spin; labels/close animation custom |
| Parabolic send flight | ● | `core-combat-shapes` `ballisticLaunchAngles`, `mechanic-bounce-projectile` `ProjectileFlight`; AllInHole `aimDotPoses` | hole | easing/land squash tuning |
| Pool | ● | `core-tick-schedule` `FreeList`, `cc-polish-kit` `PolishNodePool` | all | none |
| Timers / countdown | ● | `core-effect-motion` `CountdownTimer`, `DelayedActions` | objective | none |
| Camera shake, juice | ● | `core-effect-motion` `CameraShakeState`, `TweenReactions` | camera, vfx | none |
| Screen adapt | ● | `cc-screen-fit` | camera | none |
| Audio | ● | `cc-lazy-load` `LazyAudio`, `core-polish-profile` `SfxCue` | audio | cues must be loudness-banded per `game-feedback-mandate` |
| Tutorial hand | ● | `core-guide-sequence` `GuideSequencer`, foundation `tutorials` | tutorial | none |
| Instanced balls | ● | `cc-instanced-material`, `core-mesh-builders` `ColoredMeshArrays` | ball-sim (render) | **per-instance** colour/shade (package gives per-vertex colour); unverified |
| Win/lose, end card, CTA | ◐ | `feature-round-outcome`, `core-polish-profile` `PolishDirector`, `BounceDefendersStoreRedirectTriggerCard` | objective, end-cards | store-only route (no end card) not a shipped flag |
| Tap on 3D node | ◐ | foundation `inputService`, `cc-tap-badge` | hole | raycast onto hole meshes, not UI nodes |
| Camera params | ◐ | `cc-screen-fit` `CameraFitWidth`, `core-camera-follow` | camera | XYZ pos/rot, persp/ortho switch, zoom as params |
| Background | ◐ | `cc-polish-kit`, `coverSize` | params | gradient effect is new |
| Grayscale | ◐ | pattern only: LordMobileFuntapPLA `grayEffect`, AllInHolePLA `ref-post-grade.effect` | params | `.effect` + toggle across every custom effect |
| Entrance video / GIF | ◐ | `EntranceVideoView` (BlastParty, TimelineUp); `FrameCycleState` | params | package install name unresolved; GIF decode + resize/position |
| World-space `%` label | ◐ | pattern only: `Billboard`, `WorldHealthBar`, AllInHole `HudFrame.billboard` | disk | no package |
| Toon / outline / gloss `.effect` | ◐ | pattern only: AllInHole `hole-rim`, `ref-outline`, Arrow3D outline | art | author the effects |
| Morph target on FBX | ◐ | `cc-skeletal-pose`, TimelineUp `WoodMorphDriver` | hole | morph-weight path unverified in a playable build |
| Procedural walls | ◐ | `core-height-profile` `buildStripMesh` | board | bevel profile |
| Level JSON | ◐ | `mechanic-deposit-slot` `StageDialect`, project loaders | level-data | LoopHole schema |
| Circle solver / broadphase | ◐ | `core-collision-2d` `CollisionWorld2D.forEachOverlappingPair` | ball-sim | Gauss-Seidel position solver, centre pull, region constraints, 15k scale: custom |
| Capture-by-colour | ◐ | `mechanic-swallow-grow` `Swallower`, AllInHole `HoleCollectionModule` | ball-sim | colour-to-slot rule, gravity well, capture choreography |

**Stays custom (○):** the sleep/wake solver and its tuning for 15,000 balls, the capture choreography and colour-match-to-slot rule, toon `.effect`s and look-dev, the LoopHole level schema, the 15k perf/size work.

**Closest prior art:** AllInHolePLA (same genre), BounceDefendersPLA (pool/FX/store redirect), TimelineUpPLA (instancing, billboard, morph, entrance video), RushTankPLA (collision world, stage director), HexaBugFlowPLA (flood-fill board).

---

## 5. Module architecture

**Root:** `assets/scripts/[Name-Project]/`. `[Name-Project]` is a **placeholder: rename to the real project folder before building**. Game logic never imports `cc` outside `engine/cocos/` (`engine-decouple-cocos.md`).

| Module | Files (relative to Root) | Answers | Upstream | Reuse | Status |
|---|---|---|---|---|---|
| `level-data` | `level-data/LevelDataProvider.ts`, `level-data/LoopHoleLevelSchema.ts` | What does this level contain? | `@playablelabs/mechanic-deposit-slot` (pattern) | ◐ | TBD |
| `board` | `board/BoardModel.ts`, `board/BoardOpenSolver.ts`, `board/WallBake.ts` | Which cells are holes/walls, which holes can leave? | `@playablelabs/core-grid-flood` | ◐ | TBD |
| `hole` | `hole/HoleController.ts`, `hole/HoleFlight.ts`, `hole/HoleStateMachine.ts` | What state is each hole in and where is it flying? | `@playablelabs/core-combat-shapes` | ◐ | TBD |
| `disk` | `disk/DiskController.ts`, `disk/DiskSlot.ts` | Which slots hold which colour and how full? | `@playablelabs/cc-node-spin` | ◐ | TBD |
| `ball-sim` | `ball-sim/BallStore.ts`, `ball-sim/SpatialGrid.ts`, `ball-sim/BallSolver.ts`, `ball-sim/SleepWake.ts`, `ball-sim/CaptureSystem.ts`, `ball-sim/CaptureAnimator.ts` | Where is every ball and who captures it? | `@playablelabs/core-collision-2d` (broadphase seed) | ○ | TBD |
| `objective` | `objective/ObjectiveController.ts`, `objective/FailCountdown.ts` | Won, lost, or ticking? | `@playablelabs/feature-round-outcome` | ◐ | TBD |
| `bowl` | `bowl/BowlGlowController.ts` | Is the endgame glow/spin on? | | ◐ | TBD |
| `camera-rig` | `camera-rig/CameraRig.ts` | Where is the camera and does it shake? | `@playablelabs/cc-screen-fit` | ◐ | TBD |
| `tutorial` | `tutorial/TutorialHint.ts` | Which hole is the hand pointing at? | `@playablelabs/core-guide-sequence` | ● | TBD |
| `vfx` | `vfx/VFXController.ts` | How do events look? | `@playablelabs/cc-polish-kit` | ◐ | TBD |
| `audio` | `audio/AudioBindings.ts` | Which sound plays on which signal? | `@playablelabs/cc-lazy-load` | ● | TBD |
| `params` | `params/ParamBindings.ts` | Which client parameter drives what? | `@playablelabs/parameter-tool` | ◐ | TBD |
| `engine/cocos` (adapters, only place importing `cc`) | `engine/cocos/BallRenderer.ts`, `engine/cocos/HoleView.ts`, `engine/cocos/BoardView.ts`, `engine/cocos/DiskView.ts`, `engine/cocos/InputAdapter.ts` | How does sim state become pixels and taps? | `@playablelabs/cc-instanced-material` | ◐ | TBD |
| `GameplayController` | `GameplayController.ts` | Orchestrator | | ◐ | TBD |

Rules: modules talk only via `SignalBus` (named methods, unsubscribe in `onDestroy`); only `GameplayController` touches `GameView`/`CTAService`; `ball-sim` is engine-free and receives a read-only `DiskSnapshot` (slot colours and positions) injected by the controller, never an import of `disk`. Tests live in dot-prefixed `.tests/`. No code-built node trees (`prefab-only-construction-cocos.md`): holes via pooled `instantiate(prefab)`, walls baked offline, balls as instanced mesh.

---

## 6. Communication flow

```
-- STARTUP --------------------------------------------------------------
  GameplayController.startLevel()
    ├─> LevelDataProvider.load(level23)     # packed bin -> SoA arrays + board cells
    ├─> BoardModel.init(cells) + BoardOpenSolver.refresh()   # needs cells
    ├─> DiskController.init(5 slots)        # needs params
    ├─> BallStore.init(packed) + BallRenderer.bind()         # needs level data
    ├─> CameraRig.init(), ObjectiveController.init()
    └─> TutorialHint.start()

-- PLAYER INPUT ---------------------------------------------------------
  InputAdapter (raycast) -> HoleTapped{holeId}
    ├─> HoleController: closed? -> HoleRefused{closed}  -> VFX wobble+flash, Audio
    ├─> disk full?     -> HoleRefused{diskFull}         -> VFX crouch-shake, Audio
    └─> open & slot free -> HoleSendRequested -> DiskController reserves -> SlotReserved
          └─> HoleFlight (arc) -> HoleLanded{holeId,slot,colour}
                ├─> DiskController: seat, SlotAssigned{slot,colour}
                ├─> BoardOpenSolver.refresh() -> BoardSendableChanged
                └─> CaptureSystem arms slot after 0.25 s delay

-- GAME LOOP (per frame) ------------------------------------------------
  BallSolver.tick(dt)            # awake balls only: integrate, collide, constrain
    ├─ SleepWake.update()        # wake neighbours of vacated space, sleep settled
    ├─ CaptureSystem.tick()      # gravity well, band+angle test, adjacent pulls
    │     └─> BallCaptured{slot,n,limit} -> DiskController progress, VFX, Audio
    │           └─ n == limit -> SlotCompleted{slot} -> DiskController closes + frees
    └─ BallRenderer.sync()       # write instance buffer for awake + captured balls
  ObjectiveController.tick(dt)   # collectable check, countdown, deadlock

-- WIN ------------------------------------------------------------------
  LevelWon -> GameplayController.onWin() -> GameView.onWin() -> EndCard -> CTA
-- LOSE -----------------------------------------------------------------
  FailCountdownState{active} -> disk red pulse
  LevelLost{reason} -> GameplayController.onLose() -> shake -> EndCard / store-only route
```

---

## 7. Signal contracts

| Signal | Payload | Fired by | Consumed by | Trigger |
|---|---|---|---|---|
| `HoleTapped` | `holeId` | `InputAdapter` | `HoleController` | Player tap on a hole |
| `HoleRefused` | `holeId, reason(closed\|diskFull)` | `HoleController` | `VFXController`, `AudioBindings` | Tap on closed hole or full disk |
| `HoleSendRequested` | `holeId, colour` | `HoleController` | `DiskController` | Open hole tapped, slot free |
| `SlotReserved` | `holeId, slot, worldPos` | `DiskController` | `HoleController`, `VFXController` | Slot chosen (nearest free) |
| `HoleLanded` | `holeId, slot, colour` | `HoleController` | `DiskController`, `BoardOpenSolver` | Flight finished |
| `BoardSendableChanged` | `openIds[]` | `BoardOpenSolver` | `HoleController`, `TutorialHint`, `ObjectiveController` | Board occupancy changed |
| `SlotAssigned` | `slot, colour` | `DiskController` | `GameplayController` (-> `DiskSnapshot`), `ObjectiveController`, `VFXController` | Hole seated |
| `BallCaptured` | `slot, count, limit` | `CaptureSystem` | `DiskController`, `AudioBindings`, `VFXController` | Capture animation completes |
| `CollectableChanged` | `perColour[]` | `CaptureSystem` | `ObjectiveController` | Counts of capturable balls changed |
| `SlotCompleted` | `slot` | `DiskController` | `ObjectiveController`, `BoardOpenSolver`, `VFXController`, `AudioBindings` | Slot hits limit, hold 0.2 s |
| `FailCountdownState` | `active, secondsLeft` | `ObjectiveController` | `DiskController`, `VFXController`, `AudioBindings` | Countdown start/stop/tick |
| `EndgameEntered` | none | `ObjectiveController` | `BowlGlowController`, `DiskController`, `CaptureSystem` | 5 or fewer holes left |
| `LevelWon` | none | `ObjectiveController` | `GameplayController` | Win condition |
| `LevelLost` | `reason` | `ObjectiveController` | `GameplayController`, `CameraRig` | Countdown expiry |

No module imports another; verify by `t1k-cocos-base-system-design` `verify-manifest.cjs` once the doc is promoted.

---

## 8. Class summary, level data, class reference

### Class summary

| Class | Module | Key properties | Key methods |
|---|---|---|---|
| `LevelDataProvider` | level-data | packed buffers, `holes[]`, `wallCells[]`, `captureLimit` | `load()`, `getBalls()`, `getBoard()` |
| `BoardModel` | board | `occupancy`, `holes` | `isOccupied()`, `release()` |
| `BoardOpenSolver` | board | `goals(row 0)` | `refresh()` (uses `exposed`) |
| `HoleController` | hole | `states[]` | `onTap()`, `send()`, `refuse()` |
| `HoleFlight` | hole | `jumpDuration, jumpPower` | `fly(from,to)` |
| `DiskController` | disk | `slots[5]`, `spinSpeed` | `reserveNearest()`, `seat()`, `close()`, `setWarning()` |
| `BallStore` | ball-sim | `x,z,vx,vz (Float32Array)`, `type,state,awake (Uint8Array)` | `wake(i)`, `retire(i)` |
| `SpatialGrid` | ball-sim | `cellStart, cellItems` (CSR) | `rebuild()`, `neighbours()` |
| `BallSolver` | ball-sim | tuning block (damping, maxSpeed, centrePull, substeps) | `tick(dt)` |
| `SleepWake` | ball-sim | `restFrames[], wakeRadius` | `update()`, `wakeAround(x,z)` |
| `CaptureSystem` | ball-sim | `slotSnapshot`, `inFlight[]` | `tick()`, `tryBegin()`, `onSlotAssigned()` |
| `CaptureAnimator` | ball-sim | phase params (approach, dive, plunge) | `evaluate(t)` |
| `BallRenderer` | engine/cocos | instanced mesh, colour buffer | `sync()` |
| `ObjectiveController` | objective | `failCountdown`, `endgameThreshold` | `tick()`, `evaluate()` |
| `CameraRig` | camera-rig | pos, rot, ortho/persp, zoom | `apply()`, `shake()` |
| `VFXController` | vfx | particle refs | `on[Signal]()` |
| `GameplayController` | root | all module refs | `startLevel()`, `onWin()`, `onLose()` |

### Level-data schema (decoded from the three real samples)

Source JSON (Unity `JsonUtility`): `levelId, coloredHoleCaptureLimit, overrideColoredHoleCaptureLimit, referenceHoleCaptureRadialHalfThickness, arenaRadius, ballRadius, ballEntries[{ballType, position{x,y,z}}], pipes[], board{rows, columns, scale, cells[{cellType, unitType}]}, paletteOverrides[{ballType, ball, inside, border}]`.

| Field | Level_23 value | Note |
|---|---|---|
| `ballEntries` | 15,000 (Green 7,434 / DarkTeal 7,566) | 99.5% of minified bytes; `y` is constant 0.1109 |
| `ballRadius` / `arenaRadius` | 0.0909 / 13 | |
| `board` | 9 rows x 8 cols, 24 holes (12/12), 48 walls | `cellType` 1 = hole, 2 = wall; `unitType` = colour id |
| `coloredHoleCaptureLimit` | 200 | dashboard-config param |
| `paletteOverrides` | 2 (types 1, 2); only type 2 affects L23 | |
| Dead fields | `isHardLevel, ballCount, holeCount, sourceImageId, freezeCounter, freezeOrder, columnSlideColumns` | never read |

**Packing (billed in row 2):** offline script converts JSON to a binary asset: `x,z` as Float16 pair + `type` uint8 per ball (~75 KB vs 1.4 MB minified JSON). `y` is recomputed from the bowl height profile. Exact layout is stored, not regenerated: the data has no seed (`level-json-analysis.md` §5c). **Uncertain:** whether the client's final P1 JSON equals `Level_23.json`; schema is identical across the three samples, so the loader is unaffected.

### Class reference (bespoke modules)

- **`BallStore`**: SoA arrays sized `N=15,000`; `state` ∈ {Rest, Active, Captured}; `retire(i)` swap-removes. Fires nothing (pure data).
- **`BallSolver.tick(dt)`**: clamp `dt` to 0.05, substeps = clamp(ceil(dt/(1/60)),1,4); per substep: integrate (centre pull, gravity well, damping 0.92, max speed 8) -> `SolverIterations=5` Gauss-Seidel passes of ball-ball + region constraint (annulus: disk radius + r .. arena radius - r) -> velocity from displacement. **Awake balls only.** Reads `DiskSnapshot` for wells.
- **`SleepWake`**: a ball sleeps after `restFrames` below speed epsilon with all neighbours asleep; wakes on a neighbour moving, on a captured ball vacating, or inside a slot's well radius. Per-frame awake cap triggers staged waking (tuning, §11).
- **`CaptureSystem`**: per assigned slot after 0.25 s: candidates inside the capture band and angular window (`HoleAngularCaptureSize`), reserve `limit - captured - inFlight`, adjacent same-colour pulls (neighbour radius 2.2 ball radii), endgame radius boost +0.2; completed capture fires `BallCaptured`, `CollectableChanged`.
- **`ObjectiveController`**: win/lose per §2; countdown 5.5 s, final-hole delay 0.5 s; restarts countdown when a new colour lands; deadlock = no open hole can reach a free slot.
- **`HoleController`**: FSM Closed -> Open -> Flying -> Seated -> Completed (`StateManager`); lid open drives morph weight along an authored curve (7 keys in runner).
- **`DiskController`**: spin 65 deg/s, endgame 200 deg/s with 200 deg/s^2 acceleration; per slot: colour, count, % label, vacuum VFX attach/emission, close hop + sink.
- Framework-inherited classes (`SignalBus`, `ObjectPoolManager`, `InputService`, `AudioService`, `CTAService`, `GameView`) are one-liners from the template.

---

## 9. Estimate: line items and totals

> `Reuse`: ●=inherited (near-zero) · ◐=framework-assisted · ○=custom. **Base = O = L; P = worst-case.** `Type`: code = module (§5) · prod = cross-cutting package. Rates per `estimation-rules.md` §1b: ~200 lines/h authoring; allowance +1x debug-bound, +2x tuned-by-eye.

| # | Module / Package (classes) | Type | Reuse | Scope billed (new logic; cost driver) | O | **L** | P |
|---|---|---|---|---|---:|---:|---:|
| | **Gameplay code modules** | | | | | | |
| 1 | bootstrap + `GameplayController` | code | ◐ | Template supplies flow/CTA/bootstrap; billed: LoopHole states, init order, snapshot injection. ~150 lines, typing-bound | 1.5 | **1.5** | 2.0 |
| 2 | `level-data` (`LevelDataProvider`, packer script) | code | ◐ | Decode 15k balls into SoA, board cells, palette; offline JSON->binary packer. ~220 lines, part debug-bound (0.75 + 0.75 + 1.0 packer/verify) | 2.5 | **2.5** | 3.0 |
| 3 | `board` (`BoardModel`, `BoardOpenSolver`, `BoardView`) | code | ◐ | Grid build, centring, checker bg, occupancy; BFS via `core-grid-flood` (package covers flood, delta = wiring). ~260 lines, part debug-bound | 2.5 | **2.5** | 3.0 |
| 4 | Walls (`WallBake` offline mesh bake from `cellType 2`) | code | ◐ | Outline loops + bevel profile on `buildStripMesh`, baked to a mesh asset (no runtime node build). ~300 lines, debug-bound | 3.0 | **3.0** | 4.0 |
| 5 | `hole` states, tap, lid morph, wobble/flash/refuse (`HoleController`, `HoleStateMachine`) | code | ○ | FSM, raycast onto hole meshes, morph-weight driver on FBX blend shape (+1.5 h spike risk), 4 feedback animations. ~350 lines: 1.75 + 1.75 allowance + 1.5 morph | 5.0 | **5.0** | 7.0 |
| 6 | `hole` send flight (`HoleFlight`) | code | ◐ | Arc via `ballisticLaunchAngles`; delta: slot pick, easing, land squash, trail anchor. ~200 lines, tuned-by-eye (1.0 + 2.0) | 3.0 | **3.0** | 4.0 |
| 7 | `disk` (`DiskController`, `DiskSlot`, slot labels) | code | ◐ | Spin by `NodeSpinner` (covered); billed: 5 slots, seat/close/hop/sink, 5 world-space `%` labels (no package), endgame spin ramp. ~380 lines, part debug-bound | 4.0 | **4.0** | 5.0 |
| 8 | `ball-sim` core (`BallStore`, `SpatialGrid`, `BallSolver`) | code | ○ | Port of runner subsystem 19 minus pipes: CSR grid, integrate, Gauss-Seidel, annulus constraint, support gate. ~450 lines, **tuned-by-eye + perf-bound** (2.25 + 4.5 + 0.25) | 7.0 | **7.0** | 10.5 |
| 9 | `ball-sim` sleep/wake (`SleepWake`) | code | ○ | New (not in runner): rest detection, wake propagation, awake budget. ~200 lines, tuned-by-eye (1.0 + 2.0 + 0.5) | 3.5 | **3.5** | 5.5 |
| 10 | `ball-sim` capture (`CaptureSystem`, `CaptureAnimator`) | code | ○ | Band/angle test, well+swirl, adjacent pulls, reservation, 5-phase animation (simplified dive). ~420 lines, tuned-by-eye (2.1 + 4.2 + 0.2) | 6.5 | **6.5** | 10.0 |
| 11 | `engine/cocos` `BallRenderer` | code | ◐ | `InstancedMaterial` covers batching; delta: per-instance colour/shade (3 variants), partial buffer updates for awake balls, captured balls as pooled nodes. ~250 lines, debug-bound | 3.0 | **3.0** | 4.0 |
| 12 | `objective` (`ObjectiveController`, `FailCountdown`) | code | ◐ | `RoundOutcome` + `CountdownTimer` cover resolution/timer; billed: collectable scan, deadlock, restart-on-new-colour, endgame. ~220 lines, part debug-bound | 2.5 | **2.5** | 3.0 |
| 13 | `bowl` (`BowlGlowController`) | code | ◐ | Endgame glow/wind ramp + disk spin trigger. ~120 lines | 1.5 | **1.5** | 2.0 |
| 14 | `camera-rig` (`CameraRig`) | code | ◐ | XYZ pos/rot, persp/ortho, zoom params; framing to video (tuned-by-eye); fail shake via `CameraShakeState` (covered). ~100 lines | 1.5 | **1.5** | 2.0 |
| 15 | `tutorial` (`TutorialHint`) | code | ● | `GuideSequencer`/`FingerTutorial` wiring ≈ 1.0 (anchor) + 0.5 for persistent re-targeting to next open hole | 1.5 | **1.5** | 1.5 |
| 16 | end-cards + CTA / store-redirect | code | ◐ | Template end card + CTA ≈ 0.5; store redirect hook ≈ 1.0 (anchor) incl. store-only route | 1.5 | **1.5** | 2.0 |
| 17 | `audio` (`AudioBindings`) | code | ● | ~8 sound hooks x 5 min + BGM param ≈ 1.0 (above 0.5 h: itemised) | 1.0 | **1.0** | 1.0 |
| 18 | `vfx` (`VFXController`) | code | ◐ | Signal-driven play of send trail, vacuum per slot, flash, confetti; zero game logic. ~150 lines | 1.0 | **1.0** | 1.5 |
| 19 | `params` client-list wiring (`ParamBindings`) | code | ◐ | Implementation wiring only (template covers the features): background image/solid/gradient, grayscale toggle, entrance video/GIF resize+position, text/font/banner. Sized by estimator at 2 h | 2.0 | **2.0** | 2.5 |
| | **Subtotal: code** | | | | 54.0 | **54.0** | 73.5 |
| | **Production packages** | | | | | | |
| 20 | Art production (import, prefabs, scene) | prod | ○ | 8 meshes (ball, hole, closed hole + proxy, grid cell, border, bowl, disk/platform) x ~2 min import, prefab + scene assembly via MCP 3.5, default lighting 1.0 | 5.0 | **5.0** | 6.0 |
| 21 | Toon `.effect` authoring | prod | ○ | 4 effects x ~1.25: instanced ball toon, hole/lid toon (wobble, shape), bowl/disk (warning, wind alpha), gradient bg + grayscale pass. Sized by estimator at 5 h | 5.0 | **5.0** | 7.5 |
| 22 | Env/shading polish | prod | ○ | 5 distinct models + dynamic-colour surcharge (palette overrides), ~1 h each. Sized by estimator at 6 h, below the §5 rate of ~2 h per model | 6.0 | **6.0** | 9.0 |
| 23 | VFX assets | prod | ◐ | 5 effects x 1 h: BowlGlow glow, wind ribbon, VacuumVFX (gradient recolour), HoleSendTrail, slot-close burst | 5.0 | **5.0** | 10.0 |
| 24 | FPS optimization (runtime) | prod | ○ | 15,000-ball CPU sim + instanced render, low-end device profiling, awake-budget tuning, draw-call check. Anchor 6/9 is for ~150 skinned crowd; this is 100x entities | 14.0 | **14.0** | 24.0 |
| 25 | Build size optimization | prod | ◐ | Binary level pack, texture/mesh compression, engine module pruning (no physics), size gate on all networks. Anchor 4/6, +heavier assets | 6.0 | **6.0** | 9.0 |
| 26 | Feedback juice (WP8.1) | prod | ◐ | Feedback-matrix cells (input-ack, invalid, progress, milestone, terminal) incl. haptic + loudness-banded cues, timing pass; tuned-by-eye | 5.0 | **5.0** | 7.5 |
| 27 | QA | prod | ◐ | Win, lose-by-full, deadlock lose, endgame, params x devices | 6.0 | **6.0** | 7.5 |
| 28 | Iteration buffer (0.1 x implementation) | prod | ○ | `estimation-rules.md` §1a: 0.1 x 106 = 10.6, rounded | 10.5 | **10.5** | 15.5 |
| | **Subtotal: production (incl. buffer)** | | | | 62.5 | **62.5** | 96.0 |
| | **TOTAL: hours** | | | | 116.5 | **116.5** | 169.5 |
| | **TOTAL: dev-days** (÷8) | | | | 14.5 | **14.5** | 21.25 |
| | **TOTAL: weeks** (÷40) | | | | 3.0 | **3.0** | 4.25 |

**Excluded defaults ($0 line items):** parameter dashboard and basic value/colour/text/timing params · all-network SDK/CTA adapters + build/export pipeline · template clone, build config, folder layout. FPS and size optimization are **not** in this list; they are rows 24-25.

**VFX not double-counted:** `VFXController` (row 18, wiring) vs juice authoring (row 26, feel/timing) vs VFX asset content (row 23).

**Reconcile:** code 54.0 + production 52.0 + buffer 10.5 = 116.5 = WP sizing total (WP0 1.5 / WP4 level+board+core+objective / WP1 art / WP2 asset-side folded into row 20 and 25 / WP8 / WP9). No slice missing or double-counted. Rows 21 and 22 are deliberately separate: 21 builds the shader techniques (4 effects), 22 is look-dev per distinct model (5 models); if the studio treats them as one package, the overlap risk is small after the estimator's sizing (5 h + 6 h).

**Reuse-tag reconcile:**
- `●` rows above 0.5 h: row 15 (anchor 1.0 + 0.5 deviation named), row 17 (8 hooks + BGM itemised). OK.
- `◐` rows name their delta in `Scope billed`. Rows 2, 3, 4, 11 name package coverage vs delta.
- Per-object packages state counts: row 20 (8 meshes), row 21 (4 effects), row 22 (5 models + surcharge), row 23 (5 effects).
- Anchored components inside blended rows: tutorial 1.0, store redirect 1.0, audio 5 min/hook shown as sub-values in rows 15-17.
- Sizing-basis: every code row states volume and driver; rows 5, 8, 9, 10 are high against their line counts because they are **tuned-by-eye** (+2x), which is the named risk.
- Sign-off status of the `●` ceiling / `◐` delta rule and the FPS anchor extrapolation (no second anchor exists at this entity scale): estimator sign-off required (`estimation-rules.md` §2a, §3b).

---

## 9b. Client summary: module overview (P1 only; combined P1+P2 table in Part B §B9b)

Testing and client-iteration effort is **included**, distributed into the modules that carry it rather than listed as its own line.

| Module | `LOHO091026_P1_Con1` |
|---|---:|
| Core framework, level data, flow, tutorial, end card, audio | 9.5 h |
| Board, holes, wall layout, tap and send flight | 13.5 h |
| Spinning disk, capture rules, win/lose | 6.5 h |
| Ball simulation and rendering (15,000 balls) | 26.0 h |
| Art integration, shaders and visual polish | 24.0 h |
| Effects, game feel and client parameters | 12.0 h |
| Performance and build size | 25.0 h |
| **Total** | **116.5 h** |

> **`LOHO091026_P1_Con1`: 116.5 h ≈ 14.5 dev-days ≈ 3.0 weeks** (single developer, ideal engineering time).
> Worst case if the §11 risks land badly: ~169.5 h ≈ 21.25 dev-days.
> Redistribution check: QA 6 + buffer 10.5 = 16.5 h moved into ball simulation (+6), art/shaders (+5.5), performance (+5); total identical to §9.

---

## 10. Delivery summary

### 10a. Timeline (single dev, serial)

- **Base:** ~14.5 dev-days ≈ ~3.0 weeks. **Worst:** ~21.25 dev-days ≈ ~4.25 weeks.
- Ideal engineering time; excludes client asset delivery, review cycles and holidays.
- **Against the 2026-10-16 due date:** 5 working days remain (Mon 12 to Fri 16 Oct). One dev cannot deliver full scope. Parallel lanes that do not share files: (A) `ball-sim` + `BallRenderer` + FPS (rows 8-11, 24, ~34 h), (B) shaders + art + VFX assets (rows 20-23, 21 h), (C) board/hole/disk/objective/params (rows 1-7, 12-19, ~34 h). Even with three developers, lane A alone is ~4.25 dev-days before integration and tuning, so Oct 16 holds only with scope cuts, not by staffing alone. This needs a client conversation now, as their brief asks.

### 10b. Included at no charge (studio defaults)

Parameter dashboard and basic params · all-network SDK/CTA adapters + build/export · template bootstrap. Not included at no charge: FPS and build-size optimization (billed rows 24-25).

### 10c. Headline

> **Base plan: ~116.5 h ≈ ~14.5 dev-days ≈ ~3.0 weeks.** Worst-case ~169.5 h ≈ ~4.25 weeks.
> Reuse of grid-flood, node-spin, ballistic arc, instancing, polish/audio/guide packages and AllInHole prior art keeps the gameplay shell small. The 15,000-ball sleep/wake solver, capture choreography, toon shaders and the performance work drive it up.

---

## 11. Cost concentration & risk hotspots

1. **Ball simulation + FPS (rows 8, 9, 10, 24: 31 h base, up to 50 h).** Awake-ball count right after a tap is the unknown: a capture wave may wake thousands of balls on a low-end phone. The runner itself rebuilds its grid up to 20x per frame; nothing was profiled in Unity either. First task: a throwaway typed-array spike with 15,000 balls and a wake wave on a real low-end device. If it fails, fall back to a smaller awake budget (visible "stalling" of far balls) or sprite balls, both of which change the look the client approved.
2. **Shaders and look (rows 21, 22: 11 h, up to 16.5 h).** ShaderGraph sources do not port; effects are rebuilt from property lists. Per-instance colour on `cc-instanced-material` is unverified. Grayscale must be added to every custom effect.
3. **Hole lid morph target (row 5).** Cocos morph-target support in a playable build is unverified; the +1.5 h is a spike. Fallback (scale/rotate lid) was offered and declined, so a failed spike is a scope change to raise.
4. **Packages are corpus-grounded only.** Fit of `core-collision-2d`, `cc-instanced-material`, `core-grid-flood` and the entrance-video package name must be confirmed on disk before the module rows are final.
5. **Due date** (see 10a).
6. **P2 is priced from the video only** (Part B). Its JSON matches none of the three levels; if it carries pipes or blockers, Part B's conditions break and that part must be re-estimated.

Two-sided effects: none (no skeletal characters). Dynamic-colour surcharge applied in row 22.

---

## 12. Gut-check

Base was derived bottom-up from stated line volumes and the §1b rates, then reconciled to the WP total (re-slicing did not move the number). I do **not** have a studio estimator's number for this exact game, so this is my sanity check, not theirs: a template-based hole-collect playable with simple physics and a few hundred bodies would be a much smaller job (not measured here). The extra comes from three things the template does not cover: the 15,000-body solver with sleep/wake, a faithful toon look, and the performance work. Base is lean on purpose; the risk lives in the worst-case (row 8-10, 21, 24). **Estimator sign-off needed** on rows 21/22 overlap, FPS row 24 (extrapolated from a 150-entity anchor), and the `◐` delta rule.

## 13. Next step

1. Resolve the due-date conversation with the client (§10a, §11 item 5).
2. Spike first (half a day): 15,000-ball sleep/wake solver on a low-end device, instanced per-instance colour, morph target on the closed-hole FBX. Go/no-go for rows 8-11, 21 and 5.
3. Then `/t1k:cook` straight off §§5-8. Order from the §6 STARTUP trace: level-data -> board -> disk -> hole -> ball-sim -> objective -> camera/bowl -> vfx/audio -> params -> optimisation.
4. Promote §§5-8 to `docs/system-design.md` (`state: to-be`) when the repo is scaffolded; generate `design-manifest.json` and run `verify-manifest.cjs`. Rename `[Name-Project]`.

## Open caveats

- Ball counts and "~15 of 24 holes sent" in the video come from visual estimation, not data; the L23 colour-id mapping to Green/DarkTeal is unverified.
- P1 clip shows no win, lose or end card; those behaviours come from the runner and template defaults.
- Whether P1's final JSON equals `Level_23.json` is not confirmed (client has not shared final data).
- SFX character in the clip was not listened to; audio assets are assumed supplied.

---
---

# Part B: `LOHO091026_P2_Con1` Estimate

**Principle: P2 is billed only for what P1 does not already deliver.** Everything P1 builds (board, holes, disk, capture, ball sim, objective, params, template features, shaders already authored, VFX, audio hooks) is reused at 0 h. Verification of the second build is billed at confirmation level, not as a rebuild.

**Source:** the second creative clip in `LoopHole/` + `_scout/video-analysis.md` §2. **Level JSON not supplied.** Priced from the clip only, on the conditions below. Same units and rates as Part A.

## B1. Conditions this price depends on (break one, re-estimate that part)

| # | Condition | Source of decision | If it breaks |
|---|---|---|---|
| C1 | **Built as a separate playable from the P1 codebase**, after P1 is delivered. All P1 modules, tests and spikes are reused as-is | Gate answer 2026-10-12 | If P2 starts first or standalone, add most of Part A's rows (≥ 100 h) |
| C2 | Level schema identical to the three samples; **no pipes, no blockers**, no `board.mode != 0` | Gate answer ("assume from the clip") | New module(s) (pipes + entrance pour are ~1,500 runner lines); not priced here |
| C3 | Level = 3 colours (red, yellow, green), 15 holes on a 3 x 5 board, ~300-350 balls, large and glossy | P2 clip visual count (estimate, not data) | Ball count above ~2,000 re-opens the perf rows |
| C4 | P1 decisions carry over unchanged: runner authoritative for rules, clip for look; full toon shading; morph-target lid; portrait only; audio supplied by client; template features default; no level editor | Gate answer | See Part A §3 |
| C5 | The P2 bowl/disk/ring art exists in the Unity project (`PlatformBowl2`, socketed disk, thick-ring hole, glass materials) | Not verified | Art row grows by whatever must be modelled |
| C6 | P1 spikes pass (sim, per-instance colour, morph target) | Part A §13 | n/a |

## B2. What actually differs from P1 (the only things billed)

| Aspect | P1 | P2 | Billed as |
|---|---|---|---|
| Intro | none | **0-4.9 s close-up** tracking one yellow hole filling to 96%, then eased zoom-out at ~5 s | New intro director (B3), the one real new feature |
| Camera | high, near top-down | tilted ~45-50 deg, full bowl and rim visible | Pose + framing (B6) |
| Bowl / disk | flat disk, no sockets, no rim | socketed disk, translucent glass bowl with rim | Art swap (B10), one new glass effect (B11), look-dev (B12) |
| Level | L23, 15,000 tiny balls | 3 colours, 15 holes, ~350 large balls | Data run + retune (B2, B4) |
| Everything else (rules, capture, UI, params, template features, audio, VFX) | | identical | 0 h |

## B3. Estimate: delta line items

| # | Module / Package | Type | Reuse | Scope billed (new work only; driver) | O | **L** | P |
|---|---|---|---|---|---:|---:|---:|
| | **Gameplay code modules** | | | | | | |
| B1 | bootstrap / build config | code | ● | Second build config, Intro state before Play. ~30 lines, typing-bound | 0.5 | **0.5** | 1.0 |
| B2 | `level-data` | code | ● | Run P1 packer on the supplied JSON, validate board/colours. No loader change (C2) | 0.5 | **0.5** | 1.0 |
| B3 | `intro` (`IntroDirector`, new module) | code | ○ | Camera tracks a demo hole through send, seat, fill 0 to 96%, then eased pull-back at ~5 s; hand over balls at 0-1 s; skip on first touch. ~180 lines: 0.9 authoring + 1.8 allowance (tuned-by-eye) | 3.0 | **3.0** | 4.5 |
| B4 | `ball-sim` retune | code | ● | Constants only for ball radius ~0.29 and ~350 bodies; no new logic. tuned-by-eye | 1.0 | **1.0** | 1.5 |
| B5 | board/bowl layout values | code | ● | 3-column scale/centring, socket-slot anchors. Values only | 0.5 | **0.5** | 1.0 |
| B6 | `camera-rig` pose | code | ◐ | Tilted pose + intro end-state framing; tuned-by-eye | 1.0 | **1.0** | 1.5 |
| B7 | `params` palette | code | ● | 3-colour palette entries, intro on/off | 0.5 | **0.5** | 0.5 |
| B8 | `tutorial` | code | ● | Hand inside the intro close-up (0.5 deviation anchor, `estimation-rules.md` §2b) | 0.5 | **0.5** | 0.5 |
| B9 | `audio` | code | ● | New files + intro cue, 3 hooks x 5 min, rounded | 0.5 | **0.5** | 0.5 |
| | **Subtotal: code** | | | | 8.0 | **8.0** | 12.0 |
| | **Production packages** | | | | | | |
| B10 | Art production | prod | ○ | Import + prefab swap: glass bowl with rim, socketed disk, thick-ring hole, scene for the tilted view (C5) | 2.0 | **2.0** | 3.0 |
| B11 | Toon `.effect` authoring | prod | ○ | **1 new effect**: translucent glass bowl (sorting, rim). Glossy ball/ring = parameters of P1 effects, 0 h | 2.5 | **2.5** | 4.0 |
| B12 | Env/shading polish | prod | ○ | 1 changed lighting setup + glass bowl look-dev, 2 h (P1 materials reused for ball, hole, disk) | 2.0 | **2.0** | 3.0 |
| B13 | VFX assets | prod | ◐ | Retint/scale existing suction sparks for the close-up, 1 effect x 0.5 h | 0.5 | **0.5** | 1.0 |
| B14 | FPS check | prod | ● | Profile the second build on a low-end device (glass transparency, intro camera); sim is lighter than P1 | 1.0 | **1.0** | 2.0 |
| B15 | Build size gate | prod | ● | Second build size check on all networks (pipeline default) | 1.0 | **1.0** | 1.5 |
| B16 | Feedback juice | prod | ● | Intro cue + new visuals only; P1 matrix reused | 0.5 | **0.5** | 1.0 |
| B17 | QA | prod | ◐ | Intro skip/replay, win, lose, params on the second build | 2.0 | **2.0** | 3.0 |
| B18 | Iteration buffer (0.1 x implementation) | prod | ○ | `estimation-rules.md` §1a: 0.1 x 19.5 = 1.95, rounded | 2.0 | **2.0** | 3.0 |
| | **Subtotal: production (incl. buffer)** | | | | 13.5 | **13.5** | 21.5 |
| | **TOTAL: hours** | | | | 21.5 | **21.5** | 33.5 |
| | **TOTAL: dev-days** (÷8) | | | | 2.75 | **2.75** | 4.25 |
| | **TOTAL: weeks** (÷40) | | | | 0.5 | **0.5** | 0.75 |

**Reconcile:** code 8.0 + production 11.5 + buffer 2.0 = 21.5. Worst buffer = 0.1 x 30.5 = 3.05, rounded to 3.0.

**Reuse-tag reconcile:** `●` rows are all at or below 1.0 h, and each states what is being confirmed or configured. `◐` rows B6, B13, B17 name the delta. Per-object packages state counts: B10 (variant asset set), B11 (1 effect), B12 (1 look-dev), B13 (1 effect). Sizing basis stated for every code row; B3, B4, B6 are tuned-by-eye with the allowance named.

**Excluded defaults ($0):** same as Part A §9.

## B4. Risk hotspots (P2)

1. **JSON unknown (C2).** Largest swing: pipes or blockers would add a module not priced here. Ask the client for the JSON early.
2. **Glass bowl (B11, B12).** Transparent sorting and depth against glossy balls are the usual failure points in a size-capped playable; fallback is an opaque tinted bowl (visible look change).
3. **Intro director (B3).** If the intro must also be recorded and replayed deterministically, add ~2 h.
4. **Sequencing (C1).** The 21.5 h figure is only true if P1's code, packages and spikes exist.

## B5. Gut-check

P2 is the intro plus a handful of visual and data swaps over a finished P1. The earlier draft of this part (41.5 h) re-billed polish, FPS, size, juice, QA and VFX as if rebuilt; this version bills only the delta. Not measured against a studio number; estimator sign-off needed on B11/B12 (glass effect) and the confirmation-level B14/B15. **Consistency note:** after the estimator trimmed P1 effects to ~1.25 h each and polish to ~1 h per model, B11 (2.5 h for one effect) and B12 (2 h) sit above P1's per-unit rate; they are left as drafted pending the estimator's call.

## B9b. Client summary: both playables

Testing and client-iteration effort is **included**, distributed into the modules that carry it rather than listed as its own line.

| Module | `LOHO091026_P1_Con1` | `LOHO091026_P2_Con1` | Both |
|---|---:|---:|---:|
| Core framework, level data, flow, camera, tutorial, end card, audio | 9.5 h | 3.0 h | 12.5 h |
| Board, holes, wall layout, tap and send flight | 13.5 h | 0.5 h | 14.0 h |
| Spinning disk, capture rules, win/lose | 6.5 h | 0.0 h | 6.5 h |
| Ball simulation and rendering | 26.0 h | 1.0 h | 27.0 h |
| Art integration, shaders and visual polish | 24.0 h | 8.5 h | 32.5 h |
| Intro, effects, game feel and client parameters | 12.0 h | 6.5 h | 18.5 h |
| Performance and build size | 25.0 h | 2.0 h | 27.0 h |
| **Total** | **116.5 h** | **21.5 h** | **138.0 h** |

> **Both: 138.0 h ≈ 17.25 dev-days ≈ 3.5 weeks** (single developer, ideal engineering time).
> Worst case: ~203 h ≈ 25.5 dev-days ≈ 5.0 weeks.
> P2 depends on P1 being built first and on the JSON having no pipes or blockers.
> Redistribution check: P2 QA 2 + buffer 2 = 4 h moved into art/shaders (+2) and intro/effects (+2); totals equal §B3.

## B13. Next step (P2)

1. Get the P2 JSON; re-check C2 and C3 against it before starting.
2. Start after P1 ships; do B3 (intro director) and B11 (glass effect) first, since they carry the visual and technical risk.
3. Run `/t1k:cook` off §§5-8 with the deltas in §B2.
