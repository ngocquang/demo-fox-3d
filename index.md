# index.html — Forest Pond: các kỹ thuật 3D

Một file HTML duy nhất (`index.html`, ~5000 dòng), three.js r169 nạp bằng ES module từ jsDelivr, không build.
Mọi tài nguyên (texture PBR, lá, cây, đá, cá, cáo) đều sinh thủ tục lúc khởi động; chỉ font Helvetiker (chữ "ZPS") và HDRI tuỳ chọn lấy từ mạng, cả hai đều có fallback.

**Tổng: 46 kỹ thuật, chia 7 nhóm.** Số này là số dòng trong các bảng dưới. Tách/gộp kỹ thuật khác đi thì ra số khác. Cột "Vị trí" là số dòng trong `index.html` (dấu `≈` là ước lượng theo comment gần nhất); mỗi mục lớn có banner `// ==== TÊN ====` để grep.

## 1. Render và ánh sáng (8)

| # | Kỹ thuật | Mô tả | Vị trí |
| - | -------- | ----- | ------ |
| 1 | HDR render target | Scene render vào target half-float + depth texture float, không dùng MSAA | `HDR_RT` ≈3869, `ScenePass` 3889 |
| 2 | Tone mapping ACES Filmic | Áp ở `ToneMapPass` (không dùng tone mapping của renderer), ra sRGB | 215-225, `ToneMapPass` 4377 |
| 3 | Auto-exposure trên GPU | Đo luminance, thích nghi dần theo thời gian (như mắt người) | `ExposurePass` 4244 |
| 4 | Shadow map mặt trời 4096² | Khớp quanh ao, cập nhật 1 lần/frame; tra cứu tuỳ chỉnh 16 tap và mờ dần ở mép frustum | `PARAMS` 139, 390-430, 574 |
| 5 | IBL bằng PMREM | Env map từ HDRI (RGBELoader) hoặc env sky+canopy tự sinh | 1446, 1463-1474 |
| 6 | Sky dome thủ tục | Render vào cube map làm `scene.background` (thấy qua khe tán lá) | 1426-1450 |
| 7 | Đốm nắng = bóng thật của tán lá | Hàng nghìn card lá alpha-test đổ bóng vào shadow map, cùng gió với bản hiển thị | `leafCardGeometry` 2170, `makeShadowDepthMaterial` 799 |
| 8 | Sương mù tách không khí/nước | Mỗi tia nhìn được chia đoạn trong không khí và trong nước, fog FogExp2 cho cả hai | 540 |

## 2. Post-processing (7)

| # | Kỹ thuật | Mô tả | Vị trí |
| - | -------- | ----- | ------ |
| 9 | GTAO | Ambient occlusion chỉ dùng depth, nửa độ phân giải | 3939, `LightingPass` 4122 |
| 10 | Volumetric light | Ray-march qua shadow map, nửa độ phân giải, chạy cả trên và dưới nước | 4009 |
| 11 | God rays hướng tâm | Screen-space qua mặt nạ che của tán lá, 1/4 độ phân giải | 4062 |
| 12 | Bloom | `UnrealBloomPass` (0.25 / 0.4 / 0.9) | 4512 |
| 13 | Bokeh DoF | Gather bokeh, có autofocus theo cá gần nhất | `DoFPass` 4328, `AutoFocus` 4474 |
| 14 | SMAA | Chống răng cưa hậu kỳ thay cho MSAA | 4515 |
| 15 | Color grade và lens | Bóng teal/sáng ấm, contrast, vignette, grain 2%, gợn hình và chromatic aberration khi dưới nước, giọt nước trên ống kính | `GradePass` 4452 |

## 3. Nước (11)

| # | Kỹ thuật | Mô tả | Vị trí |
| - | -------- | ----- | ------ |
| 16 | Sóng Gerstner và FBM | 4 sóng Gerstner cộng FBM trong vertex shader, lưới 256² | GLSL ≈462-520, `buildWater` 2911 |
| 17 | Ring ripples | 16 vòng sóng lan (cá đớp mồi, bọt khí, camera chạm mặt nước) | `spawnRipple` 2693, `ringRipple` GLSL |
| 18 | Hai normal map trượt | Chi tiết sóng nhỏ tần số cao | 2810 |
| 19 | Planar reflection | Camera gương + oblique near plane tại mặt nước, nửa độ phân giải | `PlanarReflection` 2702 |
| 20 | Khúc xạ | Lấy từ bản copy frame HDR đục + depth tuyến tính của mọi thứ phía sau mặt nước | `ScenePass` ≈3917 |
| 21 | Hấp thụ Beer–Lambert | Đỏ tắt trước, cộng in-scattering của phù sa/tannin | 2846 |
| 22 | Fresnel Schlick | IOR 1.33, F0 = 0.0204 | 2854 |
| 23 | Glint GGX và lấp lánh | Phản chiếu mặt trời + sparkle từ normal tần số cao | 2857 |
| 24 | Bọt ven bờ và lớp phấn hoa | Foam/scum theo đường bờ, màng phấn hoa | 2866 |
| 25 | Cửa sổ Snell và phản xạ toàn phần | Nhìn từ dưới: tia khúc xạ thấy cả bầu trời trong nón 97° | 2883-2900 |
| 26 | Caustics | Hàm caustic động chiếu theo hướng nắng lên mọi bề mặt dưới mực nước, nhân vào phần nắng đã có bóng nên chỉ hiện trong vệt sáng | GLSL ≈433 |

## 4. Vật liệu và shader (8)

| # | Kỹ thuật | Mô tả | Vị trí |
| - | -------- | ----- | ------ |
| 27 | Vá vật liệu bằng `onBeforeCompile` | Mọi standard material được chèn fog tách đoạn, dải ướt trên mực nước, caustics, hấp thụ ánh sáng dưới nước | 583-820, `patchWorldMaterial` 754 |
| 28 | PBR triplanar | Đá dùng triplanar, rêu ở mặt hướng lên, ướt dưới mực nước | 1689-1800 |
| 29 | Parallax occlusion mapping | Chỉ bật cho các tảng đá lớn nhất (`USE_POM`) | 1727, 1756 |
| 30 | Bake texture PBR trên GPU | 2048 px: loam, silt, rock, moss, bark ×3 loài; noise tuần hoàn và Worley tuần hoàn | 822-1125, GLSL 321, 346 |
| 31 | Atlas vẽ bằng canvas | Lá, lớp lá mục, dương xỉ, cỏ | 1125-1372 |
| 32 | Lá trong suốt khi ngược sáng | Translucency + crown normals cho tán cây | 757, 2218 |
| 33 | Gió lay | Thân + rung lá, dùng chung cho tán, cỏ, dương xỉ, cành và cả vật đổ bóng | GLSL 525, `windCode` 678 |
| 34 | `MeshPhysicalMaterial` cho cá | Clearcoat, sheen, iridescence, hoa văn koi/trout thủ tục, normal map vảy | `buildFish` 3214 |

## 5. Hình học và tối ưu (7)

| # | Kỹ thuật | Mô tả | Vị trí |
| - | -------- | ----- | ------ |
| 35 | Địa hình heightfield | Hình dạng ao + độ cao, mesh có warp lưới | 1495-1688 |
| 36 | Đá thủ tục | Icosphere + FBM làm méo, vài mặt phẳng cắt (mặt vỡ), đáy dẹt; tham số kích thước, độ lồi lõm, số mặt cắt | `makeRockGeometry` 1691 |
| 37 | Cây đệ quy và tube builder | 40 cây (beech / oak / alder) với chiều cao, số cành, độ xoè tán riêng từng loài; gốc nở rễ, cành, rễ lộ | `growTree` 2052, `buildTube` 1878, 2619 |
| 38 | LOD 3 cấp | `THREE.LOD`, displacement vỏ cây ở LOD gần | `buildTrees` 2184-2216 |
| 39 | Impostor billboard | Billboard trụ instanced cho rừng xa (không đổ bóng) + dải cây mờ khói phía sau | 2260-2414 |
| 40 | GPU instancing | Lá, dương xỉ, cỏ, lớp lá mục, bụi cây, cá, lá nổi | `scatterInstanced` 2509, 2235, 3243, 2961 |
| 41 | `ExtrudeGeometry` | Chữ "ZPS" (TextGeometry, có fallback bằng shape tự vẽ) | 3796, `zpsFallbackShapes` 3743 |

## 6. Chuyển động và hạt (3)

| # | Kỹ thuật | Mô tả | Vị trí |
| - | -------- | ----- | ------ |
| 42 | Cá uốn thân trong vertex shader | Sóng cột sống + vây vỗ | `buildFishGeometry` 3027, 3236 |
| 43 | Boids | Né vật cản, hoảng sợ, lơ lửng, đớp mặt nước | `updateFish` 3282 |
| 44 | Hệ hạt | Bụi trong nắng (chỉ hiện nơi shadow map sáng), phù sa dưới nước, bọt khí (CPU), nước bắn khi camera chạm mặt | 3414-3668 |

## 7. Camera và scene (2)

| # | Kỹ thuật | Mô tả | Vị trí |
| - | -------- | ----- | ------ |
| 45 | Render layers | Camera phản chiếu chỉ thấy vật thể đã bật layer (trên mặt nước hoặc dưới đáy) | 242 |
| 46 | Camera điện ảnh | Đường Catmull-Rom 90 s (rễ → lướt mặt nước → lặn → cá → cửa sổ Snell → nổi lên → quay vòng), rung tay, focus bám cá; `F` chuyển sang bay tự do | 4521-4697 |

## Thứ tự pass mỗi frame

```
ScenePass   phản chiếu (½) → frame đục HDR + depth → bản copy khúc xạ → mặt nước + hạt
LightingPass  GTAO (½) + volumetric (½) + god rays (¼) → composite
ExposurePass  luminance GPU + thích nghi
UnrealBloomPass
DoFPass
ToneMapPass   ACES + exposure → sRGB
SMAAPass
GradePass     → màn hình
```

## Ngoài 46 kỹ thuật trên (không tính vào tổng)

- Âm thanh WebAudio thủ tục, low-pass khi dưới nước: 4698-4790.
- Render scale tự giảm khi FPS < 50: `updatePerf` 4846.
- Giao diện `lil-gui`, HUD FPS, màn hình loading, hộp báo lỗi: 4797, 4834, 206, 126.
- RNG có seed (`mulberry32` 157), noise CPU (`Noise` ≈187-205).
- Cáo low-poly dựng bằng code: 3669-3742.
- Hook test `window.__app` (`frames`, `pose`, `info`, `perf`, `measureSunCoverage`…): 4911-4978.

## Điều khiển và tham số

| Phím | Tác dụng |
| ---- | -------- |
| `F` | Chuyển camera điện ảnh ⇄ bay tự do |
| `W A S D` + chuột (click để khoá con trỏ) | Bay |
| `Shift` | Chạy nhanh |
| `Space` / `C` | Lên / xuống |
| `M` / `P` / `H` | Âm thanh / tạm dừng / ẩn UI |

Tham số URL: `?t=<giây>` `?free` `?tex=1024` `?shadow=2048` `?scale=0.75` `?nohdri` `?hdri=<url>` `?seed=<n>` `?test` (không chạy vòng lặp, tự điều khiển frame để chụp ảnh).
