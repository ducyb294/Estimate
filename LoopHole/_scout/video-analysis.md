# LoopHole — Video-1 / Video-2 gameplay and flow decode

**Verb:** observe + decode (no hours, no ranking of effort). Frames read with native `Read` (router vision role NOT used, per instruction).
**Method:** `ffprobe` metadata; scene-cut + silence detection with `ffmpeg`; contact sheets at 2 fps (every 0.5 s) over the whole first lap of each clip, 4-10 fps around transitions; full-res crops of both boards; read of `Client_Requirement.txt`, `runner-analysis.md` §1, `level-json-analysis.md`.
**Scratch frames:** `C:\Users\PC\AppData\Local\Temp\claude\C--WORK-PROJECT-COCOS-Estimate\6fe3b733-982a-42d5-bfcb-f29c6127a64d\scratchpad\dense\` (`a_0..a_3.jpg` = V1 0-24 s at 0.5 s steps; `b_0..b_3.jpg` = V2 same; `end1.jpg` = V1 20.4-21.6 s; `v2_zoomout.jpg`).
**Timestamps** are from my own extraction, anchored by ffmpeg scene-cut detection (V1 loop cut at 21.32 s). Do not use the older `frames/v1/f_NNNN` numbering for timing — it does not map to 0.5 s steps (f_0043 already shows lap 2, which starts at 21.32 s).

---

## 0. Headline findings

1. **Both videos are a looped ~21-22 s gameplay clip, not a full level.** Each is a screen recording of the game (no editing overlays), hard-cut back to a fresh board, and the second lap is truncated by the 37.2 s file length. **No win, no lose, no end card, no CTA, no intro video/GIF, no top banner, no text other than `NN%` labels.**
2. **Video-1 = Level 23** (2 colours, 24 holes, ~15,000 tiny balls). High confidence.
3. **Video-2 matches none of L21/L23/L54** (3 colours, 15 holes, a few hundred big glossy balls, a ~5 s camera close-up intro). Treat as a 4th, not-yet-supplied level.
4. Everything visible in V1 is mechanics the Unity runner already has, plus a **tutorial hand cursor**. The things the client brief asks for (banner, end card, entrance video/GIF, store redirect, grayscale, text params, bg modes) are **not in either video**, so they come from the template default feature set and cannot be derived from the clips.

---

## 1. Video-1 (PRIORITY)

### 1.1 Container facts

| Item | Value | Source |
|---|---|---|
| Duration | 37.204 s | `ffprobe` |
| Resolution / orientation | 1080 x 1920, portrait 9:16 | `ffprobe` |
| Frame rate | 59.94 fps (60000/1001) | `ffprobe` |
| Audio | AAC, integrated -14.4 LUFS, true peak -3.9 dBFS | `ffmpeg ebur128` |
| Loop point | hard cut at **21.32 s** (only scene cut > 0.12 in the file); the audio silence pattern repeats at the same period (silence 0.07-0.68 s and 21.38-21.99 s) | `ffmpeg scene` + `silencedetect` |
| Actual content | lap 1 = 0-21.32 s, lap 2 = 21.32-37.2 s (15.9 s of the same take) | derived |

So the real creative is a **21.3 s clip**; the file is that clip played 1.75 times.

### 1.2 Scene timeline (lap 1, t in seconds)

| t | What is on screen |
|---|---|
| 0.0 | Fresh state, static. Bowl already full of balls (no pipe pour, no entrance animation, no intro video/GIF, no logo). Board full: 6 columns x 4 rows, all 24 holes present. A hand-cursor sliver is visible at the right screen edge. |
| ~0.4 | Hand slides in over board cell (row 0, col 4), taps the green hole. Hole arcs off the board onto the disk (lands at ~1.0 s on the right side of the disk, label `0%`). |
| 1.0-3.0 | First hole fills `0% -> 6% (1.5) -> 45% (2.0) -> 85% (2.5)`, then shrinks/hops and disappears at ~3.0 s. Disk rotates counter-clockwise (hole travels 3 o'clock -> 12 o'clock -> 10 o'clock). |
| 3.5-4.5 | Disk empty for ~1 s (audio silent). Hand returns, taps (row 0, col 2) green hole; it lands at the near side (bottom) of the disk. |
| 4.7-7.5 | Second tap: a blue hole lands (~5.0 s). Two holes fill in parallel: green `24-28%` at 5.0-5.5, `60%/48%` at 6.0, `96%/88%` at 6.5, both ~100% and closing at 7.0. Disk empty again at 7.5. |
| 8.0-15.0 | Rapid tapping, hand hovers at lower right of board. 3 -> 4 holes on the disk at once (9.0-12.5), then a 5th lands at 15.0. Under the disk the blue-ball mass visibly erodes and green ball regions become dominant around the disk. |
| **15.0-17.5** | **All 5 slots occupied** (4 blue + 1 green at 15.0-15.5; percentages `1/52/0/45/25` -> `94/14/79/38/46` -> `84/70/66/48`). Hand parks at the right edge. This is the only moment the disk is full. No fail countdown / red flash / warning is visible (balls are still collectable). |
| 17.5-19.5 | Holes complete one by one (`85/90/85/100` -> `100/90/hole` -> `92` -> single blue -> empty at 19.5). |
| 19.5-21.3 | Hand returns; greens are tapped; 2 then 4 **green** holes sit on the disk at `0-8%` (21.0). Board still holds roughly 8-9 holes (about 15 of 24 sent). Clip is cut here, not at a win. |
| **21.32** | **Hard cut, no transition**, to the fresh board and full bowl. Lap 2 repeats. |

### 1.3 What the player does and what happens

| Aspect | Observed |
|---|---|
| Input | Tap an **open (round)** hole on the board. Shown by a hand cursor; no touch ripple, no tap sound distinguishable from the capture SFX. |
| Board | **6 columns x 4 rows = 24 holes**, in a **checker pattern of 2 colours (12 cyan-blue, 12 dark green)**. Top row (adjacent to the bowl) is open/round; rows 1-3 are **closed flat square lids** in the hole's colour. Purple wall slabs on the left and right and a purple band below; board is flat, front-facing, no perspective. |
| Opening rule | A lid becomes a round open hole once a 4-way path of empty cells to the top row exists (seen: after (r0,c4) leaves, (r1,c4) turns round at ~1.5 s). Lids flash pale as they open (seen at ~8.5 and ~20.0 s). Matches the runner's BFS-to-exit-row rule. |
| Disk | Flat dark-indigo circle, centred ~35% down the screen, ~47% of screen width. **5 slots** (5 simultaneous holes seen at 15.0-17.5 s). No recessed socket circles are drawn on it (unlike V2). Rotates counter-clockwise, measured roughly 60-65 deg/s (consistent with runner `DiskRotationSpeed = 65`). Hole lands at the **near/bottom side** of the disk, then rotates away. |
| Capture | A landed hole shows `0%` and pulls balls of **its own colour only**; label counts up to `100%`, holds a moment, then the hole hops/shrinks and vanishes, freeing the slot. Observed 0 -> 100% in about 2.5-3 s for a hole next to its colour; a hole with its colour far away stays near `0-8%` (21.0 s). Balls near the active hole stream in and cut a visible notch in the ball mass; the rest settles toward the centre. |
| Colours | **2**: cyan-blue and dark green. Balls carry slight per-ball shade variation. Mass is roughly 50/50 (matches L23 7,434 / 7,566). |
| Ball count | Tiny balls, ~7 px diameter on a 540 px frame -> roughly 80 across, 100 down visible, and the bowl continues off-screen. **Order of magnitude 10^4** (consistent with 15,000). |
| Pipes | None visible. No pipe pour, no pipe exit. |
| Blockers / specials | None: no mystery, freeze, bomb, lock, spawner, panel, elevator. |
| Bowl look | Rim is **not visible**: ball sea fills the top ~70% of the screen, disk in the middle. Camera is high and near-top-down (disk looks circular, not an ellipse), static (no shake, no zoom). |
| Background | Not visible as a background: the balls cover the screen. Visible "background" = indigo disk, purple board frame/walls (`#5b3fd0`-ish), dark navy tile floor. |

### 1.4 HUD / UI / text

| Element | Observed |
|---|---|
| Top banner | **None.** Top of screen is balls only. |
| Counters / progress bars / moves / timer | **None.** |
| `%` labels | White bold rounded sans, dark outline, integer `NN%`, floating just above each disk hole, ~28-32 px on the 1080 frame; text shrinks when 5 holes are shown. Present in the runner (progress label shown at `ProgressLabelShownLocalY 1.2`). |
| Tutorial hand | White cartoon glove, index finger pointing, soft shadow. Enters from the **right edge**, goes to the hole to tap, taps (small press), exits right. Visible at ~0.5, 3.5-4.5, 6.5-7.5, 8-14, 19.5-21 s; parked off-edge while the disk is full (15-17.5 s). |
| End card / CTA / logo / store button | **None in the clip.** |

### 1.5 VFX and audio

| Seen | Detail |
|---|---|
| Hole send | Arc jump board -> disk, hole grows ~1.5x on landing (board ~55 px vs disk ~80 px on 540 frame); faint streak/trail. |
| Suction | Small white spark lines around an active hole; balls spiral in; notch in the ball mass. |
| Slot completion | Label to `100%`, brief hold, hop + shrink + vanish. |
| Lid open | Closed square lid flashes pale and morphs to a round hole. |
| Not seen | Confetti, camera shake, glow pulse, red unsendable flash, full-disk warning flash, win/lose screen, screen transition. |
| Audio | Event-driven: bursts at about -14 to -17 dB RMS separated by **digital silence** (e.g. 3.0-4.0 s, 6.5-7.5 s, 13.0 s, 18.5-19.5 s), which line up with moments the disk is empty. **No continuous music bed in the recording.** SFX character (suction/pop loop vs per-ball tick) not determinable from stills; it needs listening. |

### 1.6 Which known level?

**Level_23** (`Level_23.json`: 2 colours, 15,000 balls, 24 holes, `coloredHoleCaptureLimit 200`, `arenaRadius 13`, board 9 rows x 8 columns, no pipes).

| Evidence | Video | L23 JSON | Match |
|---|---|---|---|
| Colour count | 2 | 2 (types 2 and 14) | yes |
| Hole count | 24 (6 x 4) | `holeCount 24`, holes 2:12 + 14:12 | yes (12 / 12 checker) |
| Ball mix | ~50/50 | 7,434 / 7,566 | yes |
| Ball scale | tiny, bowl rim off-screen | `ballRadius 0.091`, `arenaRadius 13` | yes |
| Pipes | none | `pipes: []` | yes |
| Board shell | 6 hole columns + wall slab each side + wall band below | 8 columns x 9 rows, 48 wall cells | consistent (6 + 2 wall columns; 4 hole rows + wall rows) |
| Patchy colour regions | large swirly blue/green patches | `sourceImageId` set (ball layout derived from an image) | consistent |
| vs L21 | L21 has 14 colours, 56 holes, 6 x 16 board | n/a | rejected |
| vs L54 | L54 has 4 colours, 4 pipes, 25 holes, 350 balls | n/a | rejected |

**Not verified:** the displayed colours (cyan-blue and dark green) against the JSON ballTypes 2 (Green) and 14 (DarkTeal) and the L23 `paletteOverrides` (which only cover types 1 and 2); the 200-ball capture limit versus the observed fill time; the exact camera numbers.

### 1.7 Win AND lose shown? Differences from the Unity runner

| Question | Answer |
|---|---|
| Win shown? | **No.** Clip is cut with ~9 holes still on the board. |
| Lose shown? | **No.** Disk was full at 15-17.5 s but resolved with no countdown, flash or fail. The runner's 5.5 s fail countdown (disk flashes red) and camera shake never appear. |
| Partial? | Yes, ~15 of 24 holes sent (about 60-65%). |
| End card layout | **Unknown** — not in either video. |
| Blockers / new mechanics vs runner | **None found.** Everything visible (BFS-gated lids, tap-to-send, 5-slot rotating disk, own-colour capture, % label, capture complete -> free slot) is in `runner-analysis.md` §1. |
| UI / flow not in the runner | Tutorial hand cursor; the looped restart (runner has only `R`-key restart); looks driven by a scripted or recorded session. |
| Look differences vs runner defaults | Camera is higher/closer than the README pose (README: ortho size 16, pitch 48, bowl visible). Here the bowl rim is outside the frame and the disk is round, so camera params need re-tuning (client already asks for XYZ pos/rot, persp/ortho, zoom). Board is flat and front-facing. Disk has no visible socket rings. Palette: cyan-blue + dark green, indigo/purple theme. |

---

## 2. Video-2 (differences only)

| Aspect | Video-1 | Video-2 |
|---|---|---|
| Container | 37.2 s, 1080x1920, 60 fps | same: 37.2 s, 1080x1920, 60 fps, AAC |
| Loop | hard cut at 21.32 s | hard cut at ~22.2 s (scene cut at 22.17 s; the restart begins with the close-up) |
| Level | L23 | **None of the 3 JSONs** |
| Colours | 2 (cyan-blue, dark green) | **3: red, yellow, green** (saturated, glossy) |
| Balls | ~15,000 tiny, no rim visible | **a few hundred (~300-350 est.)**, large glossy balls in coloured sectors (green top-left, yellow top-right, red bottom half) |
| Board | 6 x 4 = 24 holes, checker, flat | **3 columns x 5 rows = 15 holes** (5 red, 5 yellow, 5 green); one colour per column; narrow board between two wide purple slabs |
| Disk / bowl | flat indigo disk, no sockets, no rim visible | **lavender disk with 5 recessed socket circles, translucent glass-like bowl with a visible 3D rim, tilted camera (~45-50 deg)** — close to the README default pose |
| Hole look | thin ring, blue/dark-green | thick glossy ring, red/yellow/green |
| Intro | none | **0-4.9 s: close-up of one yellow hole filling `0 -> 24 -> 72 -> 96%` while the camera tracks it; at ~4.9-5.3 s an eased zoom-out reveals the whole bowl and board**, then normal play |
| Hand | right-edge approach | same cartoon hand; appears inside the close-up at 0-1 s over the balls, then taps board columns |
| Max simultaneous holes | 5 | 3 seen |
| Pipes | none | none visible (so not L54 either) |
| Ending in clip | cut mid-game | cut near the end: bowl nearly empty of yellow/green, red mass still large, last holes at `84%/100%`, ~3 holes still on the board; no win screen |
| Audio | event bursts + silence | same style, event bursts + silence, no music bed |
| Banner / CTA / end card | none | none |

L54 is the nearest scale match (350 balls, 5 holes per colour) but it has 4 colours and 4 pipes, and V2 shows R/G/Y only with a red-heavy distribution, so I treat V2 as an unsupplied level.

**What V2 adds that V1 does not:** a scripted camera close-up + pull-back intro (custom camera tween or camera-param keyframes); 3D tilted bowl with a visible rim and socketed disk (looks like the runner's other bowl prefab / theme); a narrow 3-column board.

---

## 3. Custom code vs template/default

Legend: **T** = covered by the standard playable template per the client brief; **C** = custom project code; **U** = unknown, needs the template/kit to confirm.

| Item seen (or required by brief) | In V1 | In V2 | Class | Note |
|---|---|---|---|---|
| Board of holes, closed lids, BFS-to-exit opening, tap to send | yes | yes | **C** | whole LoopHole mechanic |
| 5-slot rotating disk, own-colour capture, % label, close/free slot | yes | yes | **C** | runner §1 |
| Ball simulation for ~15k balls (tiny, one colour layer swaps) | yes | n/a | **C** (perf-risk) | V1 specific; V2 is a few hundred |
| Level data / JSON-driven board, balls, palette | L23 | unseen | **C** | JSON promised by client |
| Capture VFX (suction, trail, hop-close, lid-open flash) | yes | yes | **C** | runner has them |
| Camera intro close-up + pull-back | no | yes | **C** | V2 only |
| Tutorial hand cursor, optional on/off | yes | yes | **T** (hand asset/system) + **C** (target scripting per tap) | brief: "optional tutorial" |
| SFX | yes (event-driven) | yes | **T** (replace/add SFX) | no music bed in the clips |
| Background music change | no | no | **T** | in brief, not in clips |
| Background image / solid / gradient | not visible (covered by balls) | partly (indigo behind bowl) | **T** | in brief |
| Camera XYZ pos/rot, persp/ortho, zoom params | n/a | n/a | **T** (params) + **C** (tuning per level) | in brief |
| Colour/material change for environment | n/a | n/a | **T** + **C** | in brief |
| Top banner (text, custom font) | **absent** | **absent** | **T** | in brief, not in clips |
| General text params | `%` labels only | same | **T**/**C** | in brief |
| Entrance video / GIF | absent | absent | **T** | in brief |
| Grayscale toggle | n/a | n/a | **T** | in brief |
| Store redirect without end card, end card, CTA | absent | absent | **T** | in brief, not in clips |
| Win / lose flow, fail countdown (5.5 s), camera shake | not shown | not shown | **C** (runner has it) + **T** (end card) | not verifiable from clips |
| Gameplay params (limits, speeds, counts) | n/a | n/a | **C** | brief: "wherever possible" |

---

## 4. Limits and open questions

- **Frame sampling is 2 fps for the full read and 4-10 fps near transitions**; sub-0.5 s events (single-ball captures, exact tap instants) are not resolved.
- **Counts are estimates:** "about 15 of 24 holes sent", V2 "~300-350 balls", V1 "~10^4 balls" come from visual counting, not data.
- **Audio character is unknown** (no listening). Only loudness/silence structure was measured.
- **The clips never show win/lose/end card.** Anything about those flows must come from the template defaults and the runner (`LevelStateTick`, `WinLevel`, `LoseLevel`), not from the videos.
- **V2's level is unidentified.** Ask for the V2 JSON before assuming it is a variant of L54.
- **Camera numbers for V1 were not measured**; only "higher and closer than README default" is claimed, from the disk looking circular and the rim being off-screen.
- **Colour-to-ballType mapping for L23 (cyan-blue / dark green vs Green 2 / DarkTeal 14) is not verified.**
