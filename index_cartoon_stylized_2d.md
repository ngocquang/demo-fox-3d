# index_cartoon_stylized_2d.html — Clash Village: các kỹ thuật 2D

Một file HTML duy nhất (`index_cartoon_stylized_2d.html`, ~1382 dòng), Canvas 2D thuần: không thư viện, không build.
Vẽ lại ngôi làng của `index_cartoon_stylized_3d.html` theo góc nhìn isometric (cùng đường đất, ao, quảng trường, vòng tường và bố cục 34 công trình) bằng đúng pipeline màu của bản 3D. Mọi thứ được sinh thủ tục lúc khởi động (~0.3-1 s tuỳ máy); ngoài ra chỉ có font "Lilita One" lấy từ Google Fonts (có font hệ thống dự phòng).

**Tổng: 45 kỹ thuật 2D, chia 6 nhóm.** Số này là số dòng trong các bảng dưới (cùng cách đếm với `index_cartoon_stylized_3d.md`); tách/gộp khác đi thì ra số khác. Cột "Vị trí" là số dòng trong `index_cartoon_stylized_2d.html`; mỗi mục lớn có banner `// ==== TÊN ====` để grep.

## 1. Pipeline màu và ánh sáng (14)

| # | Kỹ thuật | Mô tả | Vị trí |
| - | -------- | ----- | ------ |
| 1 | Bảng màu sRGB dùng chung | `PAL`, `CL` chép nguyên từ bản 3D; Canvas 2D xuất sRGB trực tiếp (không tone mapping) nên mỗi hex ra đúng màu cũ | 96, 106 |
| 2 | Trộn màu trong linear light | `toLin` / `toSrgb`, `lin()` có memo; gradient, ánh sáng, tint, viền đều nhân ở linear rồi mới mã hoá sRGB, như `colorspace_fragment` | 120-125 |
| 3 | Hai nguồn sáng giữ nguyên số | Directional `0xffecc8` × 0.52 + Hemisphere trời `0xd8efff` / đất tím `0x8f9bd0` × 0.64 (cường độ nhân π triệt tiêu với Lambert 1/π); mặt trời ở `(-24, 36, 15)` | 126-133 |
| 4 | Toon 5 nấc theo pháp tuyến mặt | `bandOf` tra ramp `[104, 152, 200, 236, 255]` tại `dot(N, L)·0.5 + 0.5` (Nearest) cho từng mặt phẳng: mặt trên, mặt trái (+z) và mặt phải (+x) của một khối rơi vào 3 nấc khác nhau, nên một mặt có cùng RGB với bản 3D. Phần hemisphere cộng mượt theo `n.y` nên vùng tối ngả tím xanh chứ không xám | 129-140 |
| 5 | Gradient chân → đỉnh mỗi primitive | `Kit.gr` mặc định 0.84 → 1.06 (thân cây 0.78 → 1.04, tán 0.66 → 1.14) nhân vào màu trước khi chiếu sáng; mặt phẳng dùng `createLinearGradient` dọc trục dựng của mặt, cầu dùng 5 điểm dừng | 213, 300-312, 404-423 |
| 6 | Rim light ấm | cộng `0.14·(1 − N·V)^2.4 × (1, 0.85, 0.68)` sau khi nhân màu, giống phần emissive của shader 3D: mặt phẳng là hằng số theo pháp tuyến, cầu là radial gradient ở mép | 135-139, 425-426 |
| 7 | Specular cắt cứng | đốm trắng ấm đặt tại vector nửa `H = norm(L + V)`, bán kính ≈ 0.17 R (ngưỡng `N·H ≥ 0.987`), chỉ trên cầu có `sh` (vàng, đèn); hộp và trụ đứng không bao giờ đạt ngưỡng nên không cần | 148, 427 |
| 8 | Tint ngẫu nhiên theo từng thể hiện | mỗi kiểu cây / bụi / đá bake 4 biến thể với nhân màu ±10% như `setColorAt` | 1118-1126 |
| 9 | Bóng đổ bake vào mặt đất | hull của từng primitive trượt theo tia nắng xuống y = 0, gộp thành 1 mask rồi `multiply` với màu "chỉ còn hemisphere" (`SHADOW_RGB` ≈ rgb(180, 197, 215)) nên bóng ngả xanh lạnh, không tốn gì mỗi frame | 144, 621-631 |
| 10 | Bóng tiếp xúc | radial gradient `rgba(20,40,10, .5 → .28 → 0)` rộng 1.5 lần cạnh công trình, đúng thông số blob của bản 3D | 632-636 |
| 11 | Bóng mây mềm trôi | 6 đám mây, mỗi đám là 4 khối radial gradient gộp alpha trên một canvas nhỏ (chỗ chồng không tối gấp đôi), nhuộm `SHADOW_RGB` bằng `source-in`, vẽ `multiply` với alpha 0.6 nên mép mềm; dịch theo tia nắng từ độ cao 26 và mờ dần khi ra gần mép ±34 (khung shadow camera của 3D) thay vì bị cắt thẳng | 638-662 |
| 12 | Haze và vignette | gradient haze `0xa6dcae` ở 45% trên của canvas (Fog 72-160 gần như vô hình ở zoom mặc định) cộng lớp CSS `radial-gradient` giống hệt bản 3D; nền ngoài bản đồ là `PAL.border` nhân ánh sáng mặt phẳng | 43, 1289, 1333 |
| 13 | Bóng tự đổ trong sprite | trước khi vẽ mỗi primitive, hull của nó trượt theo tia nắng chiếu lên màn hình một đoạn bằng nửa độ dày (0.03–0.16 đơn vị) rồi phủ `source-atop` một lớp xanh tím `rgba(38,46,104,.26)` lên những gì đã vẽ trước (xấp xỉ `SHADOW_RGB`, không phải multiply như bóng trên đất): mái đổ bóng xuống tường, tán cây chồng nhau, khung cửa có rãnh tối. Chỉ trong một sprite, không đổ sang công trình bên cạnh | 283-293, 235 |
| 14 | AO chân tường | dải gradient 0.16 đơn vị ở chân mặt đứng của hộp và trụ đặc (cao ≥ 0.3; không áp cho viền trang trí mỏng), vẽ trong hệ toạ độ của mặt nên song song mép đáy | 284, 310-317, 375-381 |

## 2. Viền hoạt hình (4)

| # | Kỹ thuật | Mô tả | Vị trí |
| - | -------- | ----- | ------ |
| 15 | Viền silhouette bake sẵn | stroke hull của mỗi primitive rộng gấp đôi rồi fill đè lên nên chỉ còn nửa ngoài, như mesh `BackSide`; sprite được bake lại theo zoom (`OUTW = 1.7 / (S0·zoom)`) nên nét luôn ~1.7 px như hull 3D | 273-276, 241-247 |
| 16 | Màu viền = 30% màu primitive | `outlineCol` nhân 0.3 (linear) vào chính màu của primitive (gradient trung bình 0.95, có cả tint): nét tối cùng tông thay vì đen | 272 |
| 17 | Độ dày theo kích thước | `aw = clamp(0.35 + size·0.9, 0.4, 1)` với `size` là cạnh lớn nhất của bbox 3D: chi tiết nhỏ nét mảnh, khối lớn nét dày | 271 |
| 18 | Che đường nối giữa các mặt | mỗi mặt fill rồi stroke 1.2 px cùng màu, nên răng cưa AA giữa hai mặt kề nhau không để lộ màu viền ở dưới | 277-280, 309 |

## 3. Hình học iso và primitive (13)

| # | Kỹ thuật | Mô tả | Vị trí |
| - | -------- | ----- | ------ |
| 19 | Chiếu trực giao cùng rig với 3D | yaw 45°, pitch 50°: `proj = ((x − z)·0.707, (x + z)·0.707·sin 50° − y·cos 50°)`; ba mặt nhìn thấy có pháp tuyến +y, +z (trái màn hình) và +x (phải) | 109-114 |
| 20 | `Kit` → sprite | gom primitive (`box`, `cyl`, `cone`, `sph`, `roof`, `ring3`, `rod`, `disc`, `fb`) với cờ `p` / `sh` / `gr` / `tint` như Kit 3D; `bake()` tính bbox iso rồi vẽ vào 1 canvas, mỗi công trình 1 sprite | 211-252 |
| 21 | Mặt phẳng bằng affine | `face()` nhận 3 điểm 3D, vẽ hình bình hành, `clip` rồi `transform` về hệ toạ độ của mặt (đơn vị tile) để vẽ họa tiết | 300-319 |
| 22 | Hộp, mái dốc, đĩa | `DRAW.box` (3 mặt nhìn thấy), `DRAW.roof` (2 mái dốc + tam giác đầu hồi sơn màu tường), `DRAW.disc` (đồng hồ, khiên, miệng hầm), `fb()` dán cửa / cửa sổ lên tường +z hoặc +x | 224, 359-365, 430-439, 455-459 |
| 23 | Trụ và nón theo dải | chia 10-28 dải, pháp tuyến từng dải đi qua cùng `shade()` nên các nấc toon hiện thành dải màu như bản 3D; kim tự tháp là nón 4 cạnh xoay π/4 | 366-403 |
| 24 | Cầu bằng các chỏm elip lồng nhau | ranh giới nấc k là đường `dot(N, L) = c` chiếu lên màn hình, một elip tâm `c·|ℓ|`, bán trục `Lc·√(1 − c²)` và `√(1 − c²)`; tô cả đĩa bằng nấc tối nhất rồi từng chỏm sáng dần; dome nửa cầu cắt bằng elip đáy | 351-357, 404-429 |
| 25 | Họa tiết thủ tục theo đơn vị thế giới | ván dọc / ngang, gạch so le, ngói vảy cá, mái rạ, sọc, đốm lá; vẽ trong hệ toạ độ mặt nên mật độ không đổi giữa vật lớn nhỏ (tương đương `fixUV` + `PAT_GLSL`) | 165-210 |
| 26 | Vành đai, thanh xiên, họa tiết trên mặt tròn | `ring3` (cung trước của vòng đai), `rod` (3 nét tối / thân / sáng đặt theo pháp tuyến phía sáng), `courses` và `grooves` (hàng gạch, ván trên trụ và nón) | 320-349, 440-454 |
| 27 | Painter's algorithm | thứ tự `add` của Kit là thứ tự vẽ (dưới → trên, sau → trước), vòng đai sắp lùi → gần bằng `ringPts`, các thực thể sắp theo `x + z` mỗi frame | 163, 1329 |
| 28 | Thư viện 12 công trình | cùng kích thước, palette và thứ tự primitive với `BUILD.*` của 3D: Town Hall, 3 kiểu nhà, mỏ vàng, máy elixir, 2 kho, pháo, tháp cung, trại lính, trại quân, nhà thợ | 746-980 |
| 29 | Tường tự nối | mỗi ô 1 sprite: tháp ở góc và đầu mút, trụ vuông ở đoạn thẳng, cánh thấp nối sang ô E / S; xà cổng + cờ ở chỗ hai đầu mút đối diện nhau | 982-1039 |
| 30 | Cây, bụi, đá | 5 kiểu (`treeKit`) × 4 biến thể tint, ~340 cây rừng và 46 vật rải trong làng, cùng thuật toán `scatterNature` của 3D nhưng seed riêng nên không trùng từng cây | 1082-1159 |
| 31 | Nhân vật ghép khớp bằng pivot | thân, 2 chân, 2 tay là sprite riêng với pivot ở hông / vai; đi, vung, đứng chỉ là xoay trong mặt phẳng màn hình, lật ngang theo hướng đi | 1040-1081 |

## 4. Địa hình và nước (5)

| # | Kỹ thuật | Mô tả | Vị trí |
| - | -------- | ----- | ------ |
| 32 | Mặt đất vẽ lên canvas | 3072² px (48 px/ô, lề 12 ô): ô cỏ bàn cờ có nhiễu, ~2500 chùm cỏ và hoa, quảng trường lát đá 3×3 mỗi ô, vòng khảm vàng; lấy gần nguyên `paintGround` của 3D rồi nhân ánh sáng mặt phẳng | 529-617 |
| 33 | Blit mặt đất qua affine iso | 1 lệnh `drawImage` với `transform(R2/P, R2·SP/P, −R2/P, R2·SP/P, …)`: cả mặt đất (cỏ, đường, bóng) chỉ tốn một lần vẽ ảnh mỗi frame | 1327 |
| 34 | Đường đất bằng spline | Catmull-Rom → nét vẽ nhiều lớp + ~4400 viên sỏi; cùng spline được rasterize thành mask ô để chặn cây và tạo cổng tường | 487-501, 507-515, 557-572 |
| 35 | Nước ao | 3 nấc độ sâu cách bờ 0.8 / 1.6 ô (màu lấy từ giá trị linear của shader), 2 bộ sọc sóng gợn chạy, vòng bọt; bờ cát có viền | 546-553, 665-687 |
| 36 | Lấp lánh trên nước | 9 ngôi sao 4 cánh dựng đứng trong không gian màn hình, mỗi chu kỳ hiện ở một chỗ mới trong ao (seed theo chu kỳ), kích thước `0.13·sin⁴(πu)` | 1203-1213 |

## 5. Chuyển động và hạt (6)

| # | Kỹ thuật | Mô tả | Vị trí |
| - | -------- | ----- | ------ |
| 37 | Animation đạo cụ | cờ lay (shear + co giãn), lửa trại 3 lớp nhấp nháy, giọt elixir nhấp nhô | 739-745, 956-962 |
| 38 | Khói ống khói | 3 cụm mỗi ống khói, bán kính `sin(πu)`, màu trắng → `0xb9c9e4` khi bay lên | 1253-1260 |
| 39 | Dân làng đi trên đường | 7 nhân vật đi qua lại trên 4 spline đường, quay mặt theo hướng đi | 1261-1284 |
| 40 | Đời sống ở ao | hoa súng, lau sậy, cầu tàu (9 ván, 2 thanh dọc, 6 cọc) và thuyền chèo nhấp nhô chép từ bản 3D (thuyền đặt dọc trục z vì Kit không có yaw), 3 con vịt bơi | 1160-1202, 1214-1227 |
| 41 | Gió lay cây | shear theo `sin(1.7t + 0.8x + 1.1z)`, cùng công thức pha với shader sway của 3D | 1151-1156 |
| 42 | Bướm | 7 con bay theo quỹ đạo Lissajous trên cỏ trong làng, cánh vỗ bằng co giãn ngang `0.25 + 0.75·|cos 17t|`, chấm bóng dưới đất, khoá sắp xếp cập nhật mỗi frame | 1228-1251 |

## 6. Camera và hiệu năng (3)

| # | Kỹ thuật | Mô tả | Vị trí |
| - | -------- | ----- | ------ |
| 43 | Pan / zoom | Pointer Events: kéo để pan, bánh xe / chụm 2 ngón để zoom quanh điểm dưới con trỏ | 1285-1316 |
| 44 | Sprite cache bake lại theo zoom | mọi thứ tĩnh bake thành canvas; khi zoom dừng 200 ms và độ phân giải lệch > 25% thì `rebake()` vẽ lại tại chỗ cả ~128 sprite (~40 ms) ở `BS = DPR·S0·zoom·(1.6 → 1.15)`, tối đa 160 px/đơn vị; cây dùng chung 20 sprite; mỗi frame chỉ `drawImage` | 227-253, 1118-1126, 1300, 1353 |
| 45 | Culling theo khung nhìn | cây ngoài khung nhìn không vẽ | 1153 |

## Thứ tự mỗi frame

```
update(dt)  T += dt → dân làng (đi theo spline, cập nhật khoá sắp xếp)
zoom dừng  rebake(): vẽ lại mọi sprite ở độ phân giải của zoom mới (debounce 200 ms)
render      nền ngoài bản đồ → blit mặt đất (đã chứa bóng đổ + blob) qua affine iso
            → nước + hoa súng + lấp lánh → bóng mây mềm (multiply)
            → sort theo x + z → từng thực thể: sprite bake (công trình, tường, cây…) + phần vẽ trực tiếp (cờ, lửa, nhân vật)
            → khói → gradient haze → (CSS) vignette
```

Không có post-process: viền, bóng (cả bóng trong sprite và AO), họa tiết đều được bake vào sprite hoặc mặt đất; haze là một lớp gradient.

## Ngoài 45 kỹ thuật trên (không tính vào tổng)

- Không port từ bản 3D: kinh tế (mỏ, kho, bong bóng thu), nâng cấp, luyện quân, raid, shop, kéo thả đặt công trình, BFS, HUD và màn hình loading (chỉ có tiêu đề và dòng gợi ý), goblin, LOD cây gần / xa.
- Giới hạn còn lại: bóng trong sprite chỉ đổ lên chính công trình đó, không sang công trình bên cạnh (bóng xuống đất thì có); mặt đất là texture 48 px/ô nên ở zoom 3× mềm hơn sprite; trong 200 ms trước khi bake lại, sprite tạm co giãn theo zoom.
- Camera cố định hướng (không xoay), nên cửa và cửa sổ của nhà luôn quay về mặt +z (trái màn hình); archer đứng ở mép trước sàn để không bị mái che mất.

## Đối chiếu màu với bản 3D

Chụp cùng khung 1280×720 (`?test`, pose mặc định ở cả hai bản), lấy median từng nhóm điểm ảnh (RGB):

| Nhóm | 2D | 3D |
| ---- | -- | -- |
| Cỏ sáng (percentile 75) | 146, 211, 70 | 147, 208, 74 |
| Cỏ (median, gồm cả vùng bóng) | 134, 186, 64 | 138, 202, 61 |
| Mái đỏ | 193, 68, 45 | 193, 62, 40 |
| Đá | 140, 132, 121 | 137, 130, 118 |
| Đường đất | 226, 196, 136 | 219, 196, 145 |

Cỏ median lệch vì bóng mây rơi ở chỗ khác (vị trí mây theo seed riêng). Số đo này lấy trước khi thêm bóng trong sprite, AO và bóng mây mềm; chưa đo lại.

## Điều khiển và tham số

| Thao tác | Tác dụng |
| -------- | -------- |
| Kéo chuột / 1 ngón | Di chuyển bản đồ |
| Cuộn / chụm 2 ngón, `+` `-` | Zoom quanh điểm dưới con trỏ |

Tham số URL: `?scale=0.75` (nhân pixel ratio) `?seed=<n>` (đổi nền, cây, mây; bố cục công trình cố định) `?debug` (FPS, số sprite, thời gian bake lại) `?test` (không chạy vòng lặp; điều khiển qua `window.__app`: `frames(n)`, `pose(x, y, zoom)`, `info()`).

Lưu ý: `pose(x, y, zoom)` nhận toạ độ tâm màn hình theo đơn vị iso (1 đơn vị = 1 ô) và hệ số zoom, khác với `pose(x, z, dist, yaw, pitch)` của bản 3D.

## So với index_cartoon_stylized_3d.html

| | index_cartoon_stylized_3d.html | index_cartoon_stylized_2d.html |
| - | - | - |
| Render | WebGL, một lần `renderer.render`, MSAA gốc | Canvas 2D: blit mặt đất → sprite theo thứ tự `x + z` → haze |
| Tô màu | `MeshToonMaterial` + gradientMap, vertex color × ánh sáng trong shader | `shade()` trên CPU lúc bake, cùng công thức linear, tính theo pháp tuyến từng mặt |
| Viền | inverted hull trong clip space, dày cố định theo pixel | stroke silhouette bake vào sprite, bake lại theo zoom nên cũng ~1.7 px |
| Bóng | shadow map 4096² PCF + bóng mây + blob | hull trượt theo tia nắng bake vào mặt đất, bóng trong sprite (`source-atop`) + AO chân tường, bóng mây mềm, blob |
| Che khuất | z-buffer | painter's algorithm: thứ tự add trong Kit + sort `x + z` |
| Vật lặp | `InstancedMesh` + `setColorAt` | sprite cache 5 kiểu × 4 tint |
| Họa tiết | `PAT_GLSL` + `fwidth` | `drawPat` vẽ nét trong hệ toạ độ mặt, bake sẵn |
| Nước | `ShaderMaterial` | vẽ lại đa giác theo bờ mỗi frame (3 nấc, sọc, bọt) |
| Camera | Perspective 28°, pan / zoom / xoay | trực giao, pan / zoom, không xoay |
| Độ nét khi zoom | luôn nét | sprite bake lại khi zoom dừng (~40 ms) nên luôn nét; mặt đất 48 px/ô hơi mềm ở zoom 3× |
| Tương tác | chọn, kéo, đặt, xây, nâng cấp, luyện quân, raid | chỉ xem + pan / zoom |
| Thư viện | three.js r186 | không có |
