# LoopHole — Reuse Sweep (prior-art / corpus evidence)

**Target:** Cocos Creator 3.8.7, portrait **3D** playable ad ("LoopHole": a board of coloured holes that
BFS-open when a path to the top row exists, tapped to arc onto a rotating 5-slot disk that captures balls
of its own colour until 100%, plus ~15,000 tiny instanced balls in a bowl).
**Source brief:** `LoopHole/Client_Requirement.txt` (Video-1 = Level 23 priority, due 2026-10-16).
**Method:** `t1k-cocos-base-knowledge-sweep` contract — corpus search by capability keyword, then the
mandatory **library pass** (`assembly_prefix: "packages"` / `"catalog"`), then `resolve-stdlib-package.cjs`
on every `packages` hit so each REUSE names a real installable `@playablelabs/*` package.
**Date:** 2026-10-12.

## Scope swept (read this before any "not found")

| Fact | Value |
|---|---|
| Corpus reached | **yes** — `mcp__knowledge-retrieval__doc_search` answered every call (`discovery: mcp`) |
| Cocos repos indexed | **47** (of 108 indexed repos, `doc_list_indexed_repos engine:"cocos"`) |
| Cocos assemblies cited | `cocos-stdlib` → `packages` (installable) + `catalog` (mechanic cards); foundation `signalBus, objectPool, tutorials, juice, transitions, layout, ui, inputService, gameLifecycle, gameControl, score, progression, animations, StateMachine, utils, market, assetsManager, async`; project assemblies `CocosAllInHolePLA, CocosBounceDefendersPLA, CocosTimelineUpPLA, CocosRushTankPLA, CocosHexaBugFlowPLA, Cocos3DSpinReady, CocosSkullHeroPLA, CocosLordMobileFuntapPLA, CocosBeePartyPLA, CocosTripleMatchCityPLA, CocosArrow3DPLA, CocosPlayableAdsTemplate` … |
| **grep grounding** | **N/A — no local Cocos checkout.** `detect-cocos-project.cjs` on `C:\WORK\PROJECT\COCOS\Estimate` → `{"cocos": false, "playable": false, "roots": []}`; **no** `assets/packages/@playablelabs`, **no** `node_modules/@playablelabs`. Every hit below is **corpus-grounded, not disk-grounded**; every negative is scope-limited to *corpus-only* and does **not** assert absence on disk. |
| graph grounding | **unavailable** — `mcp__code-graph-cocos__*` absent this session (recorded, not skipped) |

### The two traps this sweep avoided
- **Library is absent from unscoped results.** The installable `@playablelabs/*` code lives only in the
  `packages` assembly, which does **not** surface in the broad call. Every ● below was confirmed by a
  dedicated `assembly_prefix: "packages"` call, then resolved to a real package name.
- **`project`-kind hits are precedent, not packages.** `CocosAllInHolePLA` / `CocosTimelineUpPLA` etc. are
  other playables' *source* — excellent design prior art, **zero of it installable**. Tagged ◐ with
  `CONTRIBUTE`, never ●.

## Tag legend
**●** inherited (a package/foundation member does the whole job) · **◐** framework-assisted (exists; the
delta is named) · **○** custom (nothing found — scope stated).

## Reuse table

| # | Capability | Tag | Package / symbol / path | What it covers | Uncovered delta |
|---|---|---|---|---|---|
| 1 | Game-flow FSM + win/lose + end-card + CTA + store-redirect | ◐ | `@playablelabs/feature-round-outcome` (`RoundOutcome`, `RoundOutcomeConfig.timeoutResult`), `@playablelabs/core-polish-profile` (`PolishDirector.showEndCard`/`win`, `EndCardProfile.ctaDelayMs`, `EndCardRefIds.cta`), `catalog#F:BounceDefendersStoreRedirectTriggerCard` | Win/lose/timed/manual round resolution; end-card entrance + CTA pulse; multi-rule store-redirect engine | **store-redirect WITHOUT the End Card stage** is a store-only branch (in the card pattern, not a shipped flag) — wire a store-only route; the LoopHole lid/board state machine is custom (see #22) |
| 2 | SignalBus / events | ● | foundation `signalBus` assembly (PLAGameFoundation / H5GameFoundation) | subscribe/emit pub-sub for inter-component events | use **named methods** (project rule — `unsubscribe()` matches by identity); the stdlib deliberately uses callback ports, not the bus |
| 3 | Parameter dashboard / config (colors, text, font, camera, grayscale, bg) | ● | `@playablelabs/parameter-tool` (PlayableParamterTool: `LabelConfig.fontSize`, `LabelComponentParameter`, `ALBEDO_DEFINE_EXACT`, `runtime-editor` TransformGizmo); `@playablelabs/mechanic-bounce-projectile` (`ConfigInput`, `ConfigSupplier` — live config replaced mid-session) | The dashboard + per-value live-replace plumbing the brief's 12 parameter requirements hang off | each concrete binding (which colour/font/camera field) is authored project-side |
| 4 | Audio manager (BGM / SFX + first-touch unlock) | ● | `@playablelabs/cc-lazy-load` (`LazyAudio`, `LazyAudioOptions.musicName`, `subscribeFirstInteraction`), `@playablelabs/core-polish-profile` (`SfxCue`, `validateSfxCue`, `PolishAudioPort.play`) | Deferred clip load after first frame, music name, first-input unlock; cue loudness **band validation** | `lufs` is a target, not a measurement — author each cue into its band (`game-feedback-mandate`); "change BGM / replace SFX" = authoring |
| 5 | Tutorial hand / pointer (optional, toggleable) | ● | `@playablelabs/core-guide-sequence` (`GuideSequencer`, `GuideStep.begin`, `createGuideStep`, `GuideStepSpec`, `GuidePathPort`), foundation `tutorials`/`juice` (`FingerTutorial`, `DragTrailGlow`, `TrailKit` MotionStreak) | Ordered guide steps driven by a caller-owned clock; pointer trail + hint; enable/disable | none material — the sequence is a pure state machine the host clocks |
| 6 | Object pool | ● | `@playablelabs/core-tick-schedule` (`FreeList.take/give/reset/clear`, `FreeListOptions.accepts`), `@playablelabs/cc-polish-kit` (`PolishNodePool`), foundation `objectPool` | LIFO retired-instance pool with duplicate/accepts guards; node pool for confetti/FX | none — pair `FreeList` (sim entities) with `PolishNodePool` (nodes) |
| 7 | Timers / countdown | ● | `@playablelabs/core-effect-motion` (`CountdownTimer`, `Countdown`), `@playablelabs/core-tick-schedule` (`DelayedActions`), `PolishPorts.scheduler` | One-shot countdowns, latched expiry tick, coroutine-style delays | none |
| 8 | Screen-adapt / portrait layout / safe-area | ● | `@playablelabs/cc-screen-fit` (`UIScreenResolution`, `AdaptiveDesignResolution`, `FitPolicy`, `coverSize`, `pickFitPolicy`, `assignWidget`, `refreshWidgetsAndLayouts`) | Full-window cover keeping design aspect; resize policy re-picked per aspect; widget/layout refresh | none |
| 9 | Tween / motion / juice / camera shake | ● | `@playablelabs/core-effect-motion` (`MotionEffects.popIn`/`restBob`, `TweenReactions.playCelebrate`/`playHit`, `CameraShakeState` + `addShake`), `@playablelabs/cc-polish-kit` (`PolishDirector`, `CcPortsEnv.paint`, confetti) | Pop-in, idle bob, celebrate/hit squash, additive stackable screen shake, confetti | none material |
| 10 | Input service (tap / raycast on 3D) | ◐ | foundation `inputService`; `@playablelabs/cc-tap-badge` (`TapBadge` — tappable world node, punch + report), `TapFeedbackPort`, `Packages TapShooter` (tap-to-deploy rules), `catalog#F:RushTankNearestEnemyTargetingCard` | Tap on a world node with feedback + a tap-to-select board pattern | **3D raycast onto arbitrary Mesh / 15k-ball colliders** — TapBadge is UI/world-node tap; wiring a physics/collider raycast (or a distance-to-hole test) is the delta |
| 11 | Camera controller (ortho/persp, zoom, pose params) | ◐ | `@playablelabs/cc-screen-fit` (`CameraFitWidth`, `orthoHeightToKeepWidth`), `@playablelabs/core-camera-follow` (`CameraFollowModel`, `CameraFollowConfig`, `clampCamera`) | Ortho visible-width fit; eased follow; camera clamping | explicit **XYZ pos/rot + persp↔ortho switch + zoom as dashboard params** is wiring (persp/ortho + zoom is a small delta on top of the ortho helpers) |
| 12 | Background (image / solid / gradient) | ◐ | `@playablelabs/cc-polish-kit` (`TransitionCoverCard.paintFallbackFill` — how to draw a solid that actually renders), `cc-screen-fit` `coverSize`, `catalog#F:BounceDefendersScreenShakeOnSignalCard` (backdrop-root pattern) | Solid colour (drawn, not a frame-less sprite); full-screen image via `coverSize` | **gradient** = author a gradient SpriteFrame/texture or a small gradient `.effect` — no shipped gradient generator. Solid ●, image ●, gradient ◐ |
| 13 | Grayscale / post-process | ◐ | **no shipped package.** Prior art: `CocosLordMobileFuntapPLA` `SoldierBase.grayEffect` / `_applyGrayscale` (desaturate shader swapped per-material), `CocosAllInHolePLA` `ref-post-grade.effect`, `catalog#F:BounceDefendersScreenShakeOnSignalCard` | A proven desaturate material + screen-post pattern in the fleet | author the desaturate `.effect` + a full-screen (or per-camera) pass bound to the parameter toggle — pattern only, CONTRIBUTE upstream |
| 14 | Entrance video / GIF player | ◐ | `CocosBlastPartyPLA#T:EntranceVideoPlayerView` (`play`, `resizeToCanvas`, `onPlaybackError` — `cc.VideoPlayer`), `CocosTimelineUpPLA#T:EntranceVideoView` (`teardown`; DOM-composited, not `cc.VideoPlayer`); **a package `EntranceVideoView` is referenced by both** (resolve install name via CPM). GIF via `@playablelabs/core-effect-motion` `FrameCycleState` (once/loop atlas cycle) | Video play-to-end / skippable / max-duration, canvas resize, teardown; atlas frame-cycling for GIF | **confirm the package's install name** (referenced, not resolved here); **GIF** needs frames decoded to an atlas then driven by `FrameCycleState` — resizable/repositionable/different-dimension GIF is the delta |
| 15 | Custom font loading / label text params | ● | foundation `ui` (`FontService._customFontAsset`, `CUSTOM_FONTS`), `animations` (`TextEffectManager.defaultFontSize`), `CocosH5CocosTemplate` `LOCALE_FONTS`; `@playablelabs/parameter-tool` `LabelConfig.fontSize` | Load a custom TTF/bitmap font and apply to labels; content/size/position via params | per-label binding authored project-side (cap #3) |
| 16 | Text / percentage label in 3D (world-space, above objects) | ◐ | `CocosSkullHeroPLA#T:WorldHealthBar` / `EntityHealthBar`, `CocosTimelineUpPLA#T:Billboard`, `CocosLordMobileFuntapPLA#T:Billboard` (+ `lateUpdate` no-alloc billboard), `CocosAllInHolePLA` `HudFrame.billboard` ("Size N" world label) | Billboard-and-follow a label above a 3D object; world-space number label (exactly the AllInHole "Size N") | **no library package** — ≥3 projects implement it; lift one (CONTRIBUTE). The captured-percentage label is the same primitive |
| 17 | Particle / trail VFX (trail, suction/vortex, glow) | ◐ | `@playablelabs/cc-polish-kit` (confetti, `CelebrationPresets`), foundation `juice` (`TrailKit` MotionStreak, `DragTrailGlow.spawn`), `catalog#F:BounceDefendersMeteorSkillSalvoCard`; **suction**: `CocosAllInHolePLA` `HoleCollectionModule.suction` + `ref-post-grade.effect` (`FX_PARAMS: suction, impact`) | Pooled particle effect player; UI trail / glow beam; suction visual pattern | **vortex/trail-renderer in 3D** and the exact suction shimmer are project-side / author-authored effects — pattern + pooled-player, then author the effect |
| 18 | ECS / data-oriented / typed-array sim / spatial hash / circle–circle solver | ◐ | `@playablelabs/core-collision-2d` (`CollisionWorld2D`, `forEachOverlappingPair` — unordered circle-pair broadphase), `@playablelabs/core-tick-schedule` (`FreeList` fixed-step bookkeeping), `catalog#F:RushTankCollisionWorldCard` (hand-rolled AABB/circle world) | Engine-independent 2D circle-pair collision world + pooled fixed-step bookkeeping | **3D broadphase / spatial hash for 15k balls** is not packaged — use Cocos builtin PhysX or a custom spatial hash; the 2D solver is the reuse seed. (Fleet has `Physics3DCocos` and AllInHole's JS physics as precedent) |
| 19 | GPU instancing + per-instance colour | ● | `@playablelabs/cc-instanced-material` (`InstancedMaterial`, `instancedOf`, `ColoredMeshArrays`), `@playablelabs/cc-mesh-fade` (`MeshFader` for pooled fade), `@playablelabs/core-mesh-builders` (`ColoredMeshArrays`, `buildQuadMesh`) | Batch many `MeshRenderer` copies into few draw calls; RGBA-interleaved per-vertex colour; pooled fade | **per-instance** colour (vs per-vertex) rides a per-instance uniform — confirm the shader exposes it; balls are one mesh so `ColoredMeshArrays` (vertex colour) is the direct fit |
| 20 | Toon shader — outline / gloss / rim / contact-shadow `.effect` | ◐ | **no shipped package.** `CocosTripleMatchCityPLA` `getSpineOutlineMaterial`/`getSpriteOutlineMaterial`, `CocosArrow3DPLA` `VoxelCubeView.outlineFillMaterial`/`outlineMaskMaterial`, `CocosAllInHolePLA` `hole-rim`/`hole-progress-outline`/`ref-outline` materials, `CocosBounceDefendersPLA` `CcMeteorShadowPlayer` (contact-shadow mark) | Stencil outlines, rim/emissive, planar contact shadow — proven effects in the fleet | author the toon `.effect` (outline+gloss+rim) + contact-shadow decal — pattern only, CONTRIBUTE upstream |
| 21 | Morph target / blend shape on imported FBX | ◐ | `@playablelabs/cc-skeletal-pose` (`SkeletalPoseDriver.playAttack`/`_play`, clip blend/dedupe), `CocosTimelineUpPLA#F:WoodMorphDriver.blendAnimTime` (blendshape driver) | Skeletal clip playback with one-shot→pose fallback; a blendshape driver precedent | **driving `cc.MorphTarget`** weights from an imported FBX blend shape — `cc-skeletal-pose` covers skeletal clips; the morph-weight path is the delta |
| 22 | Grid-board BFS pathfinding / puzzle grid | ● | `@playablelabs/core-grid-flood` (`floodFrom`, `shortestPath`, **`exposed`** — targets touching the flood, `orthogonalNeighbors`, `GridCell`), `@playablelabs/core-footprint-grid` (`FootprintGrid`, `CellStateGrid`, `gridMetrics`, `cellCentre`, `ClearDetector`) | Flood fill from seeds; BFS fewest-steps path to nearest goal; **`exposed()` = reached cells / targets adjacent to the flood = exactly the "path to top row → open lid" rule**; cell occupancy + pixel geometry | wiring only — feed the board occupancy as `passable`, top row as `goals`, and read `exposed` for openable lids |
| 23 | Procedural mesh generation (extruded outline walls, bevel) | ◐ | `@playablelabs/core-mesh-builders` (`buildQuadMesh`), `@playablelabs/core-height-profile` (`buildStripMesh` — world-space **strip/extruded outline wall**), `CocosArrow3DPLA` `LevelDesign#M:generateArcCurve`/`shapeMaze` | Quad + strip mesh builders; arc/curve generation | **bevelled extruded walls** with a computed profile — `buildStripMesh` gives the extrusion, the bevel profile is the delta |
| 24 | Level JSON loader / data ingestion | ◐ | `@playablelabs/mechanic-deposit-slot` (`parseIdList`, `StageDialect.readTypeId`, `StageRaw.entity`, `StageEnemyNormalized`), `CocosArrow3DPLA` `LevelDataProvider.parseFromSources`, `CocosBeePartyPLA#T:LevelLoader`, `CocosHexaBugFlowPLA#T:LevelData` | Generic dialect-driven JSON entity ingestion + several project loaders | **LoopHole's level schema** (hole grid, colours, disk slots, ball counts) is client-authored JSON ("shared once available") — schema + validation is the delta |
| 25 | Curve / jump flight tween (parabolic arc, easing) | ● | `@playablelabs/core-combat-shapes` (`ballisticLaunchAngles` — solve launch for a gravity arc), `@playablelabs/mechanic-bounce-projectile` (`ProjectileFlight` — gravity/drag/range cap, `ConfigInput`), `catalog#F:BounceDefendersKillRewardCoinFlyCard` (parabolic coin hop), `CocosAllInHolePLA` `aimDotPoses` (26-dot ballistic arc + aim preview) | Ballistic arc solve, flight integration, arc preview dots, easing | none — this is the tapped-hole → disk arc; AllInHole's `aimDotPoses` is the exact preview primitive |
| 26 | Disk / rotor spin with slots | ● | `@playablelabs/cc-node-spin` (`NodeSpinner.tick`, `SpinConfiguration`, `SpinConfigInput.nodeCount`, `SpinAxis`), `@playablelabs/core-footprint-grid` (`FootprintGrid`/`CellStateGrid` for the 5-slot occupancy); `Cocos3DSpinReady` (`landingRotation`, `SPIN_EASE`, `SpinSectorConfig`, `SpinWheelBoard` — land a target index under the arrow after N turns) | Rotate a node group about a world axis; compute the landing rotation that puts a chosen slot under the pointer; slot occupancy | none — `cc-node-spin` + `landingRotation` (project precedent) cover the spin; FootprintGrid holds the 5 slots |
| 27 | Ball/marble sorting / hole.io capture-by-colour | ◐ | `@playablelabs/mechanic-swallow-grow` (`Swallower`, `GrowthLevel`, `buildCurveLevels` — swallow-and-grow = hole.io), `@playablelabs/core-collision-2d` (capture overlap test), `CocosAllInHolePLA` `HoleCollectionModule` (`suction`, `capture`, `judgeItem`, `consume`, `consumeSunk`) | Swallow/grow levels; capture overlap; the **exact** hole-collect mechanic (tilt/capture/consume) as project prior art | **colour-matching-to-slot rule** (capture only balls of the hole's colour, fill to 100%) is the bespoke rule — custom logic on top of the swallow/capture primitives |

## (a) Relevant `@playablelabs/*` packages found (installable)

Resolved via `resolve-stdlib-package.cjs` (exit 0 on each). All `cocos-stdlib` `packages` hits:

| Package | Version | Cited for | Key symbols |
|---|---|---|---|
| `@playablelabs/parameter-tool` | (PlayableParamterTool) | params (#3, #15) | `LabelConfig`, `LabelComponentParameter`, dashboard + runtime editor |
| `@playablelabs/core-tick-schedule` | 0.4.0 | pool / timers (#6, #7, #18) | `FreeList`, `DelayedActions` |
| `@playablelabs/core-effect-motion` | 0.11.0 | timers / shake / juice / GIF (#7, #9, #14) | `CountdownTimer`, `CameraShakeState`/`addShake`, `TweenReactions`, `FrameCycleState` |
| `@playablelabs/cc-screen-fit` | 0.3.0 | screen-adapt / camera (#8, #11, #12) | `UIScreenResolution`, `AdaptiveDesignResolution`, `CameraFitWidth`, `coverSize` |
| `@playablelabs/cc-instanced-material` | 0.1.0 | instancing (#19) | `InstancedMaterial`, `instancedOf` |
| `@playablelabs/cc-mesh-fade` | 0.1.0 | instancing / pooled fade (#19) | `MeshFader` |
| `@playablelabs/core-mesh-builders` | 0.3.0 | procedural mesh / instancing (#19, #23) | `buildQuadMesh`, `ColoredMeshArrays` |
| `@playablelabs/core-height-profile` | 0.3.0 | procedural mesh (#23) | `buildStripMesh` |
| `@playablelabs/core-grid-flood` | 0.2.1 | BFS / lid-open (#22) | `floodFrom`, `shortestPath`, `exposed` |
| `@playablelabs/core-footprint-grid` | 0.10.0 | grid / slots (##22, #26) | `FootprintGrid`, `CellStateGrid`, `gridMetrics`, `cellCentre`, `ClearDetector` |
| `@playablelabs/core-collision-2d` | 0.4.0 | collision solver / capture (#18, #27) | `CollisionWorld2D`, `forEachOverlappingPair` |
| `@playablelabs/core-combat-shapes` | 0.4.0 | ballistic arc (#25) | `ballisticLaunchAngles` |
| `@playablelabs/core-guide-sequence` | 0.2.0 | tutorial (#5) | `GuideSequencer`, `GuideStep`, `createGuideStep`, `GuideStepSpec` |
| `@playablelabs/core-polish-profile` | 0.1.0 | end-card / CTA / audio / win (#1, #4) | `PolishDirector`, `EndCardProfile`, `SfxCue`, `PolishAudioPort` |
| `@playablelabs/cc-polish-kit` | 0.1.0 | juice / pool / VFX (#6, #9, #12, #17) | `PolishNodePool`, `CcPortsEnv.paint`, confetti |
| `@playablelabs/core-depth-sort` | 0.4.0 | 3D label / draw order (#16) | `DepthSorter` |
| `@playablelabs/cc-node-spin` | 0.2.0 | disk spin (#26) | `NodeSpinner`, `SpinConfiguration` |
| `@playablelabs/cc-lazy-load` | 0.1.0 | audio (#4) | `LazyAudio` |
| `@playablelabs/cc-skeletal-pose` | 0.3.0 | skeletal / blendshape (#21) | `SkeletalPoseDriver` |
| `@playablelabs/cc-tap-badge` | 0.1.0 | tap input (#10) | `TapBadge` |
| `@playablelabs/mechanic-bounce-projectile` | 0.3.2 | arc / config (#25, #3) | `ProjectileFlight`, `ConfigInput`, `ConfigSupplier` |
| `@playablelabs/mechanic-wave-spawner` | 0.7.0 | data-driven spawn (#24) | `WaveRunner`, `Wave`, `WaveSchedule` |
| `@playablelabs/mechanic-swallow-grow` | 0.1.3 | hole.io capture (#27) | `Swallower`, `GrowthLevel` |
| `@playablelabs/mechanic-deposit-slot` | 0.1.0 | level JSON ingest (#24) | `parseIdList`, `StageDialect` |
| `@playablelabs/feature-round-outcome` | 0.3.0 | win/lose (#1) | `RoundOutcome` |

**Foundation assemblies (installed via the foundation package, kind `foundation`):** `signalBus, objectPool,
tutorials, juice, transitions, layout, ui, inputService, gameLifecycle, gameControl, score, progression,
animations, StateMachine, utils, market, assetsManager, async` (PLAGameFoundation / H5GameFoundation).
**Templates:** `CocosPlayableAdsTemplate`, `CocosH5CocosTemplate`, `CocosGameTemplate`.

## (b) Top 5 prior-art projects closest to "ball-collecting hole + 15k instanced balls"

| Rank | Repo (assembly) | Why it is close | Directly reusable evidence |
|---|---|---|---|
| 1 | **The1Studio/AllInHolePLA** (`CocosAllInHolePLA`) | **Same genre** — hole-collect game. The single closest corpus match by a wide margin | `HoleCollectionModule` (`suction`, `capture`, `judgeItem`, `consume`, `consumeSunk`, `beginCollection`), `TowerSimulation.setCollecting`, ballistic `aimDotPoses` (26-dot arc + aim preview), `HoleVisualComponent` (rim/outline renderers), `ref-post-grade.effect` (`FX_PARAMS: suction, impact, focusU/V`), `HudFrame.billboard` ("Size N" world label), `HoleCollectionHostPort` |
| 2 | **The1Studio/BounceDefendersPLA** (`CocosBounceDefendersPLA`) | Pooling + FX + store-redirect + arc-coin at production quality; the "polish" spine for a 3D playable | `PooledParticleEffectPlayer`, `DamageNumberPool`, `BoardHitTextPool`, `CcMeteorShadowPlayer` (planar contact shadow), `BounceDefendersStoreRedirectTriggerCard`, `BounceDefendersKillRewardCoinFlyCard` (parabolic arc), `BounceDefendersScreenShakeOnSignalCard` |
| 3 | **The1Studio/TimelineUpPLA** (`CocosTimelineUpPLA`) | **Instancing + billboard + blendshape + entrance video + time-scaled flight** — the exact 3D rendering techniques LoopHole needs | `GameBootstrap.setupRailings` (62 meshes folded by instancing), `MeshFader`/`ProjectileSpawner.collapseTracerMeshes`, `Billboard`, `WoodMorphDriver`/`WoodChain.blendAnimTime` (blendshape), `EntranceVideoView`, `ProjectileFlightModel` |
| 4 | **The1Studio/RushTankPLA** (`CocosRushTankPLA`) | **Custom ECS world + hand-rolled AABB/circle collision + level-loading stage director + ad flow / store redirect** | `RushTankCollisionWorldCard` (broadphase), `RushTankProjectileTypesCard` (sweep), `RushTankStageDirectorCard` (JSON → spawn), `RushTankAdFlowStateMachineCard` (loading/gameplay/win/lose legal-transition table), `ballisticLaunchAngles` |
| 5 | **The1Studio/HexaBugFlowPLA** (`CocosHexaBugFlowPLA`) | **Flood-fill BFS board whose "exposed" rule is LoopHole's lid-open rule verbatim**, plus level data + tinted VFX | `BoardModel.computeAttackable` (flood from rim + exposed-block detection), `LevelData`, `VFX_TINT_SCALE` (tinted particle), `RichTextConfig.entranceAnimation` |

**Close sixth:** `The1Studio/3DSpinReady` (`Cocos3DSpinReady`) — `landingRotation`/`SPIN_EASE`/`SpinWheelBoard`/`SpinSectorConfig`: a rotating wheel that lands a chosen slot under an arrow, i.e. the disk mechanic.
Also worth a look: `CocosBeePartyPLA` (voxel + level JSON), `CocosSkullHeroPLA` (world health bar / billboard / ballistic projectile flight).

## Annotation + envelope

```
discovery: mcp | scope: 47 Cocos assemblies (of 108 indexed repos) | swept: 2026-10-12
graph: unavailable — mcp__code-graph-cocos__* absent this session
grep: not-run — no local Cocos checkout (detect-cocos-project.cjs → cocos:false); corpus-only evidence
reuse-search: found InstancedMaterial @ @playablelabs/cc-instanced-material | found floodFrom @ @playablelabs/core-grid-flood | found FreeList @ @playablelabs/core-tick-schedule | found PolishDirector @ @playablelabs/core-polish-profile | found ballisticLaunchAngles @ @playablelabs/core-combat-shapes | found Swallower @ @playablelabs/mechanic-swallow-grow | found NodeSpinner @ @playablelabs/cc-node-spin | found CameraFitWidth @ @playablelabs/cc-screen-fit
reuse-search: not-found-after-corpus-and-grep — toon/grayscale .effect, background gradient, GIF decoder, billboard/world-label, vortex/trail-renderer (scope: 47 Cocos assemblies, project precedent only, no installable package)
```

```json
{
  "schema": "t1k-reuse-search/1",
  "verdict": "REUSE-with-deltas",
  "passes": {
    "broad_corpus": 84,
    "library_packages": 26,
    "catalog": 1,
    "resolve_stdlib_package": 20
  },
  "scope": "47 Cocos assemblies of 108 indexed repos; library=packages, catalog=mechanic cards; foundation allowlist 18 subsystems",
  "swept": "2026-10-12",
  "grep": "not-run (no local Cocos checkout; detect-cocos-project.cjs → cocos:false)",
  "graph": "unavailable",
  "evidence": [
    { "need": "instancing", "tag": "inherited", "pkg": "@playablelabs/cc-instanced-material", "version": "0.1.0", "symbol": "InstancedMaterial" },
    { "need": "grid-bfs-lid-open", "tag": "inherited", "pkg": "@playablelabs/core-grid-flood", "version": "0.2.1", "symbol": "floodFrom/shortestPath/exposed" },
    { "need": "object-pool", "tag": "inherited", "pkg": "@playablelabs/core-tick-schedule", "version": "0.4.0", "symbol": "FreeList" },
    { "need": "endcard-cta-store", "tag": "framework-assisted", "pkg": "@playablelabs/core-polish-profile", "version": "0.1.0", "symbol": "PolishDirector.showEndCard" },
    { "need": "ballistic-arc", "tag": "inherited", "pkg": "@playablelabs/core-combat-shapes", "version": "0.4.0", "symbol": "ballisticLaunchAngles" },
    { "need": "hole-capture-by-colour", "tag": "framework-assisted", "pkg": "@playablelabs/mechanic-swallow-grow", "version": "0.1.3", "symbol": "Swallower" },
    { "need": "disk-spin-slots", "tag": "inherited", "pkg": "@playablelabs/cc-node-spin", "version": "0.2.0", "symbol": "NodeSpinner + landingRotation" },
    { "need": "screen-adapt-camera", "tag": "inherited", "pkg": "@playablelabs/cc-screen-fit", "version": "0.3.0", "symbol": "UIScreenResolution/CameraFitWidth" },
    { "need": "circle-collision-solver", "tag": "inherited", "pkg": "@playablelabs/core-collision-2d", "version": "0.4.0", "symbol": "CollisionWorld2D" },
    { "need": "tutorial-pointer", "tag": "inherited", "pkg": "@playablelabs/core-guide-sequence", "version": "0.2.0", "symbol": "GuideSequencer" },
    { "need": "audio-bgm-sfx", "tag": "inherited", "pkg": "@playablelabs/cc-lazy-load", "version": "0.1.0", "symbol": "LazyAudio" },
    { "need": "curve-flight-tween", "tag": "inherited", "pkg": "@playablelabs/mechanic-bounce-projectile", "version": "0.3.2", "symbol": "ProjectileFlight" },
    { "need": "level-json-ingest", "tag": "framework-assisted", "pkg": "@playablelabs/mechanic-deposit-slot", "version": "0.1.0", "symbol": "StageDialect/parseIdList" },
    { "need": "procedural-mesh-walls", "tag": "framework-assisted", "pkg": "@playablelabs/core-height-profile", "version": "0.3.0", "symbol": "buildStripMesh" },
    { "need": "toon-grayscale-effects", "tag": "custom-pattern-only", "path": "CocosLordMobileFuntapPLA.grayEffect / CocosAllInHolePLA.ref-post-grade.effect", "note": "no installable package — CONTRIBUTE" },
    { "need": "background-gradient", "tag": "custom", "note": "solid+image covered (cc-polish-kit/cc-screen-fit); gradient effect is new" },
    { "need": "world-space-label-billboard", "tag": "custom-pattern-only", "path": "CocosSkullHeroPLA.WorldHealthBar / CocosTimelineUpPLA.Billboard", "note": "no installable package — CONTRIBUTE" }
  ],
  "closest_prior_art": [
    "The1Studio/AllInHolePLA",
    "The1Studio/BounceDefendersPLA",
    "The1Studio/TimelineUpPLA",
    "The1Studio/RushTankPLA",
    "The1Studio/HexaBugFlowPLA"
  ]
}
```
