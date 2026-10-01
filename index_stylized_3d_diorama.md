# index_stylized_3d_diorama.html — Fireside Graveyard: các kỹ thuật 3D

Một file HTML duy nhất (`index_stylized_3d_diorama.html`, 1208 dòng), three.js **r186** nạp bằng ES module từ jsDelivr, không build.
Nghĩa địa ban đêm kiểu "creepy-cute" thu nhỏ (diorama): bãi lửa trại lát ~1000 viên cuội 3D, bóng ma và bộ xương chibi, cây lá đỏ, cổng đá có vòm, hàng rào sắt, hầm mộ mái ván, đèn lồng và nến. Mọi thứ sinh thủ tục lúc khởi động (~0.3-3 s tuỳ máy); ngoài three.js, chỉ font "Cormorant Garamond" lấy từ Google Fonts (có font hệ thống dự phòng).

**Bố cục lấy từ ảnh tham chiếu:** toạ độ mọi vật (lửa, ma, xương, đèn, trụ đá, đường, mảng cuội, cây đỏ, hầm mộ) được đo bằng cách chiếu ngược pixel của ảnh xuống mặt phẳng đất qua đúng camera mặc định (FOV 30°, cao 54°, cách 26 đơn vị), nên khung hình đầu tiên khớp bố cục ảnh ở tỉ lệ ~16:9 (tỉ lệ khác thì khung hình dịch đi, và màn hình dọc đổi FOV sang 42°). Độ sáng cũng khớp: trung bình 29/255 so với 27/255 của ảnh, tông ấm trung tính.

**Tổng: 55 kỹ thuật 3D, chia 9 nhóm.** Số này là số dòng trong các bảng dưới; tách/gộp khác đi thì ra số khác. Cột "Vị trí" là số dòng trong `index_stylized_3d_diorama.html`; mỗi mục lớn có banner `// ==== TÊN ====` để grep.

## 1. Render và hậu kỳ (8)

| # | Kỹ thuật | Mô tả | Vị trí |
| - | -------- | ----- | ------ |
| 1 | Bề mặt smooth kiểu "đất sét" | `smooth()` bỏ uv/normal, `mergeVertices` rồi tính lại normal nên facet mượt; `jitter()` xê dịch đỉnh theo hàm băm của vị trí (đỉnh trùng nhích như nhau, không nứt) để cuội và đá có hình không đều | 166, 176 |
| 2 | Cạnh vát bằng `RoundedBoxGeometry` | Trụ, khối tường, bệ đèn, thùng gỗ đều vát cạnh nên bắt sáng dọc mép; hình học cache theo kích thước | 179-184 |
| 3 | Chuỗi HDR có MSAA | `EffectComposer` dùng render target `HalfFloatType`, `samples: 4`; canvas tắt `antialias` vì khử răng cưa nằm ở target | 213-216 |
| 4 | Bloom ngưỡng cao | `UnrealBloomPass` ngưỡng 1.1, cường độ 0.3: chỉ lửa, than, nến, kính đèn (màu HDR > 1) mới toả sáng, không có quầng mù | 217-218 |
| 5 | Blur chỉ ở mép khung | Pass tự viết: 12 tap xoắn ốc vàng, bán kính chỉ tăng khi `abs(uv.y - 0.5) > 0.3`, tối đa 3.4 px; 60% giữa khung giữ nguyên độ nét | 226-237 |
| 6 | Grade trung tính + vignette + grain | Bão hoà 0.97, bóng đổ lệch nhẹ lạnh, vùng sáng lệch nhẹ ấm, vignette `dot(q, q)`; grain nhân 1.2% dùng interleaved-gradient-noise (không dùng `sin` nên không sọc) | 238-245 |
| 7 | Làm nét kiểu CAS | Pass sau `OutputPass` (không gian sRGB): 4 láng giềng, biên độ theo `sqrt(min(mn, 1-mx)/mx)` nên chỉ làm nét chỗ có chi tiết, không tạo quầng ở vùng tương phản cao | 248-267 |
| 8 | Tone mapping ACES, `OutputPass`, sương mù mũ | Cảnh giữ tuyến tính HDR suốt chuỗi; ACES và chuyển sRGB làm ở pass cuối; `FogExp2` nhẹ nối vòng cây ngoài rìa vào nền | 148, 155, 247 |

## 2. Ánh sáng (5)

| # | Kỹ thuật | Mô tả | Vị trí |
| - | -------- | ----- | ------ |
| 9 | Trăng và trời tối | `HemisphereLight` (trời xám lạnh, đất nâu ấm) + `DirectionalLight` ánh trăng, shadow map 4096² (2048² trên máy cảm ứng), `normalBias` chống răng cưa bóng; cường độ tính theo đơn vị vật lý từ r155 | 282-291 |
| 10 | Lửa trại đổ bóng cube | `PointLight` cam có shadow cube 1024²: bóng của cuội, lá, nhân vật ngả ra xa lửa; cường độ nhấp nháy bằng tích hai sin | 292-301, 1140 |
| 11 | Đèn lồng nhấp nháy | 5 đèn có `PointLight` thật cường độ thấp, các đèn còn lại chỉ phát sáng giả (kính `emissive`) để giữ số đèn thấp | 802-826, 1141 |
| 12 | Quầng cộng màu và vệt sáng trên đất | `Sprite` và mặt phẳng dưới đất dùng `CanvasTexture` gradient tròn, `AdditiveBlending`, `depthWrite: false` | 782-801 |
| 13 | Decal bóng tiếp xúc | Mọi vật gọi `ao(x, z, r)`; tất cả gom vào **một** `InstancedMesh` mặt phẳng, texture gradient đen, `polygonOffset` để không z-fight | 185, 1105-1118 |

## 3. Bố cục và mặt đất (6)

| # | Kỹ thuật | Mô tả | Vị trí |
| - | -------- | ----- | ------ |
| 14 | Bố cục đo từ ảnh | 7 spline `CatmullRomCurve3` cho đường đất, 5 vùng cuội (`ZONES`), danh sách vùng cấm `keepOut`; `distPath()` và `blocked()` giữ bụi cây không đè lên đường hay đạo cụ | 304-340 |
| 15 | Nền canvas 4096² | 1100 đốm gradient nhiều tông cỏ + 60 000 nét cỏ ngắn; texture sRGB, anisotropy tối đa; 2048² trên máy cảm ứng | 341-364, 414 |
| 16 | Đường đất mờ mép bằng stamp gradient | Dọc mỗi spline, cứ 0.2 đơn vị đóng một "dấu" `RadialGradient` (4 lớp bán kính/màu); chồng nhiều dấu alpha thấp cho mép mềm và lõi sáng hơn, không bị bậc như nét vẽ nhiều lớp | 365-379 |
| 17 | Hạt đất và sỏi vẽ | Hàng chục nghìn chấm nhỏ nhiều tông nâu + vài trăm viên sỏi có bóng đổ nhỏ | 380-391 |
| 18 | Lớp lót cuội dạng đa giác mượt | Tia toả 240 hướng dò biên của từng vùng, tô đa giác với `shadowBlur` để mép mềm (thay cho ô vuông gây răng cưa) | 392-406 |
| 19 | Mép mờ vào nền | Gradient tròn phủ lên canvas chuyển sang đúng màu mặt phẳng nền 300×300 bên dưới, không thấy mép | 407-422 |

## 4. Cuội 3D (2)

| # | Kỹ thuật | Mô tả | Vị trí |
| - | -------- | ----- | ------ |
| 20 | Cuội instancing | Một `IcosahedronGeometry(1, 2)` smooth + jitter, mỗi vùng là một `InstancedMesh`: lưới lệch ngẫu nhiên, bán kính và độ dẹt riêng, xoay ngẫu nhiên, màu HSL riêng (`setColorAt`); ~1000 viên đổ bóng và nhận bóng | 424-450 |
| 21 | Vùng theo hàm độ sâu | `blob()` (elip có mép lượn sóng) và `tear()` (giọt nước) trả 0..1; xác suất đặt viên và kích thước giảm dần ra mép nên mảng cuội không có đường viền cứng | 318-331 |

## 5. Lá và cây (7)

| # | Kỹ thuật | Mô tả | Vị trí |
| - | -------- | ----- | ------ |
| 22 | Thẻ lá gập và cong | `makeLeafGeo()` dựng lá từ các hàng đỉnh (trái, sống lá, phải): sống lá nhô lên tạo nếp gập chữ V, `z += 0.28·y²` làm ngọn cong; hai dạng: lá hẹp cho bụi, lá bầu dục rộng cho cây; gradient màu đỉnh tối ở gốc | 453-468 |
| 23 | Gió lay bằng shader | `onBeforeCompile` thay `begin_vertex`, pha lấy từ `instanceMatrix[3]`, biên độ tăng theo bình phương chiều dài lá và độ cao; áp dụng cả cho `MeshDepthMaterial` và `MeshDistanceMaterial` nên bóng lá lay đúng | 469-488 |
| 24 | Chunk theo ô 9×9 | Lá gom theo (ô, dạng lá, có đổ bóng hay không) thành ~50 `InstancedMesh` có bounding sphere, nên frustum culling (cả trong shadow pass) loại được phần ngoài tầm nhìn; ~30 000 lá | 494-499, 506-536 |
| 25 | Cụm bụi toả tia | `bush()` đặt lá quanh tâm, hướng lá theo `tilt` (ngoài phẳng hơn trong), trục cong hướng ra ngoài và xuống; ma trận dựng từ cơ sở trực chuẩn, có nhánh dự phòng khi trục suy biến; bảng màu xanh / xanh tối / thu / đỏ | 489-505 |
| 26 | Luống dọc đường, vườn thu, vòng ngoài | Bụi mọc dọc hai bên mỗi đường (`along`), vườn lá thu quanh lửa (không đổ bóng để lá luôn sáng, tránh xa hố lửa), luống bụi giữa bãi, vòng bụi cao ngoài rìa che mép bản đồ, lá rụng nằm phẳng trên đất | 537-582 |
| 27 | Cây đệ quy, lá bám cành | `growTree()` sinh cành đệ quy (hướng và độ dài ngẫu nhiên, thu nhỏ dần), gộp mọi cành thành một mesh bằng `mergeGeometries`; `leafy()` gắn lá quanh các đốt cuối | 585-616 |
| 28 | Cây đỏ và cây chết | Cây đỏ: cành + lá đỏ bầu dục + vài khối cầu tối bên trong chống nhìn xuyên; cây chết: đệ quy sâu 5, có cắt tỉa ngẫu nhiên; vòng cây ngoài rìa xen kẽ hai loại | 617-635 |

## 6. Đá và công trình (6)

| # | Kỹ thuật | Mô tả | Vị trí |
| - | -------- | ----- | ------ |
| 29 | Trụ đá vát cạnh | Bệ, thân, hai đai tối, mũ, chóp `ConeGeometry` 4 cạnh xoay 45°, mảng rêu xanh; dùng làm trụ cổng, trụ tường thấp và trụ hàng rào | 643-657 |
| 30 | Tường thấp răng cưa | Thân + thanh trên bằng `RoundedBox`, hàng chóp nhỏ (`ConeGeometry` 4 cạnh) dọc mép tạo răng cưa | 658-667 |
| 31 | Vòm đá dốc xuống | 6 khối vát cạnh xếp dọc đường cong `y0 + 0.28·sin(πt) − 0.95·t²` từ trụ này sang trụ kia, mỗi khối xoay theo tiếp tuyến bằng cơ sở trực chuẩn | 668-685 |
| 32 | Hàng rào sắt | Thanh đứng và mũi giáo bằng `InstancedMesh`, hai thanh ngang, trụ đá thấp ở hai đầu và mỗi 2.3 đơn vị | 686-707 |
| 33 | Bia mộ đùn từ Shape | `ExtrudeGeometry` từ `Shape` có cung tròn, bevel 3 đoạn rồi làm mượt normal; thêm biến thể bia hộp và thánh giá, bệ, rãnh khắc, nghiêng ngẫu nhiên | 708-744 |
| 34 | Hầm mộ mái ván | Mái dốc là hộp nghiêng 0.42 rad với texture ván vẽ canvas (rãnh chạy dọc mái, dùng luôn làm `bumpMap`), khối tối bên cạnh, gờ trên có quả bí, thùng gỗ nắp xanh | 745-780 |

## 7. Đạo cụ và nhân vật (8)

| # | Kỹ thuật | Mô tả | Vị trí |
| - | -------- | ----- | ------ |
| 35 | Đèn lồng | Đế, thân kính `emissive` cam, 4 trụ, gờ mái, mái chóp, vòng treo `TorusGeometry` nằm ngang; tuỳ chọn bệ đá | 802-826 |
| 36 | Nến, lọ, gốc cây | Nến trụ + hạt lửa nhỏ + quầng; lọ gốm có nắp vòm; gốc cây cắm nến; nến nhỏ rải quanh bãi bia | 827-855 |
| 37 | Hố lửa | Chảo `SphereGeometry` nửa cầu kim loại, vành than `TorusGeometry` HDR đỏ, than hồng `MeshBasicMaterial` HDR, 4 khúc củi cháy xém, vòng đá nhỏ | 856-875 |
| 38 | Khúc gỗ và bí ngô | Khúc gỗ dùng mảng vật liệu: thân vỏ có bump + hai mặt cắt có texture vòng năm vẽ canvas; bí ngô là mặt cầu bị bóp và tạo múi bằng `cos(8θ)` trên toạ độ đỉnh | 876-899 |
| 39 | Bóng ma bằng `LatheGeometry` | Đường viền 11 điểm xoay 36 phân đoạn (thân, thắt eo, đầu phình), gấu áo lượn sóng bằng cách nâng đỉnh thấp theo `sin(4θ)`; `MeshPhysicalMaterial` có `sheen` như vải; tay là hai mặt cầu bị kéo dài, mắt bóng `clearcoat` | 916-938 |
| 40 | Bộ xương chibi gắn khớp | Sọ to (đầu to thân nhỏ), cột sống, lõi tối để lộ khe giữa 5 vòng sườn `TorusGeometry` dẹt, xương chậu; tay chân là pivot `Group` | 939-968 |
| 41 | Sọ dùng chung | `makeSkull()` dựng một lần (sọ, gò má, hàm, hốc mắt, lỗ mũi, đường miệng), dùng cho đầu bộ xương và sọ đạo cụ to gấp đôi ngồi trên thùng gỗ, ba cây nến cắm trên đỉnh | 906-915, 969-986 |
| 42 | Animation nhàn rỗi | Ma bay lượn trên đường, nhấp nhô và luôn quay mặt về phía xương; xương nhìn về phía ma và camera, gật đầu, tay đung đưa; ngọn lửa và đèn nhấp nháy | 1140-1152 |

## 8. Hiệu ứng (6)

| # | Kỹ thuật | Mô tả | Vị trí |
| - | -------- | ----- | ------ |
| 43 | Billboard instancing | `billboards()`: `InstancedBufferGeometry` một quad, thuộc tính `aPos` (vị trí) và `aDat` (tuổi, rộng, cao, hạt giống) cập nhật động; đỉnh dịch trong không gian view nên luôn quay mặt camera và co theo phối cảnh (không lệ thuộc giới hạn kích thước point) | 989-1009 |
| 44 | Lưỡi lửa | Fragment shader vẽ lưỡi lửa: bề rộng thu dần theo chiều cao, uốn lượn theo thời gian, sọc dọc, màu vàng → cam → đỏ (HDR, cộng màu) | 1015-1025 |
| 45 | Cột khói | Fragment shader: nhiễu fbm làm mép đĩa gồ ghề, pháp tuyến giả lập từ toạ độ điểm để tô sáng cam ở đáy (gần lửa) và xanh nhạt ở đỉnh; 90 luồng bay ngược gió lên và ra xa camera | 1026-1036 |
| 46 | Tàn than | 40 hạt HDR nhỏ bay lên, lắc ngang, chớp theo thời gian, tắt dần từ vàng sang đỏ | 1037-1043 |
| 47 | Bụi và lấp lánh của ma | 160 hạt bụi xanh nhạt trôi khắp khung, chớp riêng; 22 hạt lấp lánh sinh ra quanh vị trí hiện tại của ma | 1044-1053 |
| 48 | Trạng thái hạt phía CPU | Mỗi nhóm giữ mảng trạng thái (tuổi, vận tốc, hạt giống), hồi sinh khi hết tuổi, ghi thẳng vào `Float32Array` của thuộc tính instance rồi đánh dấu cần tải lại | 1054-1065, 1153-1195 |

## 9. Camera, giao diện, tối ưu (7)

| # | Kỹ thuật | Mô tả | Vị trí |
| - | -------- | ----- | ------ |
| 49 | Orbit giảm chấn | `OrbitControls` có damping, chặn góc cực và khoảng cách, tắt pan | 1067-1068 |
| 50 | Trôi camera trong biên | `autoRotate` cộng giới hạn azimuth ±0.35 rad quanh góc hiện tại; chạm biên thì đảo dấu tốc độ; kéo tay thì gỡ giới hạn, thả tay thì đặt lại | 1077-1085, 1133-1137 |
| 51 | Bay giữa 4 góc nhìn | `goTo()` nội suy vị trí và target theo `easeOutCubic` 1.4 s, xong thì cập nhật lại biên azimuth; góc mặc định trùng camera của ảnh tham chiếu | 1069-1076, 1087-1091, 1125-1132 |
| 52 | Panel Tools theo transitions-dev | Panel mở bằng opacity + translate + blur với biến CSS `--panel-*`, thời gian mở / đóng khác nhau; có `prefers-reduced-motion` | 53-57 |
| 53 | Dock và công tắc | Khung dock viền vàng có hai hình thoi ở mép, 3 nút vuông; Bloom / Edge blur / Sharpen / Auto-orbit là checkbox nối thẳng vào pass và controls; nút sách đổi góc nhìn, nút khoá bật / tắt auto-orbit, nút `?` hiện gợi ý | 60-75, 1092-1102 |
| 54 | Màn hình khởi động, báo lỗi, hook | Lớp "Lighting the fire…" mờ dần sau khung hình đầu; lỗi JS và promise hiện toàn màn hình; `window.__app` (`frames`, `boot`, `scene`, `camera`, `renderer`, `controls`); `?debug` hiện FPS | 76-77, 124-127, 1119-1123, 1199-1203 |
| 55 | Giảm tải máy cảm ứng | `COARSE` giảm shadow map, canvas nền và `DENSITY` (số lá) xuống; `?scale` chỉnh độ phân giải; `Timer` thay `Clock` (đã deprecated) và tự xử lý tab ẩn | 137-138, 286, 342, 1119-1120 |

## Thứ tự mỗi frame

```
loop        Timer → camera (bay giữa góc nhìn hoặc trôi auto-orbit) → controls.update
            → đèn lửa / đèn lồng nhấp nháy → ma, xương → cập nhật hạt (lửa, khói, than, bụi, lấp lánh)
shadow      moon (1 map 4096²) + lửa (cube 6 mặt); lá dùng depth/distance material có gió
composer    RenderPass (MSAA 4x, HDR) → UnrealBloom → blur mép + grade + vignette + grain
            → OutputPass (ACES, sRGB) → làm nét CAS
```

## Điều khiển và tham số

| Thao tác | Tác dụng |
| -------- | -------- |
| Kéo chuột / 1 ngón | Xoay góc nhìn |
| Cuộn / chụm 2 ngón | Zoom |
| Nút sách (dưới) | Chuyển sang góc nhìn kế tiếp (4 góc) |
| Nút khoá | Bật / tắt tự trôi camera |
| Nút `?` | Hiện / ẩn dòng hướng dẫn |
| Tools | Bloom, Edge blur, Sharpen, Auto-orbit |

Tham số URL: `?scale=0.75` (hệ số độ phân giải) `?seed=<n>` (đổi cách rải cây, cỏ, bia) `?orbit=0` (tắt trôi camera, hữu ích khi chụp ảnh so sánh) `?debug` (FPS, số draw call, số lá).

## So với index_cartoon_stylized_3d.html

| | index_cartoon_stylized_3d.html (Clash Village) | index_stylized_3d_diorama.html (Fireside Graveyard) |
| - | - | - |
| Phong cách | Hoạt hình sáng: toon 3 nấc, viền, màu bão hoà | Diorama "creepy-cute" tối: bề mặt smooth vát cạnh, tương phản đèn ấm / nền tối trung tính |
| Pipeline | Một lần render, MSAA gốc, viền bằng hull | `EffectComposer` HDR: bloom, blur mép, grade, ACES, làm nét CAS |
| Ánh sáng | Trời + mặt trời, bóng mây | Ánh trăng, lửa trại đổ bóng cube, đèn lồng nhấp nháy, quầng cộng màu, decal bóng tiếp xúc |
| Mặt đất | Canvas 1152² kẻ ô, đường spline | Canvas 4096² mềm mép + ~1000 viên cuội 3D instancing |
| Cây cỏ | Cây tròn, thông, bụi instancing | ~30 000 thẻ lá gập-cong chunk theo ô, gió lay, cây đệ quy có lá bám cành |
| Hiệu ứng | Khói ống khói, bụi, đạn | Lưỡi lửa, khói lit có nhiễu, than, bụi, lấp lánh: tất cả billboard instancing trên GPU |
| Bố cục | Vẽ tay theo lưới | Đo từ ảnh tham chiếu bằng phép chiếu ngược qua camera |
| Tương tác | Chọn, kéo, đặt, xây, raid | Chỉ xem: orbit, zoom, 4 góc nhìn, công tắc hiệu ứng |
| three.js | r186 | r186 |
