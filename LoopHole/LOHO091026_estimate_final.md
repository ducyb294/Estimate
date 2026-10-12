# LoopHole Playables: Estimate chốt (P1 + P2)

**Ngày:** 2026-10-12 · **Engine:** Cocos Creator 3.8.7 (3D, portrait) · **Nền:** CocosPlayableAdsTemplate
**Hai bản game:** `LOHO091026_P1_Con1` (ưu tiên, giao 2026-10-16) và `LOHO091026_P2_Con1` (giao sau, tính theo phần chênh lệch so với P1)
**Cơ sở:** 1 dev · giờ lý tưởng · 8h = 1 dev-day · 5 dev-day = 1 tuần
**Trạng thái:** đã chốt 2026-10-12, mức cắt giảm được xác nhận. **90 h dev + 6 h 3D support** (P1 76.5 + 4.0, P2 13.5 + 2.0)
**Nguồn kiến trúc & module:** `LOHO091026_estimate.md` §§5-8 (không đổi). File này chỉ chốt lại con số.

## 1. Con số chốt

| | P1 | P2 (delta) | Cả hai |
|---|---:|---:|---:|
| **Dev** | **76.5 h** | **13.5 h** | **90.0 h** |
| **3D support** | **4.0 h** | **2.0 h** | **6.0 h** |
| **Tổng** | **80.5 h ≈ 10.0 d** | **15.5 h ≈ 1.9 d** | **96.0 h = 12.0 d = 2.4 tuần** |

- **Dev 90h** = code module + shader/polish/VFX + FPS/size + QA + buffer.
- **3D support 6h** = nhập mesh/FBX, dựng prefab và scene, ánh sáng mặc định (P1 4h); thay bowl kính, đĩa có socket, vòng hole dày cho góc nghiêng (P2 2h). Đây là dòng "Art production" của bản 138h (5h + 2h), rút còn 6h.
- P2 chỉ đúng khi đã có code P1 (xem §5).

## 2. Bảng module cho client

Test và vòng chỉnh sửa đã được chia vào từng module, không tách dòng riêng.

| Module | P1 | P2 (delta) | Cả hai |
|---|---:|---:|---:|
| Core framework, level data, flow, camera, tutorial, end card, audio | 8.0 h | 3.0 h | 11.0 h |
| Board, hole, tường, tap và bay vào đĩa | 11.5 h | 0.5 h | 12.0 h |
| Đĩa quay, luật capture, win/lose | 7.5 h | 0.0 h | 7.5 h |
| Mô phỏng và render 15,000 bóng | 19.0 h | 1.0 h | 20.0 h |
| Shader toon, look-dev, polish | 9.0 h | 4.0 h | 13.0 h |
| Intro, VFX, game feel, tham số client | 9.0 h | 4.0 h | 13.0 h |
| Hiệu năng và dung lượng build | 12.5 h | 1.0 h | 13.5 h |
| **Dev** | **76.5 h** | **13.5 h** | **90.0 h** |
| 3D support (import, prefab, scene) | 4.0 h | 2.0 h | 6.0 h |
| **Tổng** | **80.5 h** | **15.5 h** | **96.0 h** |

## 3. Chi tiết theo module (nội bộ)

Cột Module khớp §5 của `LOHO091026_estimate.md`. L = base.

### P1: 76.5 h dev

| # | Module / gói | Gốc | **Chốt** | Ghi chú cắt giảm |
|---|---|---:|---:|---|
| 1 | bootstrap + `GameplayController` | 1.5 | **1.0** | |
| 2 | `level-data` (+ packer JSON → binary) | 2.5 | **2.0** | Dùng cùng schema 3 level mẫu |
| 3 | `board` (BoardModel, BoardOpenSolver, BoardView) | 2.5 | **2.0** | BFS dùng `core-grid-flood` |
| 4 | Walls (WallBake offline) | 3.0 | **2.0** | Bevel profile đơn giản |
| 5 | `hole` FSM, tap, lid morph, wobble/refuse | 5.0 | **4.0** | Spike morph target làm trước |
| 6 | `hole` send flight | 3.0 | **2.0** | Dùng `ballisticLaunchAngles` |
| 7 | `disk` (5 slot, nhãn %) | 4.0 | **3.5** | |
| 8 | `ball-sim` core (store, grid, solver) | 7.0 | **6.0** | |
| 9 | `ball-sim` sleep/wake | 3.5 | **3.0** | |
| 10 | `ball-sim` capture + animator | 6.5 | **5.0** | Dive/plunge rút gọn |
| 11 | `BallRenderer` (instanced, màu per-instance) | 3.0 | **2.5** | |
| 12 | `objective` (win/lose, countdown 5.5s) | 2.5 | **2.0** | |
| 13 | `bowl` glow | 1.5 | **1.0** | |
| 14 | `camera-rig` | 1.5 | **1.0** | |
| 15 | `tutorial` | 1.5 | **1.0** | |
| 16 | end card + store redirect | 1.5 | **1.0** | Gồm route redirect thẳng, không end card |
| 17 | `audio` hooks | 1.0 | **1.0** | |
| 18 | `vfx` controller | 1.0 | **0.5** | |
| 19 | `params` (bg, grayscale, video/GIF, text/font) | 2.0 | **2.0** | Giữ nguyên, client yêu cầu rõ |
| 20 | Toon `.effect` (4 effect) | 5.0 | **4.0** | |
| 21 | Env/shading polish | 6.0 | **4.0** | |
| 22 | VFX assets (5 effect) | 5.0 | **3.0** | |
| 23 | FPS optimization | 14.0 | **8.0** | Rủi ro lớn nhất, xem §4 |
| 24 | Build size optimization | 6.0 | **4.0** | |
| 25 | Feedback juice | 5.0 | **3.0** | |
| 26 | QA | 6.0 | **4.0** | |
| 27 | Buffer | 10.5 | **4.0** | |
| | **Dev P1** | **111.5** | **76.5** | |

### P2: 13.5 h dev (delta)

| # | Module / gói | Gốc | **Chốt** |
|---|---|---:|---:|
| B1 | build config + Intro state | 0.5 | **0.5** |
| B2 | `level-data` (chạy packer trên JSON P2) | 0.5 | **0.5** |
| B3 | `intro` IntroDirector (cận cảnh hole lên 96%, zoom out ~5s) | 3.0 | **2.5** |
| B4 | `ball-sim` retune (~350 bóng lớn) | 1.0 | **0.5** |
| B5 | board/bowl layout (3 cột) | 0.5 | **0.5** |
| B6 | `camera-rig` góc nghiêng | 1.0 | **0.5** |
| B7 | `params` palette 3 màu | 0.5 | **0.5** |
| B8 | `tutorial` trong intro | 0.5 | **0.5** |
| B9 | `audio` | 0.5 | **0.5** |
| B10 | Toon effect kính (bowl trong suốt) | 2.5 | **2.0** |
| B11 | Env/shading polish | 2.0 | **1.5** |
| B12 | VFX assets (retint) | 0.5 | **0.5** |
| B13 | FPS check | 1.0 | **0.5** |
| B14 | Build size gate | 1.0 | **0.5** |
| B15 | Feedback juice | 0.5 | **0.5** |
| B16 | QA | 2.0 | **1.0** |
| B17 | Buffer | 2.0 | **0.5** |
| | **Dev P2** | **19.5** | **13.5** |

### 3D support: 6.0 h

| Phần | P1 | P2 |
|---|---:|---:|
| Import mesh (ball, hole, hole đóng + proxy, cell, border, bowl, disk) | 1.0 | |
| Prefab + scene assembly qua MCP | 2.0 | |
| Ánh sáng mặc định | 1.0 | |
| Bowl kính có rim, đĩa socket, vòng hole dày, scene góc nghiêng | | 2.0 |
| **Cộng** | **4.0** | **2.0** |

## 4. Cái giá của con số 96h (đọc trước khi gửi client)

Bản bottom-up gốc là 138h (P1 116.5 + P2 21.5). Con số chốt thấp hơn khoảng 30%, và phần cắt nằm ở chỗ có rủi ro:

1. **FPS 14h → 8h, buffer 10.5h → 4h.** Đây là hai dòng ôm rủi ro 15,000 bóng. Nếu spike sleep/wake trên máy yếu không đạt, 90h không đủ. Gốc worst-case là 169.5h cho P1.
2. **Spike làm trước (nửa ngày):** solver 15,000 bóng trên máy thấp cấu hình, màu per-instance của `cc-instanced-material`, morph target của lid. Fail một trong ba thì phải báo lại scope.
3. **Hạn 2026-10-16 chỉ còn 5 ngày làm việc.** P1 80.5h ≈ 10 dev-day, một dev không kịp. Cần chia lane song song hoặc cắt scope (xem §10a của bản gốc), và nói với client sớm theo đúng yêu cầu của họ.
4. **Chưa có JSON của P2.** Giá P2 giả định không có pipe, không có blocker, 3 màu, 15 hole, khoảng 350 bóng. Nếu JSON có pipe/blocker thì phải estimate lại phần đó.
5. **P2 phụ thuộc P1 đã xong** (dùng lại code, package, spike). Làm P2 độc lập thì cộng thêm từ 100h.

## 5. Không nằm trong con số

Dashboard tham số cơ bản, adapter SDK/CTA mọi network, template bootstrap (mặc định $0) · level editor · landscape · pipe và blocker · chuyển level · localization.
