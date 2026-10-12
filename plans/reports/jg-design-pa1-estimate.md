# Jigsaw Card Swap Playable (JG Design PA 1) — System Design & Development Estimate

**Date:** 2026-10-08   **Engine:** Cocos Creator 3.8.7 (2D)   **Built on:** CocosPlayableAdsTemplate
**Source spec:** `[JG] Design PA 1/client-brief.txt` · `[JG] Design PA 1/game.html` (reference build, Cocos 3.8.7 super-html single file; unpacked and played headless)
**Basis:** 1 dev · ideal hours · 8h = dev-day · 5 dev-days = week
**Estimate points:** O = L = **Base** (the plan number) · P = **Worst-case** (base + risk on the downside only). Gap is risk-driven, not a uniform band.

This report IS the system design + the estimate. `/t1k:cook` builds straight off §§5–8 with no separate planning pass.

---

## 2. Project understanding & gameplay

**Core loop (read back and confirmed by the requester on 2026-10-08):**
> Loading screen (logo + progress bar) → tutorial finger drags one card onto the spot that completes a picture. Board = N columns (columns param, up to 6) of face-up cards; above each column is a face-down stack. Player drags a card (or a merged group) onto another to swap them; correctly-adjacent pieces of the same picture merge with a shared border. When all pieces of a picture (4 cards in 2×2, or 2 in 1×2) are assembled, it clears with FX, the column cards fall down and new cards drop from the stacks above. Start board + stack queues are auto-generated from the image collection (runtime-sliced). Win = "picture count" param reached → end card (logo + Play). No lose state, no timer.

**Genre / structure:** 2D portrait jigsaw "picture swap" puzzle, one level, 3 phases: Loading → Gameplay (with tutorial hint) → End card. No 3D, no rigged models.

**Reference decode (measured in the running reference, not assumed):** 6×6 board (36 slots), 6 face-down stacks (108 cards total = 36 on board + 72 queued), 20 pictures (17 of 2×2, 3 of 1×2), win after `maxSuccessNum = 4` pictures, `swapDistance 100`, star-burst FX on clear, rounded-corner frame per group shape (10 corner/edge materials). The reference uses **pre-sliced** pieces and a **hand-authored** 6×6 start layout; this build replaces both with runtime slicing + a generator (client decision).

| Question | Answer |
|---|---|
| On-screen elements | Top: N face-down card stacks · Middle: N-column board of face-up picture pieces · Bottom: game logo · Background (image/colour param) · tutorial finger |
| The ONE primary interaction | Drag a card (or merged group) and drop it on another cell |
| What it does | Swaps the two positions; adjacent correct pieces merge into a group (shared frame); a finished picture clears, columns collapse, new cards drop from the stack above |
| Win | `picture count` param reached (reference default 4) |
| Lose / stuck | None (no timer, no move limit). The generator guarantees a solvable board, and the refill keeps it solvable |
| Store redirect | End card Play button (+ template standard triggers, e.g. after N interactions, available as dashboard params) |

## 3. Scoping assumptions

| Driver | Decision | Effect on number |
|---|---|---|
| Starting point | Reuse framework (CocosPlayableAdsTemplate / PLAGameFoundation / cocos-stdlib), not greenfield | Flow, CTA, loading, end card, tutorial, audio, pooling are wire-only |
| Reference authority | **C · mixed** — gameplay/feel = game.html exactly; loading, bottom logo, end card = brief | Screens priced from template views; mechanic priced from ref decode |
| Secondary mechanics IN | Group merge + group drag; 1×2 (2-card) pictures | Own module rows (`grouping`), generator handles 2 shapes |
| Secondary mechanics OUT | Rotating promo text labels, any lose/timer/move limit | Not billed |
| Image collection param | **Runtime auto-slice**: client drops full pictures, game cuts them 2×2 / 1×2 | COMPLEX param → billed `picture-source` module |
| Columns param | Configurable, **max 6**, board **auto-generated** (seeded, solvable, not pre-solved) | COMPLEX param → billed `board` generator + layout fit |
| Other params | Background image/colour, card tint colour, picture count | BASIC → parameter dashboard default, $0 |
| Art | Client supplies all final art (logo, card back/frame, background, pictures, buttons, SFX/BGM) | Import + wire only; no art production |
| Level editor | **Not wanted** — board is generator + params only | WP10 not billed |
| Orientation | Portrait only | Single layout pass |
| Ad networks / size | Studio default: all networks, hard **4.8 MB** cap | SDK/export = excluded default; size sized in Build-size package |
| Team | Single dev, serial, no deadline | No parallel discount |

**Out of scope:** rotating promo texts, lose state / timer, landscape, visual level editor, localization, multiple aspect-art sets, remote config, analytics beyond standard hooks.

## 4. Reuse basis

`discovery: live-corpus` — Knowledge MCP `doc_search`, 10 capability queries across the whole corpus on 2026-10-08 (engine filter `cocos` plus one cross-engine query). Assemblies seen: CocosPlayableAdsTemplate, PLAGameFoundation (`tutorials`, `objectPool`), cocos-stdlib (`packages`), Cocos3DOTS_PA2, CocosPixelDefendersPLA, CocosH5CocosTemplate, PlayableParamterTool.

| Need | Member(s) found | Tag | Feeds module |
|---|---|---|---|
| Game flow, win → end card, store CTA | `CocosPlayableAdsTemplate#GameView.onWin`, `EndCardView` (+ `actionButton`, `fullScreenCTA`, `onActionButtonClicked`); ≥9 sibling PLAs share it | ● | bootstrap, end-cards |
| Loading screen + progress bar | `CocosPlayableAdsTemplate#LoadingView.FillBar` | ● | loading |
| Tutorial drag hint | `PLAGameFoundation/tutorials#FingerTutorial`, `FingerAction.DRAG`, `dragTargetNode`, `setTarget`, `hide` | ● | tutorial |
| Input | `CocosPlayableAdsTemplate#InputService` | ● | piece (drag) |
| Drag snap glide | `CocosPixelDefendersPLA#HeroDragController.changeSnap` (glide-in/out pattern) | ◐ | piece (drag) |
| Column gravity + refill | `Cocos3DOTS_PA2#RefillPlanner.plan` (bottom-up write cursor), `Grid.clear`, `BoardView.applyMutation` (replay slide-down + enter-from-above) | ◐ | cascade |
| Seeded RNG + shuffle | `cocos-stdlib packages#DeterministicRandom`, `shuffleInPlace`; `Cocos3DOTS_PA2#Rng` | ● | board (generator) |
| Motion / juice presets | `MotionPresetConfig` (`popIn`, `pulse`, `collect`; 9+ projects), `cocos-stdlib#TweenReactions.playCelebrate` | ◐ | polish, vfx |
| Object pooling | `PLAGameFoundation/objectPool#ObjectPool` | ● | vfx, cascade |
| Image params (base64 sprite swap) | `PlayableParamterTool#replaceParameterSprite`, `replaceParameterSpriteProperty` | ◐ | picture-source |
| Parameter dashboard | `SpriteConfig` / PlayableConfig (template) | ● default | — |

**Stays custom (○, nothing found in the sweep):** runtime **picture slicing** into 2×2/1×2 SpriteFrames (searched: "slice texture into sub SpriteFrames at runtime" — only 9-slice/param-swap hits); **jigsaw group merge + group swap** (searched: "pieces adjacent correct neighbours merge into group" — no Cocos hit; the only swap prior art is Unity DOTS `DOTSPuzzle.Input.PieceExchange`, not portable); the **solvable, not-pre-solved board + stack generator**; the group-shaped frame/corner visuals.

## 5. Module architecture

`[Name-Project]` is a placeholder. Rename it to the real project folder before building.

**Root:** `assets/scripts/[Name-Project]/`

| Module | Files | Responsibility | Upstream | Status |
|---|---|---|---|---|
| `bootstrap` | `bootstrap/GameplayController.ts` | Orchestrator: flow states, wires signals → GameView/CTA | `CocosPlayableAdsTemplate` GameView | TBD |
| `picture-source` | `picture-source/PictureLibrary.ts`, `picture-source/PieceSlicer.ts` | Which pictures exist, and what are their piece SpriteFrames? |  | TBD |
| `board` | `board/BoardModel.ts`, `board/BoardGenerator.ts`, `board/BoardLayout.ts` | Where is every card, and what does a fresh solvable board look like for N columns? |  | TBD |
| `piece` | `piece/PieceView.ts`, `piece/PieceDragController.ts`, `piece/StackView.ts` | How does a card look, and how is it dragged and dropped? |  | TBD |
| `grouping` | `grouping/GroupResolver.ts` | Which pieces are correctly joined, and how does a group swap? |  | TBD |
| `cascade` | `cascade/PictureClearer.ts`, `cascade/RefillController.ts` | What happens when a picture completes: clear, collapse, refill? |  | TBD |
| `objective` | `objective/ObjectiveController.ts` | How many pictures done, and is the game won? |  | TBD |
| `tutorial` | `tutorial/TutorialController.ts` | Where does the hint finger point? | PLAGameFoundation `FingerTutorial` | TBD |
| `vfx` | `vfx/VFXController.ts` | How do merge / clear / win events look? | PLAGameFoundation `ObjectPool` | TBD |
| `audio` | `audio/AudioHooks.ts` | Which sound plays on which event? | template `AudioService` | TBD |

Loading screen and end card are template views (`LoadingView`, `EndCardView` subclass) and are configured, not new modules. Rule: modules never import each other; they talk only via `SignalBus`, and only `GameplayController` touches `GameView`/`CTAService`. `BoardModel`, `BoardGenerator`, `GroupResolver` and `RefillController`'s planner are pure TS, so Vitest can cover them. Tests are colocated in dot-prefixed `.tests/` folders.

## 6. Communication flow

```
STARTUP
  GameplayController.start → LoadingView (FillBar) ─ params ready ─→
  PictureLibrary.load(params.images) → PieceSlicer.slice(each) ──────→ emits PICTURES_READY
  BoardGenerator.generate(cols, pictures, seed) → BoardModel (board + N stack queues)
  BoardLayout.fit(cols) → PieceView/StackView spawn (pooled) ─────────→ emits BOARD_READY
  TutorialController ← BOARD_READY : finds the 1-move completion → FingerTutorial DRAG A→B

PLAYER INPUT
  PieceDragController (InputService) : pick → emits PIECE_PICKED (tutorial hides)
  drop on cell → BoardModel.swap(group|piece, target) → emits PIECES_SWAPPED
  GroupResolver ← PIECES_SWAPPED : recompute groups → emits GROUP_MERGED (PieceView frame/corners)

GAME LOOP
  GroupResolver: picture complete → emits PICTURE_COMPLETED {pictureId, cells}
  PictureClearer ← PICTURE_COMPLETED : clear anim + VFX → RefillController.plan (collapse + draw from stacks)
  → emits BOARD_REFILLED → GroupResolver re-check (a refill can complete another picture)
  ObjectiveController ← PICTURE_COMPLETED : done++ → emits PROGRESS_CHANGED

WIN
  ObjectiveController : done == params.pictureCount → emits GAME_WON
  GameplayController ← GAME_WON → GameView.onWin → EndCardView (logo + Play → CTAService)

LOSE
  none (by design)
```

## 7. Signal contracts

| Signal | Payload | Fired by | Consumed by | Trigger |
|---|---|---|---|---|
| `PICTURES_READY` | `{ pictures: PictureDef[] }` | PictureLibrary | GameplayController | all images sliced |
| `BOARD_READY` | `{ cols, rows }` | BoardLayout | TutorialController, GameplayController | board spawned |
| `PIECE_PICKED` | `{ pieceIds: number[] }` | PieceDragController | TutorialController, AudioHooks, PieceView | drag start past threshold |
| `PIECES_SWAPPED` | `{ moved: CellMove[] }` | PieceDragController (via BoardModel) | GroupResolver, AudioHooks | valid drop |
| `DROP_REJECTED` | `{ pieceIds }` | PieceDragController | AudioHooks, PieceView | drop out of board / no-op → snap back |
| `GROUP_MERGED` | `{ groupId, cells }` | GroupResolver | PieceView, VFXController, AudioHooks | new correct adjacency |
| `PICTURE_COMPLETED` | `{ pictureId, cells }` | GroupResolver | PictureClearer, ObjectiveController, VFXController, AudioHooks | all pieces joined |
| `BOARD_REFILLED` | `{ mutation: BoardMutation }` | RefillController | GroupResolver, PieceView | collapse + drop done |
| `PROGRESS_CHANGED` | `{ done, target }` | ObjectiveController | (optional HUD) | picture cleared |
| `GAME_WON` | `{}` | ObjectiveController | GameplayController, AudioHooks, VFXController | target reached |

No module imports another; all cross-module traffic is above.

## 8. Class summary, config schema, class reference

### Class summary

| Class | Module | Key properties | Key methods |
|---|---|---|---|
| `GameplayController` | bootstrap | flow state | `start()`, `onWon()` |
| `PictureLibrary` | picture-source | `pictures: PictureDef[]` | `load(params)` |
| `PieceSlicer` | picture-source | — | `slice(tex, shape): SpriteFrame[]` |
| `BoardModel` (pure TS) | board | `cells[row][col]`, `stacks[col]: PieceRef[]` | `swap()`, `pieceAt()`, `cellsOf(pictureId)` |
| `BoardGenerator` (pure TS) | board | `rng: DeterministicRandom` | `generate(cols, pictures, seed): BoardState` |
| `BoardLayout` | board | `cardSize`, `gap` | `fit(cols)`, `cellToLocal()` |
| `PieceView` | piece | sprite, frame/corner sprites | `bind(piece)`, `setFrame(neighbourMask)`, `playClear()` |
| `PieceDragController` | piece | `moveThreshold`, `dragParent` | `onTouchStart/Move/End`, `findNearestCell()` |
| `StackView` | piece | face-down card nodes | `bind(queue)`, `dealTop(): Node` |
| `GroupResolver` (pure TS) | grouping | `groups: Map` | `resolve(board)`, `canSwap(group, target)`, `isComplete(pictureId)` |
| `PictureClearer` | cascade | — | `clear(cells)` |
| `RefillController` | cascade | planner (pure TS) | `plan(board): BoardMutation`, `play(mutation)` |
| `ObjectiveController` | objective | `done`, `target` | `onPictureCompleted()` |
| `TutorialController` | tutorial | `FingerTutorial` ref | `pointAtSolvingMove()` |
| `VFXController` | vfx | pooled FX prefabs | `burstAt()`, `celebrate()` |
| `AudioHooks` | audio | — | signal → `AudioService.play` |

### Config schema (PlayableConfig params; no level JSON)

| Param | Type | Class | Billed? |
|---|---|---|---|
| `background.image`, `background.color` | image / colour | BASIC | $0 dashboard |
| `card.tintColor` | colour | BASIC | $0 dashboard |
| `pictures[]` | image list (full pictures) + `shape: '2x2' \| '1x2'` per image | COMPLEX | yes, `picture-source` |
| `pictureCount` | int (win target, ref 4) | BASIC | $0 dashboard |
| `columns` | int, max 6 | COMPLEX | yes, `board` (generator + layout) |
| `seed` | int (optional, reproducible board) | BASIC | $0 dashboard |

### Class reference (bespoke modules)

- **`PieceSlicer`** (pure helper using `cc.SpriteFrame`): for each picture, builds 4 frames (2×2) or 2 frames (1×2) by setting `rect` on the shared texture, so there is no new texture memory. Input comes from the template's base64 image param path, so `replaceParameterSprite` handles load and build-removal.
- **`BoardGenerator`** (pure TS): picks pictures so that every picture's pieces total `cols × rows` on the board plus `stack depth × cols` in the queues; lays pieces out so **no picture starts complete** and **at least one picture is 1 move from complete** (the tutorial target). The stacks are ordered so every refill keeps a completable picture available. Rows = cols (square board, ref 6×6).
- **`BoardModel`** (pure TS): cell grid + per-column queues; `swap(source cells, target cells)` handles single ↔ single, group ↔ cells of equal footprint, and rejects a drop that would split a group off the board.
- **`GroupResolver`** (pure TS): two pieces of one picture are joined when their board offset equals their in-picture offset; joined sets form groups (union-find). A group drags as one footprint. Returns each piece's neighbour mask (top/right/bottom/left) so `PieceView` picks the frame/corner sprite (the ref uses 10 rounded variants).
- **`PieceDragController`** (`cc.Component`): uses InputService ●, a move threshold before a drag starts, and single-touch lock (the reference blocks multi-touch). On release, the nearest cell within `swapDistance` gets a glide snap (pattern: `HeroDragController.changeSnap`); otherwise the piece glides back.
- **`RefillController`**: the planner is adapted from `RefillPlanner.plan` (bottom-up write cursor), but new cards come from that **column's stack queue**, not RNG. Playback follows `BoardView.applyMutation` (survivors slide, new cards flip from the face-down stack and drop).

## 9. Estimate — line items & totals

> `Reuse`: ●=inherited (near-zero) · ◐=framework-assisted · ○=custom. **Base = O = L; P = worst-case.** Inherited/mechanical rows are near-flat; bespoke gameplay carries the `○` risk premium (×1.5), `◐` ×1.25 (`estimation-rules.md` § 1). `Type`: code = gameplay module (maps to a §5 class) · prod = cross-cutting production package.

| # | Module / Package (classes) | Type | Reuse | Scope billed | O | **L** | P |
|---|---|---|---|---|---:|---:|---:|
| | **— Gameplay code modules —** | | | | | | |
| 1 | bootstrap (`GameplayController`) | code | ◐ | Template supplies folder/build/flow/CTA (anchor 0–0.5 h). Delta: 3 flow states + module init order + param read, ~80 lines typing-bound | 1 | **1** | 1.5 |
| 2 | loading (template `LoadingView`) | code | ● | Logo + FillBar wiring; await params + slicing before hiding | 0.5 | **0.5** | 0.5 |
| 3 | picture-source (`PictureLibrary`, `PieceSlicer`) | code | ○ | Runtime rect-slicing 2×2/1×2 from base64 image params, ~150 lines → 0.75 h authoring + 0.75 h allowance (part debug-bound: texture/rect/UV quirks in release build) | 1.5 | **1.5** | 2.5 |
| 4 | board — generator (`BoardGenerator`) | code | ○ | Seeded (DeterministicRandom ●) generator: N cols ≤6, picture mix, stack queues, "none complete / one 1-move solve" constraints, ~200 lines → 1 h + 1.5 h allowance (debug-bound: solvability edge cases across column counts) | 2.5 | **2.5** | 3.75 |
| 5 | board — model + layout (`BoardModel`, `BoardLayout`) | code | ○ | Cell grid, queues, swap footprints, fit-to-width for 1–6 cols, ~220 lines → 1 h + 1 h allowance (part debug-bound) | 2 | **2** | 3 |
| 6 | piece — view (`PieceView`, `StackView`) | code | ○ | Sprite bind, tint param, neighbour-mask frame/corner selection (10 variants), face-down stack render, ~180 lines → 1 h + 1 h allowance (verified in editor) | 2 | **2** | 3 |
| 7 | piece — drag (`PieceDragController`) | code | ○ | InputService ● + snap glide pattern ◐; bespoke: threshold, single-touch lock, group drag, nearest-cell resolve, snap-back, ~200 lines → 1 h + 2.5 h allowance (tuned-by-eye feel) | 3.5 | **3.5** | 5.25 |
| 8 | grouping (`GroupResolver`) | code | ○ | Union-find joins, group-vs-cells swap rules, completion detect, re-check after refill, ~200 lines → 1 h + 2 h allowance (debug-bound: group swaps that partly overlap) | 3 | **3** | 4.5 |
| 9 | cascade (`PictureClearer`, `RefillController`) | code | ◐ | `RefillPlanner`/`applyMutation` pattern covers collapse + playback. Delta: draw from per-column stack queue, flip-and-drop from stack, chain re-check, ~150 lines → 0.75 h + 1.25 h allowance | 2 | **2** | 2.5 |
| 10 | objective (`ObjectiveController`) | code | ● | Counter + win emit, ~30 lines | 0.5 | **0.5** | 0.5 |
| 11 | layout & UI (top stacks, bottom logo, safe area) | code | ◐ | Template adapt/Widget ●; delta: top/middle/bottom split scaling with column count, ~60 lines + editor layout | 1.5 | **1.5** | 2 |
| 12 | tutorial (`TutorialController`) | code | ◐ | incl. FingerTutorial DRAG wiring ≈ 1 h (anchor) + 0.5 h bespoke: ask BoardModel for the generated 1-move solve to set A/B | 1.5 | **1.5** | 2 |
| 13 | end-cards (template `EndCardView` subclass) | code | ● | Logo + Play button → CTA | 0.5 | **0.5** | 0.5 |
| 14 | audio (`AudioHooks`) | code | ● | ~6 hooks (BGM, pick, swap, merge, clear, win) × 5 min | 0.5 | **0.5** | 0.5 |
| 15 | vfx (`VFXController`) | code | ● | Pooled spawn of 3 FX on signals (ObjectPool ●) | 0.5 | **0.5** | 0.5 |
| | **Subtotal — code** | | | | 23 | **23** | 32.5 |
| | **— Production packages —** | | | | | | |
| 16 | Art integration | prod | ◐ | Client-final 2D art: card/slot/stack prefabs, frame+corner sprites (10), logo, background, buttons, font; import + atlas | 2 | **2** | 2.5 |
| 17 | FPS optimization | prod | ○ | 2D, ≤108 card nodes + FX: batching (shared atlas, avoid break by tint), low-end profile during cascade + drag | 1 | **1** | 1.5 |
| 18 | Build size optimization | prod | ○ | 4.8 MB cap with **photographic** picture params (ref already 4.4 MB): JPEG quality pass on pictures, mp3 bitrate, engine module pruning, single-file verify | 2 | **2** | 3 |
| 19 | VFX assets | prod | ○ | 3 effects × 1 h: merge sparkle, picture-clear star burst, win celebration | 3 | **3** | 6 |
| 20 | Polish / juice | prod | ◐ | Motion presets ◐: pick-up lift, swap glide, merge pulse, clear pop, drop bounce, stack flip; ≥3 variants for swap/drop SFX (fires >3×/session, V1) | 3 | **3** | 4.5 |
| 21 | QA | prod | ◐ | Column counts 1–6 × picture-count × image sets × devices; solvability spot-runs | 2.5 | **2.5** | 3 |
| 22 | Iteration buffer | prod | — | 0.1 × implementation (36.5 h base / 53 h worst) | 3.5 | **3.5** | 5.5 |
| | **Subtotal — production** | | | | 17 | **17** | 26 |
| | **TOTAL — hours** | | | | 40 | **40** | 58.5 |
| | **TOTAL — dev-days** | | | | 5 | **5** | 7.25 |
| | **TOTAL — weeks** | | | | 1 | **1** | 1.5 |

**Excluded defaults ($0):** parameter dashboard (background, tint, picture count, seed), all-network SDK/CTA adapters + build/export pipeline, template loading/end-card/CTA framework.

**Not double-counted:** VFX code (#15, spawning), VFX content (#19) and juice (#20) are three separate rows. Gameplay rows #3–#9 contain no juice.

**Reconcile:** code 23 + production 17 = **40 h** = the WP sizing total (WP0 3 · WP4 17 · WP3 1.5 · WP5 0.5 · WP7 0.5 · VFX glue 0.5 · art 2 · FPS 1 · size 2 · WP9 3 · WP8.1 3 · WP8.3 2.5 · WP8.4 3.5). Nothing is missing or double-counted.

**Reuse-tag reconcile:** every ● row is ≤0.5 h. Each ◐ row names its delta (rows 1, 9, 11, 12, 16, 20, 21). Each ○ row is absent from the sweep (§4 "Stays custom"). The FingerTutorial anchor (1 h) is shown separately in row 12. Per-object rows state their counts: VFX 3 × 1 h; frames 10.

**Feedback Matrix note (game-feedback-mandate):** haptic cells are waived (`feedback-waiver: no haptic mechanism in kit; H5 target`). Swap/drop fires >3× per session, so it ships ≥3 SFX variants (in #20). Clear and win are covered by VFX + SFX + visual.

## 9b. Client summary — module overview

Testing and client-iteration effort is **included**: it is distributed into the modules that carry it rather than listed as its own line.

| Module | Hours |
|---|---:|
| Project setup, loading screen & game flow | 2 h |
| Core puzzle mechanic: drag & swap, piece merging, card visuals | 14.5 h |
| Board generation, picture slicing & card refill | 8 h |
| Layout, tutorial & end card | 3.5 h |
| Art & audio integration | 2.5 h |
| Effects & polish | 6.5 h |
| Performance & build size | 3 h |
| **Total** | **40 h** |

> **40 h ≈ 5 dev-days ≈ 1 week** (single developer, ideal engineering time).
> Worst case if the §11 risks land badly: ~58.5 h (≈ 1.5 weeks).

## 10. Delivery summary

**10a. Timeline:** Base ~5 dev-days ≈ 1 week · Worst ~7.25 dev-days ≈ 1.5 weeks. Ideal engineering time; excludes client asset delivery and review cycles. A second dev could run generator/model (pure TS) in parallel with views, but this estimate assumes one dev.

**10b. Included at no charge:** parameter dashboard (background image/colour, card tint, picture count) · all-network SDK/CTA adapters + build export · template loading view, end card and CTA framework.

**10c. Headline:**
> **Base plan: ~40 h ≈ 5 dev-days ≈ 1 week.** Worst-case ~58.5 h ≈ 1.5 weeks.
> Kept down by the template (flow, loading, end card, tutorial, CTA, pooling, audio all wire-only), the 3DOTS refill pattern, and client-final art. Pushed up by the bespoke drag + group-swap mechanic, runtime picture slicing, the solvable generator behind the configurable column count, and the 4.8 MB cap with photo content.

## 11. Cost concentration & risk hotspots

1. **Drag + group swap (#7 + #8, 6.5 h → 9.75 h).** Group-onto-cells swaps where the footprints partly overlap a target group are the main debug risk, and drag feel is tuned by eye.
2. **Board generator (#4, 2.5 h → 3.75 h).** Guaranteeing "none complete, one 1-move solve, refill stays solvable" across 1–6 columns and mixed 2×2/1×2 pictures.
3. **Build size (#18, 2 h → 3 h).** Pictures are photos supplied as params; a large image collection can blow the 4.8 MB cap. Agree a max picture count / resolution with the client.
4. **Runtime slicing (#3, 1.5 h → 2.5 h).** Rect frames on base64-loaded textures must survive the release build (no spread over iterators; verify in the built single-file HTML, not only preview).

## 12. Gut-check

A 2D swap-puzzle playable with one bespoke mechanic, on the studio template with final art, typically lands at 4–6 dev-days. 40 h sits in that range. Re-slicing the WP totals into modules did not move the number.

## 13. Next step

Run `/t1k:cook` off §§5–8. Implementation order follows the §6 STARTUP trace:
`picture-source` → `board` (model + generator, with Vitest) → `piece` view + drag → `grouping` → `cascade` → `objective` → bootstrap flow + loading/end card → tutorial → audio/vfx → polish → size/FPS pass → QA.
On scaffolding, copy §§5–8 into `docs/system-design.md` (`state: to-be`), then generate `docs/design-manifest.json`.
