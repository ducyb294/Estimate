# LoopHole LevelRunner — System Map (read-only investigation)

**Target:** `C:\WORK\VIDEO\LoopHole\Assets\LevelRunner\LoopHoleLevelRunner.cs` (7,783 lines, 307,321 bytes, one sealed `MonoBehaviour`) plus `README.md`, and the three shipped level JSONs under `Levels/`.
**Verb:** map + inventory + measure (constants, physics weight, rendering weight, JSON contract).
**Source hash/basis:** file read in full in this session (offsets 1→7783). All locators are `LoopHoleLevelRunner.cs:<line>` unless another file is named.
**Not in scope / not done:** ranking, recommendations, effort or hour estimates (none given). No source, config or data store was modified. No Unity project was opened or run — every claim is a static read of the C# and the JSON; runtime numbers are NOT measured (see "Could not measure").

---

## 1. Game loop in plain words

The player is looking at a **tilted 3D bowl** viewed from above/behind (camera pose `(0, 12.03, -14.96)`, Euler `(48,0,0)`, orthographic size 16 — `README.md:33`, applied at `1225-1235`). Inside the bowl lies a pile of **coloured balls**. In the middle of the bowl, at the bottom, sits a flat **disk** that spins slowly (`DiskRotationSpeed = 65`, `481`) and carries **5 "hole" sockets** (`HoleCount = 5`, `434`) on an orbit (`HoleOrbitRadiusConfig = 2.3`, `433`).

In front of the bowl (further from camera) floats a **board** — a grid of tiles (`BoardWorldPosition (0, 7.55, -24.65)`, `598`). Each **hole unit** on the board is a coloured "lid" (a closed lid sprite) sitting on a cell. The player's entire interaction is: **tap/click a hole on the board**.

- A hole is **"open"/sendable** when there exists a 4-way path of unoccupied tiles from its cell to **row 0** ("the exit row", the edge nearest the bowl) — computed by BFS `BuildPathToExit` (`2089-2146`), `IsExitCell` (`2164-2177`: `coordinates.y != 0` fails). Sendable state is refreshed by `RefreshAllSendableStates` / `RefreshUnitSendableState` (`2034-2062`).
- Tap an **open** hole → it **jumps/fly-arcs off the board onto a free disk slot** (`OnHoleTapped` `1774-1813`, `SendHoleToDisk` `1869-1890`, `FlyHoleToDisk` `1908-1969`), picking the slot nearest the near side of the disk (`FindNearestDiskSlot` `2939-2960`, reference `(0,-orbitRadius)` `1827`).
- Tap a **closed** hole → it wobbles and its grid cell flashes red (`PlayWobble` `2515-2532`, `FlashUnsendableGridCell` `1584-1610`, `UnsendableFlashColor` `620`). No penalty.
- Tap an open hole while **all 5 disk slots are taken** → the hole plays a "crouch-lurch-bounce-shake" refusal animation (`PlayDiskFullFeedback` `2535-2575`); the board is NOT failed immediately.
- A hole that has landed on a **disk slot** now **pulls in every ball of its own colour** from the bowl: a pre-capture "gravity well" (`ApplyHoleGravityWell` `6170-6219`: radial pull `HoleGravityWellStrength=9` + tangential swirl `HoleVortexSwirlStrength=7`, quadratic falloff over `HoleGravityWellRadiusBallRadii=9` ball-radii), then a real capture when the ball enters the radiating **capture band** around the disk edge (`IsBallInCaptureBand` `3648-3658`) **and** is inside the hole's **angular window** (`IsAngleInHole` `3660-3677`, half-width `GetHoleAngularCaptureSizeDeg` `3364-3370`). The captured ball plays a scripted **wind-in → arc → coil → dive → plunge → vanish** animation (the bulk of lines `3372-4092`) and is counted.
- A slot fills at `_coloredHoleCaptureLimit` balls (default 35, `531`; level JSON may override, e.g. 100 and 200 in the shipped levels). On reaching the limit the slot shows 100%, waits `HoleCloseHoldAtFullPercentSeconds=0.2`, then **closes and frees** (`CompleteHoleCapture` `3758-3777`, `ScheduleHoleCompletion` `3263-3274`, `ResetSlotToDefault` `3289-3313`), and the freed slot can accept a new board hole.

**Win** (`OnDiskHoleCompleted` `6964-6971`, `WinLevel` `1274-1286`): a disk hole completed **and** there is no hole left on the board **and** no disk slot still filled. Then `Finished?.Invoke(true)`.

**Lose** (`LevelStateTick` `6888-6926`, `RunFailCountdown` `7015-7054`, `LoseLevel` `1288-1301`): with **4 or 5 of 5** slots filled (`GetAssignedHoleCount() >= HoleCount-1`) and **no collectable ball** for any assigned colour (`HasCollectableBallForAssignedHoles` `3721-3756`), a `FailCountdownSeconds = 5.5` countdown ticks (disk flashes red, `SetFailCountdownWarning` `7063-7084`). If it expires while the disk is full (`AllDiskHolesActive` `6881`) → lose "disk full, no ball to collect"; or while the board is deadlocked (`IsBoardDeadlocked` `7056-7060`) → lose "no hole can reach the free slot"; there is also a short `FailAfterFinalHoleDelaySeconds = 0.5` terminal countdown (`CountdownDisplay.Finalizing`). A **new colour** arriving on the disk restarts the countdown (`OnDiskHoleAssigned` `6974-6989`). Then `Finished?.Invoke(false)` + camera shake (`PlayFailShake` `1237-1257`).

**Capture/colour rules in one line:** a slot accepts **only its own `BallType`** (`IsHoleCaptureStillValid` `3704-3718`, `TryBeginHoleCapture` `3633`), a hole starts pulling `HoleCaptureStartDelaySeconds=0.25` after it lands (`3639`), it reserves at most `limit - already-captured - in-flight` balls (`GetRemainingHoleCaptureSlots` `3693-3702`), and when it captures one ball it **also grabs same-colour neighbours within `AdjacentCaptureNeighborRadiusFactor=2.2` ball-radii inside the wider `AdjacentCaptureZoneScale=2` band** ("pull the ball behind", `ProcessAdjacentCaptures` `3565-3611`). Each capture is a fixed clock: wind-in duration scales with distance (`CaptureApproachDurationConfig=0.45`), plunge is fixed-duration (`CaptureFallDurationConfig=0.28`), and a hard cap `CaptureMaxDurationSeconds=4` guards the whole thing (`3411`). Near the end (`EndgameCaptureBoostHoleThreshold=5` holes left total) the capture band widens by `EndgameCaptureRadiusBoost=0.2` (`3344-3348`).

Two more lifecycles: **level entrance** (optional) — the bowl balls are loaded into two temporary pipes and **poured in**, flowing clockwise to their baked positions while the simulation is frozen (`_playEntrance`, `TryPrepareEntrance` `6240-6311`, `PlayEntrance` `6314-6355`, full subsystem `6232-6867`); **pipe exit** — when a level pipe empties its door swings open and the pipe slides off-screen (`NotifyBallLeftPipe` `5380-5405`, `PlayPipeExits` `5420-5500`).

**Restart:** `R` key → `SceneManager.LoadScene(gameObject.scene.name)` (`Update` `921-948`). The runner has no UI/SDK/analytics (README).

---

## 2. System / subsystem inventory

Line ranges are inclusive; "size" is approximate line count.

| # | Subsystem | Lines | ~Size | What it is |
|---|---|---|---|---|
| 1 | Header, usings, inspector fields, public API (`Finished`/`IsFinished`/`IsWon`), enums, JSON model classes, runtime-state classes | 1–397 | 397 | `LevelJson`/`BallJson`/`PipeJson`/`SegmentJson`/`BoardJson`/`CellJson`/`PaletteJson`; `BallState`, `DiskSlot`, `HoleView`, `GridCellView`, `PipeView`, `ResolvedPipe`, `EntranceBall/Stream`, palette structs. JSON field names **are** the JSON keys (`81`). |
| 2 | **Tuning constants** (all game values hard-coded here) | 399–763 | 365 | see §3. |
| 3 | Runtime state fields | 765–889 | 125 | balls list, grid buffers, pipes, disk slots, board dictionaries, entrance, outcome. |
| 4 | Unity entry points | 891–954 | 64 | `Awake` (camera fallback), `Start` (load JSON → `BuildLevel`), `Update` (R-restart, input, endgame spin, `PhysicsTick`, `LevelStateTick`), `OnDestroy` (`DOTween.KillAll`). |
| 5 | Level load + validation + unsupported-content warnings | 956–1064 | 108 | `TryLoadLevel` (JsonUtility, ball-count/ballRadius guards), `WarnAboutUnsupportedContent`. |
| 6 | Palette lookup (ball/hole colour per `BallType`, level overrides) | 1066–1114 | 49 | battery of `Get*Color` accessors + theme path builders. |
| 7 | **Level build** orchestration + camera | 1116–1235 | 120 | `BuildLevel`, `LoadPrefabs` (Resources.Load), `SetupEnvironment`, `BuildBowl`, `PlayEntranceThenStart`, `ApplyCameraTransform`. |
| 8 | Transform helpers | 1237–1268 | 32 | `PlayFailShake`, `StopFailShake`. |
| 9 | **Win/lose** | 1270–1301 | 32 | `WinLevel`, `LoseLevel`. |
| 10 | **Board build**: grid cells, holes, background, procedural walls | 1303–1722 | 420 | `BuildBoard`, `SpawnGridCell`, `SpawnBoardHole`, column centring, `ApplyGridBackgroundLayout`, checker pattern, `SpawnProceduralWalls` + `GenerateWallLoops`. |
| 11 | **Board input & sending** (tap → raycast → reserve slot → fly) | 1724–2028 | 305 | `HandleBoardInput`, `TryGetTap`, `OnHoleTapped`, `TryReserveSend`, `SendHoleToDisk`, `FlyHoleToDisk`, `EaseFlightProgress`, `ResolveLandScale`. |
| 12 | **Sendable state + pathfinding** (BFS to exit row) | 2030–2177 | 148 | `RefreshAllSendableStates`, `CanSendAnyHole`, `HasRouteToExit`, `BuildPathToExit` (BFS), `CanTraverseCell`, `IsExitCell`. |
| 13 | **Hole prefab instance** + colour/open-closed visuals + tap feedback + send trail | 2179–2690 | 512 | `CreateHoleView`, state visuals, `PlayClosedHoleOpenAnimation` (blend-shape lid), `PlayWobble`, `PlayDiskFullFeedback`, `PlaySendTrail`/`SyncSendTrail`. |
| 14 | **Disk**: slots, placeholders, progress labels, seat/despawn tweens, vacuum VFX | 2754–3313 | 560 | `CreateDisk`, `UpdateDiskVisual`, `AssignBallTypeToSlot`, `EnsureSlotVisual`, `AttachVacuumVfx`, `UpdateVacuumVfxEmission`, completion scheduling. |
| 15 | **Ball capture** (band/angle tests, gravity well, wind-in/dive/plunge animation, adjacent pulls, swallow visuals) | 3315–4092 | 778 | `ProcessHoleCaptures`, `BeginCapture`, `ProcessAdjacentCaptures`, `TryBeginHoleCapture`, `CompleteHoleCapture`, `ComputeCaptureVisualPosition`, ~20 shape/ease helpers. |
| 16 | **Bowl visual** (dish, height profile, near-side wedge, endgame glow) | 4094–4438 | 345 | `CreateBowlVisual`, `ComputeBowlHeightOffset`, `NearSideWeight`, `DirectionalInnerRadius`, `CreateBowlGlow`, `SetBowlGlowActive`. |
| 17 | **Balls** (spawn, prefab LOD tiering, shade variants, pipe-ball culling mesh, density maths) | 4440–4630 | 191 | `SpawnBakedBalls`, `CreateBall`, `ApplyShadeVariant`, `LoadBallPrefab` (LOD by visual scale), `ApplyPipeBallCullingMesh`, density/radius formulas. |
| 18 | **Pipes** (creation, materials, tunnel geometry, door, hex ball grid, exit) | 4632–5500 | 869 | `CreatePipes`/`CreatePipeView`/`SetupPipe`, `PipeLocalXZ`/`PipeGovernsArenaPosition`/`PipeClampArenaPosition`/NearestPointInTunnel, mouth seal state machine, hex grid generators, `PlayPipeExits`. |
| 19 | **Custom 2D physics solver** (the core simulation) | 5502–6230 | 729 | `PhysicsTick`, `BuildSpatialGrid` (CSR counting sort), `SolveBallCollisions`, `UpdateInwardSupport`, `SolveRegionConstraints`, `ProjectIntoValidRegion`, `EnforcePipeMouthBarriers`, `StabilizePipeMouthVelocities`, `ApplyHoleGravityWell`. |
| 20 | **Level entrance pour** (queue, slide, clockwise flow, landing) | 6232–6867 | 636 | `TryPrepareEntrance`, `BuildEntranceStream`/`Slots`, `UpdateEntranceStream/Sliding/Flowing/Landing`, `ApplyEntranceTransforms`, `AbortEntrance`, ease helpers. |
| 21 | **Level state machine** (win/lose countdown display, disk warning pulse) | 6869–7084 | 216 | `LevelStateTick`, `SetCountdownDisplay`, `RunFailCountdown`, `IsBoardDeadlocked`, `SetFailCountdownWarning`. |
| 22 | **Procedural wall mesh generation** (outline loops, bevel prism, ear-clip) | 7086–7782 | 696 | `GenerateOutlineLoops`, `BuildClosedLoops`, `OffsetFootprint`, `BuildWallMesh`, `EmitSideWalls`/`EdgeArcStrip`/`CornerPatch`, `EarClip`, polygon helpers. |

Notably: the file contains **no** `MonoBehaviour` other than the runner, and **no** coroutine-free threading — all animation is either DOTween, coroutines, or hand-integrated in `PhysicsTick`.

---

## 3. Tuning constants block (`399–763`) — the values

### Resources (Unity `Resources.Load` paths) — `403–424`
`Gameplay/Ball_LowPoly`, `Ball_MidPoly`, `Ball_HighPoly`, `BallMatUpper`, `BallMatLower`, `Gameplay/Pipe2`, `Pipe01Upper`, `Pipe01Lower`, `TubeGlassUpper`, `TubeGlassLower`, `Gameplay/PlatformBowl`, `PlatformBowl2`, `Gameplay/Prefabs/VFX/BowlGlow`, `VacuumVFX`, `Particles/HoleSendTrail`, `Gameplay/Hole`, `ClosedHole`, `ClosedHole_Proxy`, `GridCell`, `BoardGridBackground`, `Plane`.

### Arena / disk — `426–446`
`DefaultArenaRadius 7` · `DefaultBallRadius 0.2` · `TargetDensity 1` · `DiskRadius 3` · `ReferenceDiskRadius 3.5` · `HoleVisualRadiusConfig 1` · `HoleOrbitRadiusConfig 2.3` · `HoleCount 5` · `DiskBallPadding -0.7` · `DiskFillYScale 0.02` · `DiskVisualHeight -0.08` · `HolePancakeYScale 0.008` · `BallVisualLift 0.02` · `PositionEpsilon 0.0001` · `CollisionStiffness 0.5` · `MaxCorrectionPerPassFactor 0.5` · `LowPolyMaxBallScale 0.3` · `MidPolyMaxBallScale 0.6` · `InitialVelocityRange 1`.

### Bowl shape (near-camera softening) — `448–456`
`PhysicsInnerRadiusShrink 0.2` · `PhysicsInnerRadiusShrinkFarScale 1` · `NearSideInnerRadiusPadding 1` · `NearSidePaddingAngleDegrees 180` · `NearSidePaddingHeightScale 6` · `BowlBasinDepth -1.07` · `BowlHeightCurveExp 0.95` · `NearSideDepthScale 0`.

### Physics — `458–483`
`Damping 0.92` · `MaxSpeed 8` · `CenterPullStrength 30` · `SolverIterations 5` · `MaxSubstepMoveInnerBallRadii 2` · `MaxSubstepMoveOuterBallRadii 200` · `OuterJunctionZoneBallRadii 10` · `InnerMaxSpeedMultiplier 12` · `RadialOscillationDamp 0.25` · `GatePullByInwardSupport true` · `SupportContactCosThreshold 0.34` · `SupportDiskBandBallRadii 1.1` · `PullFadeOutSeconds 0.15` · `PullFadeInSeconds 0.08` · `JunctionSurfaceHysteresisBallRadii 5` · `PipeMouthBarrierCorrectionFactor 0.22` · `PipeMouthStabilizeZoneBallRadii 2.5` · `PipeMouthVelocityDamp 0.35` · `PhysicsStep 1/60` · `MaxPhysicsFrameTime 0.05` · `MaxPhysicsSubSteps 4` · `SpatialCellSizeMultiplier 2.25` · `DiskRotationSpeed 65` · `DiskEndgameRotationSpeed 200` · `DiskSpinAcceleration 200`.
(Ramp coroutine `5517-5537` temporarily overrides `CenterPullStrength→20`, `MaxSubstepMove{Inner,Outer}BallRadii→3` for 1 s after the pour.)

### Pipes — `485–514`
`EntranceLeftConfigIndex 1` · `EntranceRightConfigIndex 2` · `DefaultPipeWidth 2` · `PipeFlowTargetDensityDefault 0.6` · `PipeMouthSealHysteresis 0.08` · `PipeMouthStateHoldSeconds 0.1` · `PipeMouthDensitySmoothSeconds 0.25` · `PipeMouthSealBlendSeconds 0.15` · `ScalePullWithBallLoss true` · `CenterPullLossScaleMax 3.3` · `DoorOpenDuration 0.4` · `PipeExitDuration 0.6` · `DoorStartRotationX 180` · `DoorFinishRotationX 360` · `PipeExitLocalX 7.88` · `PipeExitLocalZTop 13.02` · `PipeExitLocalZBottom -13.02` · keyed tube positions · `DefaultPipeBallCullingBounds`.
`PipeConfigs[5]` — local positions/rotations for slots Middle/Left/Right/LeftBottom/RightBottom (`507-514`).

### Level entrance (pour) — `516–528`
`EntrancePipeWidth 2` · `EntranceBlockStagger 0` · `EntranceRowsPerBlock 5` · `EntranceSlideDuration 0` · `EntranceHandoffThreshold 1` · `EntranceRadialFormDuration 0` · `EntranceFlowAngularSpeed 450` · `EntranceMinSweepDuration 0.4` · `EntranceMaxSweepDuration 0.9` · `EntranceLandDuration 0.26` · `EntranceLandSquashScale 0.5` · `EntrancePipeExitDuration 0.45`.

### Hole capture — `530–579`
`DefaultColoredHoleCaptureLimit 35` · `HoleCaptureRadialHalfThicknessFallback 0.4` · `HoleCaptureAngleMarginFallback 0.6` · `ReferenceBallScale 0.3651003` · `DefaultReferenceHoleCaptureRadialHalfThickness 0.2` · `ReferenceHoleCaptureAngleMargin 0.6` · `AdjacentCaptureZoneScale 2` · `AdjacentCaptureNeighborRadiusFactor 2.2` · `EndgameCaptureBoostHoleThreshold 5` · `EndgameCaptureRadiusBoost 0.2` · `HoleCaptureStartDelaySeconds 0.25` · `HoleGravityWellStrength 9` · `HoleVortexSwirlStrength 7` · `HoleGravityWellRadiusBallRadii 9` · `CaptureNearLipBias 0` · `CaptureVanishDistanceFactor 0.4` · `CaptureContainmentRadiusFactor 0.6` · `CaptureFallDurationConfig 0.28` · `CaptureSquashAmount 0.3` · `CaptureSquashSpringHz 3.2` · `CaptureSquashDamping 0.45` · `CaptureArchHeightNear 0.16` · `CaptureArchHeightFar 0.55` · `CaptureApproachDurationConfig 0.45` · `CaptureApproachFullTravelHoleRadii 1.5` · `CaptureApproachShortTravelFactor 0.5` · `CaptureInwardNearSpeedMultiplier 1.5` · `CaptureApproachSweepDegrees 30` · `CaptureSwirlEnabled false` · `CaptureDiveEnabled true` · `CaptureDiveDescentPower 4` · `CaptureDiveCoilT 0.15` · `CaptureDiveCoilDip 0.3` · `CaptureDiveCoilSquash 0.22` · `CaptureDiveTipOverStartT 0.38` · `CaptureDiveTipOverEndT 0.72` · `CaptureDiveEntryDepth 1.6` · `CaptureDiveEntryStartT 0.62` · `CaptureDiveEntryPeakFallT 0.5` · `CaptureDivePlungeSink 0.6` · `CaptureDiveEntryStretch 0` · `CaptureMaxDurationSeconds 4` · `CaptureBallRenderQueue 3010` · `CaptureVarianceSpread 0.22` · `CaptureArchLayerSpread 0.35` · `CaptureSquashOvershootHeadroom 1.5` · `CaptureAimTurnRate 13` · `CaptureVanishStartT 0.82` · `CaptureDiveSinkMaxHoleRadii 0.35`.

### Disk holes (visuals) — `581–591`
`HoleCloseHoldAtFullPercentSeconds 0.2` · `HoleCloseHopHeight 0.45` · `HoleCloseHopDuration 0.12` · `HoleCloseHopScaleUp 1.15` · `HoleCloseSinkDuration 0.2` · `HoleSeatDuration 0.18` · `VacuumVfxLocalY 0.13` · `ProgressLabelShownLocalY 1.2` · `ProgressLabelShownXRotation 40` · `ProgressLabelShownScale 0.8`.

### Level state — `593–595`
`FailCountdownSeconds 5.5` · `FailAfterFinalHoleDelaySeconds 0.5`.

### Board — `597–629`
`BoardWorldPosition (0,7.55,-24.65)` · `BoardWorldRotation (0,0.97629607,-0.2164396,0)` · `BoardReferenceDimension 8` · `BoardReferenceScale 2.2` · `CellSpacing 1` · `UnitYOffset 0.12` · `JumpDuration 0.35` · `JumpPower 4.8` (=`1.2*4`) · `SendCruiseEase 0.2` · `SendStreakOutAt 0.78` · `SendLandScale 0.94` · `SendLandTiltX 25` · `JumpPlaceholderHideLead 0.01` · `EdgeWallColumnsPerSide 4` · `GridBackgroundMarginColumns 0.55` · `GridBackgroundMarginRows 0.02` · `ProceduralWallHeight 0.5` · `ProceduralWallBevelSize 0.1` · `ProceduralWallBevelSegments 4` · `ProceduralWallGapCells 0.03` · `UnsendableFlashInDuration 0.08` · `UnsendableFlashOutDuration 0.35` · `UnsendableFlashColor (199,81,76)/255` · `BoardInputLayerMask 55` · `NeighborOffsets` (4-way).

### Hole prefab look — `631–662`
`HoleSendableScale 0.95` · `ClosedHoleValueScale 0.9` · `SendTrailHeadWhiten 0.15` · `SendTrailMinValue 0.75` · closed shadow (`0/0.15/0.52`), open shadow (`1/0/0.8`) · `ClosedHoleProxyBlendShapeName "FaceBlendShape.Target_Hole_Proxy"` · `StretchScale (0.72,1.45,0.72)` · `SquashScale (1.15,0.8,1.15)` · `StretchRiseDuration 0.1` · `StretchHoldDuration 0.08` · `ScaleBackDuration 0.18` · `BlendshapeCurve` (7 keys) · `SendTrailAlphaShape` (3 keys).

### Camera shake / bowl glow / shader property IDs — `664–687`
`FailShakeDuration 0.45` · `FailShakePositionStrength (0.1,0.14,0.03)` · `FailShakeRotationStrength (0.6,0.4,0.9)` · `BowlGlowCloseScale 0.3` · `BowlGlowOpenFadeSeconds 0.35` · `BowlGlowCloseFadeSeconds 0.6`. Shader property IDs: `_BaseColor`, `_ShadowTint`, `_GlossTint`, `_OutlineColor`, `_Hole_Color`, `_OutlineWidth`, `_Shape`, `_Corner`, `_Radius`, `_WobbleStartTime`, `_WarningIntensity`, `_MainAlphaChannel`, `_GlossPoseRefAxis` (`675-687`).

### Palettes — `689–763`
`BallPalettes[16]` and `HolePalettes[16]` — each entry = 4 `Color` values (runtime/inside, shadow/border, gloss, outline) indexed by `BallType` 0–15 (`697-735`). `VacuumColors[16]` (`738-756`). `BallShadeVariants[3]` = `(0,0)`, `(0.075,-0.06)`, `(-0.075,0.06)` (`758-763`). Default tints at `692-695`.

---

## 4. Physics weight

**There is no Unity physics for the simulation.** Grep across the whole file for `Rigidbody`, `PhysX`, `Gravity`, `Joint`, `OverlapSphere`, `Collision`:
- `zero occurrences of Rigidbody` and `zero of Physics.Gravity` across `LoopHoleLevelRunner.cs` (7,783 lines).
- Unity physics is used for exactly two things:
  1. **Input raycasting only** — `Physics.Raycast(ray, out hit, 100f, BoardInputLayerMask /*55*/, QueryTriggerInteraction.Ignore)` (`1741`) against the hole prefabs' `BoxCollider` (`2191`), looked up in a `Dictionary<Collider, HoleView>` (`871`, `1746`). The tap colliders are counter-scaled to keep authored world size (`ApplyTapColliderCompensation` `2485-2491`) and are **disabled on disk-slot visuals** (`3027-3030`).
  2. **`MeshCollider` on the procedurally generated wall meshes** (`1654`) — present but never queried by this class.
- Balls have **no** collider: any collider the ball prefab carries is destroyed on spawn (`4489-4494`).

The simulation is a **hand-rolled 2D (XZ-plane) position-based solver**, in arena space where `x = world X`, `y = world Z` (`5503`). Per `PhysicsTick` (`5553-5713`) it:
1. clamps `dt` to `MaxPhysicsFrameTime 0.05`, computes `subSteps = clamp(ceil(dt/PhysicsStep), 1, 4)` and runs each substep at `dt/subSteps` (substep because collisions are hard positional pushes, not impulses);
2. **integrate** — centre pull `CenterPullStrength * dynamicScale * (support-gated) * pipeFlowFactor` toward origin, plus the hole gravity well, then `position += velocity * dt` (capped `MaxSpeed 8`);
3. run `SolverIterations = 5` **Gauss-Seidel** passes of `SolveBallCollisions` + `SolveRegionConstraints`;
4. enforce pipe-mouth barriers, transition pipe→bowl balls, update the inward-support gate, clamp per-substep move (`2` ball-radii inner / `200` outer), recompute velocity `(pos - substepStart)/dt`, apply `Damping^ (dt*60)`, radial viscosity, speed caps (inner balls ×12), then stabilise pipe-mouth velocities.
5. Diffuse: `ProcessHoleCaptures` + `SyncAllBallTransforms`.

**Collision detection is a flattened CSR counting-sort uniform grid** (`BuildSpatialGrid` `5752-5825`): cell size `ballRadius*2 * SpatialCellSizeMultiplier(2.25)`, one-cell border, 3×3 neighbour scan, ball-vs-ball only. There is no AABB tree, no broadphase for static geometry — walls/rim/disk/pipe are all handled by analytic region projection (`ProjectIntoValidRegion` `6048-6071`, clamping to annulus `[DiskRadius+ballRadius, arenaRadius-ballRadius]`, pipe tunnels, or nearest valid surface with pipe-vs-rim hysteresis).

**Cost drivers per frame, in the order they are paid:**
- `SolverIterations × substeps` **full grid rebuilds** — `BuildSpatialGrid` is called **inside** `SolveBallCollisions` (`5848`), so a 4-substep frame rebuilds the grid up to **20 times/frame**.
- 3×3-neighbour pair scan per grid pass (bounded by local density).
- `ProcessHoleCaptures` loops all balls each frame (`3386-3485`) and calls `CountBallsInFlightToHole` which itself scans all balls (`3682-3688`) → each ball processed can trigger an O(N) scan. `HasCollectableBallForAssignedHoles` (`3721-3756`) scans all balls then all holes.
- Per-frame `UpdateInwardSupport` (`5928-6006`) does a second 3×3 grid scan per ball.
- `ApplyHoleGravityWell` (`6170-6219`) is per ball per substep and scans up to 5 slots.
- `UpdatePipeMouthSealState`/`GetBowlAnnulusDensity` (`5223-5262`, `5332-5347`) scan all balls and, per ball, call `TryGetGoverningPipe` → `PipeGovernsArenaPosition` per pipe (`5338`, `5191-5204`).
- `SyncBallTransform` per ball each frame; it also repairs parentage (`4042-4045`).

**How many bodies alive at once:** up to `ballEntries.Count`. The shipped levels are 350 / 5,000 / 15,000 balls (`Level_54/21/23.json`). Balls are never pooled — each is `Instantiate`d (`4489`) and `Destroy`ed on capture (`3766`) / on removal (`5720-5723`). There is **no object pool** for balls.

**Pooling / spatial structures already present:** the CSR grid + preallocated `Vector2[]`/`float[]`/`int[]` buffers (`EnsurePhysicsBuffers` `5730-5749`, `EnsureSealStateArrays` `5320-5330`); a shared capture-material cache keyed by source material (`CaptureMaterialsBySource` `838`, `3807-3828`); a single shared pipe-ball culling mesh (`4586-4602`). No transform-instancing / GPU-instancing for balls (see §5).

**Determinism note:** the solver is deterministic in structure but consumes `UnityEngine.Random` at spawn (`InitialVelocityRange`, `4523`), at capture (`CaptureVariance`, `3505`), for shade variants (`4536`) and for pathological-coincident overlap angles (`5894`, derived from indices). Simulation depends on `Time.deltaTime`, so **per-frame timestep → frame-rate dependent** (no fixed accumulator beyond the substep clamp).

---

## 5. Rendering weight

- **Instancing:** none. Balls are individual `GameObject`s (prefab instances) each with its own `MeshRenderer` and a `MaterialPropertyBlock` for per-ball `_BaseColor`/`_ShadowTint`/`_GlossTint`/`_OutlineColor` (`4508-4517`). Holes likewise carry 4 property blocks (`252-255`). No `Graphics.DrawMeshInstanced`, no `MeshRenderer.SetPropertyBlock` batching beyond Unity's SRP batcher expectations.
- **Meshes / LOD:** three ball prefabs — `Ball_LowPoly`, `Ball_MidPoly`, `Ball_HighPoly` (`404-406`). The tier is chosen **once** from the ball's visual scale (`LoadBallPrefab` `4552-4573`): low if scale ≤ `LowPolyMaxBallScale 0.3`, mid if ≤ `0.6`, else high. Shipped levels: `ballRadius` 0.286 → scale ~0.78 → high-poly tier (`Level_54`); 0.077 → ~0.21 → low-poly (`Level_21`); 0.091 → ~0.25 → low-poly (`Level_23`).
- **Prefab meshes referenced (loaded from `Resources`):** ball LODs, `Gameplay/Hole`, `Gameplay/ClosedHole`, `Gameplay/ClosedHole_Proxy`, `Gameplay/GridCell`, `Gameplay/BoardGridBackground`, `Gameplay/Plane`, `Gameplay/Pipe2` (with children `MouthAnchor`/`Tube`/`Tube/Door`), `Gameplay/PlatformBowl`, `Gameplay/PlatformBowl2`.
- **Shaders/materials:** the runner never names a shader; it loads Materials by path and writes these properties: `_BaseColor`, `_ShadowTint`, `_GlossTint`, `_OutlineColor`, `_Hole_Color`, `_OutlineWidth`, `_Shape`/`_Corner`/`_Radius` (contact-shadow silhouette morph `ApplyShadowShape` `2470-2482`), `_WobbleStartTime` (lid wobble `2515-2532`), `_WarningIntensity` (disk warning `2971-2982`), `_MainAlphaChannel` (bowl-glow wind `4406-4420`), and the **global** `_GlossPoseRefAxis` set from the board rotation (`1347`). Materials are per theme: `Themes/BackgroundThemes/{theme}/*`, `Themes/BowlThemes/{theme}/*` (`1113-1114`), plus `BallMatUpper/Lower`, `Pipe01Upper/Lower`, `TubeGlassUpper/Lower`, `Pipe02{Upper,Lower}`, `TubeLines{Upper,Lower}`.
- **VFX/particle prefabs referenced and what they do:**
  - **`Gameplay/Prefabs/VFX/BowlGlow`** (`416`) — the endgame glow, instanced under the bowl (`CreateBowlGlow` `4333-4362`). Internally it looks up child particle systems `Glow`, `Arrow Rain_Wind 1`, `Arrow Rain_Wind 1/WindRibbon_DoubleSided` (`4346-4348`); a DOTween ramp drives scale and the wind's master alpha while the disk spins fast at endgame (`SetBowlGlowActive` `4370-4396`).
  - **`Gameplay/Prefabs/VFX/VacuumVFX`** (`417`) — one per filled disk slot (`AttachVacuumVfx` `3157-3171`); its particle colour-over-lifetime gradient is rebuilt from `VacuumColors[BallType]` (`ApplyVacuumColor` `3173-3205`, gradient stops at `578/42663/42484/56002` of 65535). Emission is enabled only while that slot is actively pulling a ball (`UpdateVacuumVfxEmission` `3236-3259`). Destroyed when the slot frees (`ReturnVacuumVfx` `3224-3234`).
  - **`Particles/HoleSendTrail`** (`418`) — a `TrailRenderer` spawned **on the board** (not under the shrinking hole) behind a hole flying to the disk (`PlaySendTrail` `2593-2617`); per-hole `Gradient` colour (HSV value floored at `SendTrailMinValue 0.75`) and a 3-key alpha shape (`657-662`); detached and self-`Destroy`ed after `trailRenderer.time` (`ReleaseSendTrail` `2668-2690`).
  - Wall meshes are generated procedurally (`BuildWallMesh` `7277-7382`) — a beveled prism with quarter-circle top bevel and quarter-torus corners, top-cap ear-clipped (`EarClip` `7661-7711`).
- **Hole lid opening** uses a **SkinnedMeshRenderer blend shape** `FaceBlendShape.Target_Hole_Proxy` driven from an `AnimationCurve` (`2319-2320`, `2383-2442`, curve `649-655`) — i.e. a skinned mesh, not a shader deform.
- Progress on each disk slot is a **TextMeshPro** `3D` text component ("0%"…) created in code (`2808-2829`).

---

## 6. Level JSON contract

Field names are the JSON keys verbatim (`81`, classes `83-156`). Deserialized with `UnityEngine.JsonUtility.FromJson<LevelJson>` (`965`). Structure:

```
LevelJson
  int   levelId
  int   coloredHoleCaptureLimit      = 100
  bool  overrideColoredHoleCaptureLimit
  bool  overrideReferenceHoleCaptureRadialHalfThickness
  float referenceHoleCaptureRadialHalfThickness = 0.2
  float arenaRadius
  float ballRadius
  List<BallJson>    ballEntries
  List<PipeJson>    pipes
  BoardJson         board
  List<PaletteJson> paletteOverrides

BallJson  { int ballType;  Vector3 position }                    // position = x, y(vertical), z
PipeJson  { int configIndex; int ballCount; float width = 2;
            float flowTargetDensity = 0.8; List<SegmentJson> segments;
            List<Vector2> ballLocalPositions }
SegmentJson { int ballType; int count }
BoardJson { int rows; int columns; float scale; int mode;
            List<CellJson> cells; List<CellJson> spawners;
            List<CellJson> blockerPanels; List<CellJson> elevators }
CellJson  { int cellType; int unitType; int objectKind;
            bool isMystery; bool isFreeze; bool isBomb; int pairId }
PaletteJson { int ballType; Color ball; Color inside; Color border }
```

**Note the real files carry extra keys the class ignores** (Unity `JsonUtility` simply drops unknown keys): shipped JSONs also contain `isHardLevel`, `ballCount`, `holeCount`, `sourceImageId`, `columnSlideColumns`, `freezeCounter`, `freezeOrder` (verified by reading all three JSONs).

**Entity types handled vs ignored** (warnings emitted by `WarnAboutUnsupportedContent` `1020-1059`):

| Content | Handling |
|---|---|
| `cellType == 1` with visible `unitType` → a hole | **Handled** (`BuildBoard` `1351-1378`, holes spawned only when `objectKind == 0`) |
| `cellType == 1`, `unitType == 0` (None) | Ignored silently |
| `cellType == 2` → a wall cell | **Handled** (fed to `GenerateWallLoops` `1688-1704`) |
| Other `cellType` values | Ignored silently |
| `cell.objectKind != 0` (key/lock) | **Ignored**, warned (`1042`, `1049`); that cell spawns no hole |
| `isMystery` | **Ignored**, warned (`1043`, `1050`) |
| `isFreeze` | **Ignored**, warned (`1044`, `1051`) |
| `isBomb` | **Ignored**, warned (`1045`, `1052`) |
| `pairId != 0` | **Ignored**, warned ("sent one by one") (`1046`, `1053`) |
| `board.spawners` | **Ignored**, warned (`1026`) |
| `board.blockerPanels` | **Ignored**, warned (`1027`) |
| `board.elevators` | **Ignored**, warned (`1028`) |
| `board.mode != 0` (column-slide) | **Ignored**, warned (`1025`) |
| `pipes[]` with `ballCount > 0` and valid `configIndex` 0–4 | **Handled** (`CreatePipes` `4657-4712`) |
| `pipes[]` out-of-range `configIndex` | Skipped with warning (`4671`) |
| `paletteOverrides[]` | **Handled** when `_applyPaletteOverrides` (`1002-1014`) |

`TryLoadLevel` hard-fails (returns false, disabling the component) on: no JSON assigned (`905-910`), parse exception (`967-971`), **zero `ballEntries`** (`973-977`), or **`ballRadius <= 0`** (`979-983`, "legacy level"). `board.cells` / `pipes` are null-coalesced to empty (`985-987`).

**Measured level shapes (these three files):**

| File | levelId | arenaRadius | ballRadius | ballEntries | pipes (ballCount each) | board rows×cols | cells (type1/type2) | captureLimit | paletteOverrides |
|---|---|---|---|---|---|---|---|---|---|
| `Level_54.json` | 54 | 7.0 | 0.28572 | 350 | 4 (400 each) | 8×7 | 25 / 31 | 100 | 0 |
| `Level_21.json` | 21 | 7.0 | 0.07731 | 5000 | 0 | 6×16 | 56 / 40 | 100 | 14 |
| `Level_23.json` | 23 | 13.0 | 0.09091 | 15000 | 0 | 9×8 | 24 / 48 | 200 | 2 |

Cell counts are exactly `rows*columns`, row-major; `cellType` is 1 (hole) or 2 (wall) in these files; all `objectKind` are absent (no blockers in the shipped trio). Holes per level: 25 / 56 / 24; `unitType` values span 1–14.

---

## 7. Hard-in-a-Cocos-web-playable items (facts only)

Observations tied to the source; these are things the Cocos port must reproduce, each with the locator that makes it a constraint. No recommendation is made.

- **15,000 individual bodies, one `GameObject`+`MeshRenderer`+`MaterialPropertyBlock` each** (`4486-4530`), no pooling, `Instantiate`/`Destroy` per ball (`4489`, `3766`). The whole solver is CPU, O(N) per pass over ~20 grid passes/frame (§4). This is a **CPU-simulation and draw-call-count** constraint, not a physics-engine constraint — there is no engine physics to port.
- **Collision response is a hand-written Gauss-Seidel positional solver** with `SolverIterations 5` inside up to `MaxPhysicsSubSteps 4`, grid rebuilt per pass (`5844-5925`, `5568`, `5768-5824`). Determinism depends on `Time.deltaTime` (`5560`, substep clamp) — frame-rate-coupled.
- **`UnityEngine.Random`** is used for initial velocities and per-capture variance (`4523`, `3505`), so a re-implementation must supply a comparable RNG or the captures will not match.
- **Region constraints are analytic, not engine colliders**: annulus clamp, pipe-tunnel local-space clamp (`PipeClampArenaPosition` `5109-5116`), nearest-valid-surface with hysteresis (`6073-6109`). All expressed in an arena plane where world Z is the plane's Y (`5503`) — the Cocos port must decide its own axis convention.
- **Custom shader properties** written per object per frame: `_ShadowTint`, `_GlossTint`, `_OutlineColor`, `_OutlineWidth`, `_Shape`/`_Corner`/`_Radius`, `_WobbleStartTime`, `_WarningIntensity`, `_MainAlphaChannel`, and a **global** `_GlossPoseRefAxis` (`675-687`, `1347`). These are custom toon-shader contracts; Cocos has no equivalent built in.
- **Blend-shape (skinned mesh) lid opening** on `ClosedHole_Proxy` (`2318-2320`, `2395`, `2417-2420`) with an authored `AnimationCurve` (`649-655`) and a `SkinnedMeshRenderer.SetBlendShapeWeight` — Cocos Creator's web build supports morph targets only via specific import paths / 3D material features; this is a model-and-shader dependency.
- **Per-ball LOD tier chosen from scale** (`4552-4573`), with three separate prefabs; the port needs three meshes and the same scale thresholds (`0.3`, `0.6`).
- **Pipe-ball frustom-culling override**: pipe balls use a shader that curves vertices, so the class swaps in a mesh copy with oversized `bounds` (`4586-4602`, bounds `DefaultPipeBallCullingBounds` `505`). A Cocos port needs an equivalent culling-bounds escape.
- **Mesh generation at runtime** for walls (`BuildWallMesh` `7277-7382` incl. ear-clipping, miter insets, bevel torus) — geometry built in C# `Mesh` API, not authored assets.
- **DOTween is used heavily** for all UI/board/disk animation (`PlayClosedHoleOpenAnimation`, `PlaySeatAnimation`, `PlayDespawnAnimation`, `PlayDiskFullFeedback`, fail shake, bowl glow, label pop, endgame spin tweens) and `DOTween.KillAll()` is called on destroy (`953`) — must be replaced by a Cocos tween system.
- **Coroutines** drive the pour (`PlayEntrance` `6314-6355`), pipe exits (`5420-5500`), the send flight (`1908-1969`), and the fail countdown (`7015-7054`).
- **`TextMeshPro` 3D text** created in code for disk progress labels (`2820-2829`).
- **Unity-only APIs in use:** `JsonUtility` (`965`), `Resources.Load`/`Resources.GetBuiltinResource` (`1156-1163`, `2797`, `2868`), `SceneManager.LoadScene` (`925`), `Physics.Raycast` (`1741`), `Camera.ScreenPointToRay` (`1740`), `MaterialPropertyBlock`, `MeshCollider`/`BoxCollider`, `Shader.PropertyToID`/`Shader.SetGlobalVector`, `ParticleSystem` emission/`colorOverLifetime`, `TrailRenderer`, `Gradient`/`AnimationCurve`, `Color.RGBToHSV`/`HSVToRGB`, `Vector2/3`, `Mathf`, `Time`, `Input` (`touchCount`/`GetMouseButtonDown`/`GetKeyDown`), `Application` none, `SkinnedMeshRenderer`.

---

## Could not measure

- **Runtime performance / frame timing of the shipped levels** — no Unity project was opened or run; per-frame cost claims in §4 are static call-structure analysis, not profiler output. Verbatim basis: the file only; no `Profiler`, no `Time.deltaTime` sample exists off the hot path.
- **Actual particle-system counts / live ball counts mid-frame** — not instrumented; ball counts in §4 are the JSON `ballEntries` sizes, which is an upper bound per spawned ball, not an observed concurrent count.
- **Whether all 15,000 balls are ever resident simultaneously in `Level_23`** — the code path (`PipeIndex`) supports a pipe-held state, but `Level_23.json` has `pipes: []`, so all 15,000 are spawned as bowl balls (`SpawnBakedBalls`). This is a static reading of the file + JSON; not observed at runtime.
- **Cocos-side feasibility of any single item in §7** — out of scope; no Cocos project was inspected. No estimate or recommendation is given here (owner for evaluation/trade-offs: `t1k-researcher`; for measurable Cocos cost numbers, a profiling pass, not this report).

---

## UNVERIFIED LEADS — not findings

Each is beyond the brief; the command that would check it follows.

- The unsupported blocker content (mystery/freeze/bomb/pair/key-lock/spawner/panel/elevator) has no code path here at all — the shipped levels exercise none of them. `rg -n "isBomb|isFreeze|isMystery|pairId|objectKind" C:/WORK/VIDEO/LoopHole/Assets/LevelRunner/LoopHoleLevelRunner.cs` shows only the warning collector.
- The ball/hole/VFX prefabs and the toon shaders live under `Resources/Gameplay`, `Resources/Themes`, `Resources/Particles` and `_Art` (README lines 47-48); their internal structure (submesh counts, shader keys) was not read. `ls -R C:/WORK/VIDEO/LoopHole/Assets/Resources C:/WORK/VIDEO/LoopHole/Assets/_Art`.
- `PipeConfigs` covers 5 slots but `EntranceRightConfigIndex/Left` reuse slots 2/1; whether any level uses all four pipe slots simultaneously is a data question across all level files, not just the three shipped. `rg -o "\"configIndex\":[0-9]+" <levels-dir>/*.json | sort | uniq -c`.
- The reference game build (`lh-0.0.3`, commit `a64c854`, per file header line 3) is not in this repo, so "the game's values" claims in the header (line 400) could not be cross-checked against the shipping game.
- `LevelRunner.unity` scene wiring (which camera/prefab the Inspector references) was not read. `Read C:/WORK/VIDEO/LoopHole/Assets/LevelRunner/LevelRunner.unity`.
