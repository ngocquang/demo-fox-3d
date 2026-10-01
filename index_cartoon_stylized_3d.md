# index_cartoon_stylized_3d.html — Clash Village: các kỹ thuật 3D

Một file HTML duy nhất (`index_cartoon_stylized_3d.html`, ~2444 dòng), three.js **r186** nạp bằng ES module từ jsDelivr, không build.
Làng kiểu Clash of Clans (nhà cửa, cây cối, đường đất, hồ nước, tường, phòng thủ, quân lính) được sinh thủ tục hoàn toàn lúc khởi động; ngoài three.js, chỉ font "Lilita One" lấy từ Google Fonts (có font hệ thống dự phòng).

**Tổng: 41 kỹ thuật 3D, chia 7 nhóm.** Số này là số dòng trong các bảng dưới (cùng cách đếm với `index.md`); tách/gộp khác đi thì ra số khác. Cột "Vị trí" là số dòng trong `index_cartoon_stylized_3d.html`; mỗi mục lớn có banner `// ==== TÊN ====` để grep.

## 1. Render và ánh sáng (8)

| # | Kỹ thuật | Mô tả | Vị trí |
| - | -------- | ----- | ------ |
| 1 | Canvas MSAA gốc, không post-process | `antialias: true`, pixel ratio tối đa 2 nhân `?scale`; không tone mapping, xuất sRGB để giữ màu bão hoà kiểu hoạt hình | 285 |
| 2 | Toon shading 5 nấc | `MeshToonMaterial` + `gradientMap` là `DataTexture` 5 texel `[104, 152, 200, 236, 255]` (`RedFormat`, `NearestFilter`) nhân với vertex color; đất của Hemisphere là tím lạnh nên vùng tối ngả xanh thay vì xám | 306, 377 |
| 3 | Shadow map mặt trời | 4096² (2048 trên máy cảm ứng), `PCFShadowMap` (r186 đã bỏ `PCFSoftShadowMap`), camera trực giao phủ vừa cả làng | 782 |
| 4 | Bộ đèn hai nguồn | Hemisphere (trời xanh / đất xanh lá) + Directional ấm từ trên-trái; cường độ nhân π vì đèn dùng đơn vị vật lý từ r155 | 781-812 |
| 5 | Bóng mây trôi | 6 khối "mây" vô hình (`colorWrite: false`, `depthWrite: false`) chỉ để đổ bóng vào shadow map, trôi chậm qua làng | 790 |
| 6 | Bóng tiếp xúc | Mặt phẳng `CanvasTexture` gradient tròn dưới mỗi công trình (`transparent`, `depthWrite: false`) | 807 |
| 7 | Sương mù và vignette | `THREE.Fog` hoà vào màu haze; vignette là lớp CSS `radial-gradient` phủ trên canvas | 294, 62 |
| 8 | Rim light và specular kiểu hoạt hình | `onBeforeCompile` chèn vào `<emissivemap_fragment>`: viền sáng ấm theo `pow(1 - N·V, 2.4)` và một đốm bóng cắt cứng (`smoothstep(0.5, 0.58, pow(N·H, 48))`) nhân thuộc tính `aM.y` để chỉ vàng / nước / kim loại mới bóng | 367 |

## 2. Viền hoạt hình (5)

| # | Kỹ thuật | Mô tả | Vị trí |
| - | -------- | ----- | ------ |
| 9 | Inverted hull, dày cố định theo pixel | Mesh phụ `BackSide` + vertex shader đẩy đỉnh trong clip space theo hướng normal (`clip.xy += … * clip.w`), nên nét vẽ luôn ~1.7 px dù zoom | 378 |
| 10 | Normal viền hàn theo vị trí | Gộp normal của các đỉnh trùng vị trí để khối hộp không bị hở ở góc | 411 |
| 11 | Viền cho `InstancedMesh` | Hull dùng chung `instanceMatrix`, shader có nhánh `USE_INSTANCING` và cùng công thức gió lay với mesh chính | 432 |
| 12 | Đổi màu viền theo trạng thái | 4 trạng thái (thường, đang chọn, hợp lệ, sai chỗ) bằng cách hoán đổi vật liệu dùng chung, không clone | 403, 442 |
| 13 | Viền có màu và độ dày theo từng primitive | Viền thường lấy 30% màu đỉnh (nét tối cùng tông với vật thể); thuộc tính `aW` (0.4–1, theo kích thước primitive) nhân vào độ dày nên chi tiết nhỏ có nét mảnh, khối lớn có nét dày | 381 |

## 3. Hình học thủ tục (11)

| # | Kỹ thuật | Mô tả | Vị trí |
| - | -------- | ----- | ------ |
| 14 | `Kit`: gộp primitive có vertex color | Mỗi công trình là hàng chục primitive rồi `mergeGeometries` thành 1 mesh (1 draw + 1 draw viền) | 473 |
| 15 | Bo góc và lăng trụ | `RoundedBoxGeometry` (chamfer) cho khối "mập", `ExtrudeGeometry` cho mái dốc | 223, 506 |
| 16 | Họa tiết thủ tục trong fragment shader | `PAT_GLSL`: ván dọc / ngang, gạch so le, ngói vảy cá, mái rạ, đá cuội, lá / đá (value noise), sọc, vỏ cây. Chọn theo `aM.x` từng primitive, chống răng cưa bằng `fwidth` nên nét vữa và mép ngói sắc ở mọi mức zoom, không cần texture | 313 |
| 17 | UV theo đơn vị thế giới | `fixUV` viết lại UV mỗi primitive (box: chiếu phẳng theo mặt; trụ / nón / cầu: nhân bán kính; lăng trụ: nhân độ dốc) để mật độ họa tiết không đổi giữa vật lớn nhỏ; họa tiết nhiễu dùng phép chiếu xiên liên tục nên cầu / nón không có đường nối hay điểm co ở cực | 450 |
| 18 | Gradient đỉnh bake sẵn | Mỗi primitive được nhân sáng dần từ chân (0.84) lên đỉnh (1.06); tán cây dùng dải rộng hơn (0.66–1.14). Cho chiều sâu và giả AO mà không tốn pass hay texture | 492 |
| 19 | Đá icosphere nhiễu đỉnh + AO khe nứt | `rockGeo`: `IcosahedronGeometry` (detail 2–3) đẩy đỉnh bằng nhiễu lượng giác chỉ phụ thuộc vị trí (đỉnh trùng của mesh non-indexed vẫn khít), đáy cắt phẳng; độ lõm được ghi vào thuộc tính `shade` để khe nứt tối hơn, `Kit.add` đọc rồi bỏ thuộc tính này trước khi merge | 526 |
| 20 | Thư viện 12 công trình | Town Hall (tháp đồng hồ, 4 tháp mái xanh), Gold Mine, Elixir Collector, 2 kho, Barracks, Army Camp, Cannon, Archer Tower, Builder's Hut, Wall, Cottage (3 kiểu: cottage, tudor, chuồng gỗ); nâng cấp thêm chi tiết vàng | 932 … 1327 |
| 21 | Instancing thiên nhiên + LOD cây gần / xa | Cây tròn, thông, bụi, đá: mỗi loại 1 `InstancedMesh`, màu lệch nhẹ theo từng instance (`setColorAt`). Cây tròn tách 2 bộ: bản chi tiết (~1.000–1.200 tam giác, gốc xoè, cành, 6 tán) cho cây trong làng và cây cách tâm < `HI_D`, bản rẻ (~390) cho rừng xa; thông 4 tầng nón | 1428 |
| 22 | Gió lay bằng shader | `onBeforeCompile` chèn dịch chuyển đỉnh theo `uTime` và vị trí instance, cùng công thức với hull | 1392 |
| 23 | Tường tự nối | Mọi mảnh tường gộp thành 1 mesh, dựng lại khi thay đổi. `wallPiece` dựng từng ô: tháp tròn mái ngói ở góc / đầu mút, trụ vuông có răng cưa ở đoạn thẳng, thành tường nối sang ô liền kề; cổng (xà + cờ) tự hình thành nơi đường cắt vòng tường. Ghost đặt tường và ảnh Shop dùng chung `wallPiece` nên luôn giống hàng thật | 1061 |
| 24 | Nhân vật gắn khớp bằng pivot | Thân, hai chân, hai tay (một tay cầm vũ khí, tay kia vung ngược pha) là mesh riêng; chu kỳ đi / đánh / đứng chỉ là xoay pivot | 1354, 1383 |

## 4. Địa hình và nước (4)

| # | Kỹ thuật | Mô tả | Vị trí |
| - | -------- | ----- | ------ |
| 25 | Mặt đất vẽ lên canvas | 2304² px (48 px/ô): ô cỏ bàn cờ có nhiễu, ~2500 chùm cỏ ngắn, hoa, quảng trường lát đá bo góc 3×3 mỗi ô, vòng khảm vàng; texture sRGB, anisotropy tối đa | 630 |
| 26 | Đường đất bằng spline | Catmull-Rom → nét vẽ nhiều lớp (viền cỏ, viền đất, lõi, vệt bánh xe) và ~4400 viên sỏi nhỏ có bóng tiếp xúc; cùng spline đó được rasterize thành mask ô để chặn đặt công trình | 544, 582, 603 |
| 27 | Shader nước ao | `ShaderMaterial`: độ sâu chia 3 nấc theo khoảng cách tới bờ (hàm bán kính theo góc dùng chung với JS), sọc sóng chạy bằng value noise, vòng bọt méo theo thời gian | 733 |
| 28 | Bờ ao và đời sống | `ExtrudeGeometry` từ `Shape` có lỗ tạo bờ cát; hoa súng, lau sậy, cầu tàu, thuyền, 3 con vịt bơi | 761, 1475 |

## 5. Chuyển động và hạt (6)

| # | Kỹ thuật | Mô tả | Vị trí |
| - | -------- | ----- | ------ |
| 29 | Animation đạo cụ | Cờ, bánh xe, lửa trại nhấp nháy, giọt elixir nhấp nhô, nòng pháo quay và giật, cung thủ ngắm | 1292, 1194, 1234 |
| 30 | Khói ống khói | Một `InstancedMesh` cầu 14×10, 3 cụm mỗi ống khói, kích thước theo sin, màu chuyển từ trắng sang xanh nhạt khi bay lên (`setColorAt`) | 1527 |
| 31 | Bể hạt FX | 220 instance có màu riêng: bụi khi xây, lấp lánh khi thu tài nguyên, bột khi trúng đạn, "poof" | 1712 |
| 32 | Nảy khi chọn và hiện công trình | Squash & stretch tắt dần khi chọn, `easeOutBack` khi xây hoặc nâng cấp xong | 2330 |
| 33 | Dân làng đi trên đường | 7 nhân vật đi qua lại theo spline đường, quay mặt theo hướng đi | 1540 |
| 34 | Đạn | Đạn pháo bay parabol và nổ lan (sát thương vùng), mũi tên tự dò theo mục tiêu | 2226, 2247 |

## 6. Camera và tương tác (6)

| # | Kỹ thuật | Mô tả | Vị trí |
| - | -------- | ----- | ------ |
| 35 | Camera rig | Pan bằng "nắm điểm trên mặt đất" (giao tia–mặt phẳng nên điểm dưới con trỏ đứng yên), zoom giảm chấn (bánh xe / chụm 2 ngón), xoay yaw; dùng Pointer Events nên chạy cả cảm ứng | 1575, 1592 |
| 36 | Chọn bằng raycast | Raycast vào nhóm công trình; tường tra theo ô; điểm rơi trên mặt đất là phương án dự phòng | 1973 |
| 37 | Lớp phủ lưới đặt | Lưới ô (texture lặp), mask ô bị cấm (`DataTexture` N×N, Nearest), ô footprint xanh / đỏ | 1902, 1911, 1901 |
| 38 | Kéo thả bám ô | Giữ offset từ con trỏ tới tâm công trình, làm tròn về ô, nâng lên khi kéo; đặt mới có ghost + nút ✔ / ✖ | 2002, 2010 |
| 39 | Tìm đường BFS | Lưới 40×40, 8 hướng không cắt góc; đơn vị nhảy qua tường (vòng cung theo khoảng cách tới ô tường) | 1779, 1799 |
| 40 | Nhãn DOM theo toạ độ thế giới | Bong bóng thu tài nguyên, đồng hồ xây: `Vector3.project` mỗi frame; icon bay về HUD bằng Web Animations | 1737, 1757 |

## 7. Render ra texture (1)

| # | Kỹ thuật | Mô tả | Vị trí |
| - | -------- | ----- | ------ |
| 41 | Ảnh 3D trong Shop | Mỗi công trình dựng bằng chính builder của nó, render vào `WebGLRenderTarget` 192² (MSAA 4x, `colorSpace: sRGB`, nền trong suốt), `readRenderTargetPixels` → canvas → PNG; độ dày viền được hiệu chỉnh cho kích thước target | 2092 |

## Thứ tự mỗi frame

```
update(dt)  camera rig (zoom/xoay giảm chấn) → mây → animation công trình → dựng lại tường (nếu đổi)
            → kinh tế + xây dựng + overlay → dân làng, trại quân, ao, khói → đơn vị, raid, FX, nhãn DOM
render      pass đổ bóng (mặt trời) → pass chính: mesh toon + hull viền (BackSide), fog
            → lớp trong suốt theo renderOrder (bóng tiếp xúc, lưới, footprint, hạt FX)
```

Không có EffectComposer: viền, bóng, sương mù đều nằm trong một lần `renderer.render`.

## Ngoài 41 kỹ thuật trên (không tính vào tổng)

- Kinh tế: mỏ vàng / máy elixir tích trữ theo thời gian, kho quyết định trần chứa, thu bằng bong bóng; nâng cấp tốn tài nguyên, có giàn giáo và đồng hồ, "Finish now" bằng gem: 1671-1706, 1837-1896.
- Luyện quân từ Barracks (đi bộ theo BFS về trại) và **Raid**: goblin chạy tới kho / mỏ, ăn cắp rồi tháo chạy; pháo, tháp cung và lính thả bằng cách chạm đất chặn chúng: 1819-1836, 2159-2327.
- HUD viết bằng HTML/CSS với transitions-dev: number pop-in cho bộ đếm tài nguyên, modal cho Shop, panel reveal cho thanh hành động; có `prefers-reduced-motion`.
- Khung báo lỗi, màn hình loading, hook `window.__app` (`frames`, `pose`, `info`, `perf`, `select`, `beginPlace`, `startRaid`…): 2378-2400.

## Điều khiển và tham số

| Thao tác | Tác dụng |
| -------- | -------- |
| Kéo chuột / 1 ngón | Di chuyển bản đồ |
| Cuộn / chụm 2 ngón, `+` `-` | Zoom |
| Kéo chuột phải hoặc `Shift` + kéo, `Q` `E` | Xoay góc nhìn |
| Chạm công trình | Chọn (nảy, viền vàng, thanh hành động hiện lên) |
| Kéo công trình đang chọn | Di chuyển (xanh = hợp lệ, đỏ = sai chỗ) |
| Shop → chọn thẻ → ✔ / ✖ | Đặt công trình mới (tường đặt liên tiếp) |
| Raid → chạm mặt đất | Thả Barbarian chặn goblin |
| `Esc` | Đóng Shop, huỷ đặt, bỏ chọn |

Tham số URL: `?shadow=2048` `?scale=0.75` `?seed=<n>` `?debug` (FPS, số draw call) `?test` (không chạy vòng lặp, tự điều khiển frame qua `window.__app`).

## So với index.html

| | index.html (Forest Pond) | index_cartoon_stylized_3d.html (Clash Village) |
| - | - | - |
| Phong cách | Ảnh thực: PBR, HDR, ACES | Hoạt hình: toon 5 nấc, viền, rim + specular, màu bão hoà |
| Pipeline | EffectComposer nhiều pass (GTAO, volumetric, bloom, DoF, SMAA, grade) | Một lần render, MSAA gốc, viền bằng hull |
| Bóng | Shadow map 4096² PCFSoft + tra cứu tuỳ chỉnh, đốm nắng qua tán lá | Shadow map 4096² PCF, bóng mây, bóng tiếp xúc |
| Nước | Phản chiếu phẳng, khúc xạ, Fresnel, caustics | Shader toon: nấc độ sâu, sọc sóng, vòng bọt |
| Texture | Bake PBR 2048 trên GPU | Vẽ canvas 2D (mặt đất), màu đỉnh cho công trình |
| Camera | Điện ảnh 90 s + bay tự do | Xiên cố định kiểu Clash of Clans, pan / zoom / xoay |
| Tương tác | Không (chỉ xem) | Chọn, kéo, đặt, xây, nâng cấp, luyện quân, raid |
| three.js | r169 | r186 |
