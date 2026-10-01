# Phong cách Cartoon Stylized 3D cho three.js

Tài liệu mô tả **art style** của `index_cartoon_stylized_3d.html` (Clash Village): nhìn vào đâu, thấy gì, vì sao nó ra "chất" hoạt hình. Không có code. Mọi yếu tố ở đây đều làm được bằng tính năng có sẵn của three.js, trong một lần render, không cần post-process.

Phần kỹ thuật (shader, số dòng) xem `index_cartoon_stylized_3d.md`.

---

## 1. Tinh thần chung

Một ngôi làng đồ chơi đặt trên bàn, nhìn từ trên xuống. Mọi thứ mập, tròn, sáng, hơi lắc lư như đang sống. Người xem phải đọc được từng công trình chỉ bằng hình dáng và màu, kể cả khi thu nhỏ.

Ba từ khoá: **đồ chơi**, **ấm áp**, **dễ đọc**.

---

## 2. Hợp đồng phong cách

| Nên | Không nên |
| - | - |
| Màu bão hoà, sáng, như kẹo | Màu thật, xám, nhạt |
| Đổ bóng phẳng theo nấc | Đổ bóng mượt kiểu vật lý (PBR) |
| Viền tối cùng tông với vật | Viền đen thuần, hoặc không viền |
| Khối mập, bo góc, tỉ lệ phóng đại | Khối sắc cạnh, tỉ lệ thật |
| Họa tiết vẽ tay đơn giản | Texture ảnh chụp |
| Bóng ngả xanh tím | Bóng xám hoặc đen |
| Chuyển động nảy, có đà | Chuyển động tuyến tính, cứng |
| Camera xiên cố định, gần isometric | Camera góc rộng, góc thấp |

Những thứ phá phong cách ngay: tone mapping kiểu điện ảnh (làm nhạt màu), ánh sáng môi trường từ HDRI, bloom, SSAO, viền dựng bằng post-process (dày mỏng theo độ phân giải).

---

## 3. Màu sắc

### Bảng màu

Mọi màu lấy từ một bảng cố định. Không tự pha màu mới cho từng vật.

| Nhóm | Màu | Ghi chú |
| - | - | - |
| Không khí | haze `#a6dcae` | Nền và sương cùng màu xanh lá nhạt, để rìa cảnh tan vào cỏ |
| Cỏ | `#9bd94a`, `#8fce40`; viền `#63a83b`, `#4c8c2e` | Hai tông xen kẽ kiểu bàn cờ |
| Đất, đường | `#e9c98c`, mép `#cfa267`, tối `#bf9459`, sáng `#f6e0b2` | Vàng be ấm |
| Đá | `#d8cfbb`, `#bdb5a3`, tối `#a39a86` | Đá ngả kem, không xám lạnh |
| Gỗ | `#9a5b2e`, tối `#6b3a1d`, sáng `#c78a4d` | Nâu cam |
| Tường | vữa `#f4e7c4`, kem `#f3e6c2` | Trắng ngà, không trắng tinh |
| Mái | đỏ `#d8432b`, xanh `#3f86d9`, cam `#ee8a26`, tím `#8b5cc7`, ngọc `#2fa39a` | Mỗi loại công trình một màu mái, đọc được từ xa |
| Tài nguyên | vàng `#ffc933` / `#d99a10`, hồng elixir `#ea4fb8` / `#b02a86` | Màu "quý", luôn có độ bóng |
| Kim loại | `#5d6675` | Xám xanh, dùng ít |

### Quy tắc màu

- Mỗi vật có **một màu chủ đạo** và tối đa hai màu phụ. Màu mái là thứ nhận diện chính.
- Mặt sáng ngả vàng ấm, mặt khuất ngả xanh tím. Không bao giờ để bóng thành xám đục.
- Màu hiện ra trên màn hình gần đúng màu đã chọn: không tone mapping, không lọc màu.
- Rìa màn hình có vignette nhẹ màu xanh rừng đậm, kéo mắt vào giữa làng.

---

## 4. Ánh sáng

Hai nguồn sáng là đủ:

- **Mặt trời**: ánh vàng kem, chiếu xiên từ trên trái phía trước, đổ bóng rõ xuống đất.
- **Ánh trời / ánh đất**: trời xanh nhạt từ trên, đất **tím lạnh** từ dưới. Đây là núm quan trọng nhất của cả phong cách: nó làm phần khuất sáng có màu, nên cảnh trông "hoạt hình" thay vì "render".

Không có ánh sáng môi trường, không phản chiếu. Không khí xa là sương cùng màu nền, tăng dần theo khoảng cách.

---

## 5. Đổ bóng bề mặt (toon shading)

- Ánh sáng trên bề mặt chia **5 nấc phẳng**, chuyển nấc đột ngột, không gradient.
- Nấc tối nhất vẫn giữ khoảng 40% độ sáng: bóng vẫn còn màu, không đen.
- Ít nấc hơn (3) thì phẳng kiểu cel anime; nấc tối thấp hơn thì gắt kiểu manga. Phong cách này đứng giữa.

Thêm hai lớp sáng phụ:

- **Rim light**: viền sáng ấm, mềm quanh mép khối ở phía ngược camera. Tách vật khỏi nền.
- **Đốm bóng cắt cứng**: chỉ trên vàng, kim loại, nước. Đốm là mảng sáng có mép sắc như được tô bằng cọ, không phải chấm mờ. Tường, gỗ, cỏ hoàn toàn không bóng.

---

## 6. Viền

Viền là đặc trưng dễ nhận nhất.

- **Dày cố định theo pixel màn hình** (khoảng 1.7 px), không đổi khi zoom gần hay xa, không đổi trên màn retina.
- **Màu viền = màu vật, tối đi còn khoảng 30%**: mái đỏ có viền nâu đỏ, cỏ có viền xanh đậm, gỗ có viền nâu sẫm. Đây là khác biệt lớn nhất so với viền đen kiểu truyện tranh.
- **Chi tiết nhỏ nét mảnh hơn**: cửa sổ, đinh tán, cành nhỏ có nét khoảng một nửa so với khối lớn. Nét chính bao hình khối, nét phụ chỉ gợi ý.
- Viền liền mạch ở mọi góc, không hở tại cạnh hộp.
- Cây lay theo gió thì viền lay đúng theo cây.
- Trạng thái tương tác đổi màu và độ dày viền: được chọn thì viền đôi dày và màu đặc; đặt hợp lệ là xanh, sai chỗ là đỏ.

---

## 7. Hình khối

### Nguyên tắc

- **Mập**: khối chính luôn bo góc nhẹ. Chỉ dầm gỗ, chi tiết mảnh mới giữ cạnh sắc.
- **Phóng đại**: mái to và cao hơn thật, nền đá dày và rộng hơn thân nhà, cửa nhỏ so với tường. Công trình trông như đồ chơi gỗ.
- **Hình dáng đọc được từ xa**: mỗi công trình có bóng dáng riêng (mái chóp, tháp tròn, mái dốc hai bên, kho tròn).
- **Ít đa giác**: hình học đơn giản (hộp, trụ, nón, cầu, lăng trụ tam giác) ghép lại. Độ chi tiết đến từ cách ghép, không từ số đa giác.

### Công trình

- Xây theo lớp: nền đá → thân → dầm, cột gỗ → mái → đỉnh trang trí (quả cầu vàng, cờ).
- Mái dốc là lăng trụ tam giác; mái chóp là nón 4 cạnh xoay 45°; tháp có mái nón tròn.
- Cấp độ đọc bằng mắt: nhà cấp cao thêm viền vàng, thêm tầng, thêm cờ.

### Đá

Đá là khối tròn bị bóp méo ngẫu nhiên nhưng vẫn mập. Khe lõm tối hơn mặt lồi, như được tô bóng tay.

### Cây

- **Cây tròn**: 4–6 quả cầu chồng nhau, mỗi quả một tông xanh khác.
- **Thông**: 4 nón chồng, sáng dần lên ngọn.
- Gốc xoè nhẹ, có vài cành. Rừng xa dùng phiên bản đơn giản hơn.
- Mỗi cây lệch màu khoảng ±10% và chiều cao ±10% để rừng không lặp lại.

### Nhân vật

Đầu to (khoảng nửa chiều cao thân), mắt là hai chấm đen, tóc là cầu dẹt. Tay chân là khối riêng, xoay quanh khớp vai, khớp hông. Viền nhân vật mảnh hơn công trình.

---

## 8. Chất liệu và họa tiết

Không có texture ảnh. Màu đến từ từng khối; họa tiết là các vân vẽ đơn giản phủ lên màu đó:

| Họa tiết | Dùng cho |
| - | - |
| Ván gỗ dọc / ngang | Tường gỗ, hàng rào, sàn |
| Gạch so le | Tường đá, nền |
| Ngói vảy cá | Mái |
| Rạ | Mái tranh, kho cỏ |
| Đá cuội | Nền, tường thành |
| Vân lá / vân đá | Tán cây, đá |
| Sọc | Cờ, mái vải |
| Vỏ cây | Thân cây |

Quy tắc:

- Họa tiết chỉ **làm tối** màu gốc, không đổi màu. Mạch vữa, khe ván tối tới khoảng 55%, không tới đen, để vẫn đọc là "màu tường" chứ không thành lưới.
- Mỗi viên gạch, mỗi tấm ván lệch sáng tối vài phần trăm: trông như vẽ tay, không như máy in.
- **Kích thước họa tiết theo thế giới**: viên gạch trên tháp lớn và tháp nhỏ to bằng nhau.
- Mép họa tiết mềm vừa đủ để không răng cưa khi nhìn xa.

### Chiều sâu giả

Mỗi khối **tối ở chân, sáng ở đỉnh** (khoảng 84% → 106% độ sáng; tán lá chênh mạnh hơn 66% → 114%). Đây là "AO vẽ sẵn": khối trông nặng, đứng vững trên mặt đất, không cần hiệu ứng nào.

Phép thử: tắt mặt trời, chỉ giữ ánh trời/đất, cảnh vẫn phải đọc được khối nhờ gradient này và viền.

---

## 9. Mặt đất

Mặt đất là một bức tranh vẽ phẳng, không phải địa hình 3D. Thứ tự lớp:

1. **Cỏ** theo ô bàn cờ hai tông, chênh nhau rất nhẹ. Mỗi ô có vạch sáng mép trên và vạch tối mép dưới, gợi ô đất hơi nổi.
2. **Đường đất** nhiều lớp: mép cỏ mờ → mép đất tối → lõi đất → dải sáng giữa đường. Thêm hai vệt bánh xe và rất nhiều sỏi nhỏ có bóng.
3. **Quảng trường lát đá**: viên đá bo góc, mỗi viên có highlight trên và bóng dưới.
4. **Chi tiết rải**: hàng nghìn chùm cỏ cong, hoa 5 cánh, sỏi.

Mặt đất phải giữ nét ở xa dù camera nhìn xiên.

---

## 10. Nước

Nước không phản chiếu, không trong suốt thật. Nó là mảng màu phẳng:

- Độ sâu chia **3 nấc** từ xanh ngọc ở bờ tới xanh dương ở giữa.
- Sóng là các **vạch trắng mảnh** chạy chậm, không phải gợn mượt.
- Bờ có **vòng bọt trắng** lượn sóng, méo nhẹ theo thời gian.

---

## 11. Bóng

- **Bóng mặt trời**: đổ rõ, mép hơi mềm.
- **Bóng tiếp xúc**: vệt tròn mờ dưới mỗi công trình, **màu xanh lá đậm** chứ không đen, để hoà với cỏ. Neo vật xuống đất.
- **Bóng mây**: vài mảng bóng lớn trôi chậm trên mặt đất, mây thì không nhìn thấy. Làm cảnh có thời tiết, có sự sống.

---

## 12. Chuyển động

Chuyển động là một nửa "chất" hoạt hình. Mọi thứ có đà và giảm chấn, không có gì dừng đột ngột.

| Hiệu ứng | Cảm giác |
| - | - |
| Chọn công trình | Nảy squash & stretch: dẹt ra rồi vươn lên, rung tắt dần trong khoảng 0.8 s |
| Công trình xuất hiện | Phóng to vượt quá kích thước một chút rồi bật về (overshoot) |
| Cây | Lay theo gió: gốc đứng yên, ngọn lay nhiều, mỗi cây lệch pha |
| Cờ | Phấp phới, xoay qua lại, co giãn nhẹ |
| Khói | Cụm tròn bay lên, phồng ra rồi tan, trắng ngả xanh nhạt |
| Nhân vật đi | Chân bước rộng, tay vung ngược pha, thân nhún theo nhịp bước |

Squash & stretch giữ thể tích: vươn cao thì co ngang, dẹt xuống thì phình ngang.

---

## 13. Camera và bố cục

- Nhìn xiên khoảng **50°**, xoay **45°**, góc nhìn hẹp (FOV khoảng **28°**): gần isometric nhưng vẫn có phối cảnh nhẹ. Góc rộng hơn làm công trình méo, mất cảm giác "đồ chơi trên bàn".
- Kéo, zoom, xoay đều mượt và có quán tính.
- Làng ở giữa, rừng bao quanh, rìa rừng tan vào sương khi zoom xa.
- Khoảng sương khớp với khoảng zoom: gần thì trong, xa thì rìa cảnh mờ dần.

---

## 14. Ánh xạ sang three.js

Bảng này chỉ để biết dùng khối nào của three.js cho từng yếu tố hình ảnh.

| Yếu tố | Khối dựng trong three.js |
| - | - |
| Toon 5 nấc | `MeshToonMaterial` + gradient map 5 nấc, lọc nearest |
| Rim, đốm bóng cắt cứng, họa tiết, gió | Mở rộng `MeshToonMaterial` qua `onBeforeCompile` |
| Viền theo pixel | Inverted hull (mesh mặt sau, đẩy theo normal) |
| Màu từng khối | Vertex color, gộp nhiều khối thành một mesh |
| Khối bo góc | `RoundedBoxGeometry` |
| Cây, đá, bụi | `InstancedMesh`, tint màu từng instance |
| Mặt đất | `CanvasTexture` |
| Nước | `ShaderMaterial` riêng |
| Ánh sáng | `DirectionalLight` ấm + `HemisphereLight` (trời xanh / đất tím) |
| Sương | `Fog` cùng màu nền |
| Bóng mây | Khối vô hình chỉ đổ bóng |
| Bóng tiếp xúc | Plane trong suốt với gradient tròn |
| Vignette | CSS đè lên canvas |

---

## 15. Bảng núm chỉnh phong cách

| Núm | Giá trị gốc | Đẩy lên | Kéo xuống |
| - | - | - | - |
| Số nấc toon | 5 | Mượt dần, mất chất | 3: phẳng kiểu cel |
| Nấc tối nhất | ~40% | Phẳng kiểu flat design | Tương phản gắt kiểu manga |
| Màu đất của ánh sáng | Tím lạnh | Ấm: hoàng hôn | Xám: bóng đục, cảnh "chết" |
| Độ dày viền | ~1.7 px | Nét đậm kiểu comic | Mảnh, gần low-poly |
| Độ tối viền | 30% màu vật | Viền nhạt, mềm | Viền đen thuần, cứng |
| Chênh nét lớn / nhỏ | Nhỏ bằng ~nửa lớn | Nét đều | Chênh rõ hơn |
| Rim light | Nhẹ, mềm | Phát sáng viền kiểu glow | Rim mảnh, gần như không |
| Gradient chân → đỉnh | 84% → 106% | Khối nổi, AO đậm | Phẳng |
| Gió | Nhẹ | Lay mạnh, kiểu bão | Gần đứng yên |
| Sương | Xa | Thấy nhiều rừng | Không khí dày, mơ màng |

---

## 16. Checklist khi dựng cảnh mới

1. Chốt bảng màu trước: cỏ, đất, đá, gỗ, 4–5 màu mái, màu "quý".
2. Ánh sáng ấm từ trên, tím lạnh từ dưới; không tone mapping.
3. Toon 5 nấc, nấc tối còn màu.
4. Viền theo pixel, màu tối cùng tông, chi tiết nhỏ nét mảnh.
5. Khối mập, bo góc, tỉ lệ phóng đại; mỗi công trình một bóng dáng.
6. Họa tiết vẽ tay chỉ làm tối màu gốc; gradient chân tối, đỉnh sáng.
7. Mặt đất vẽ phẳng nhiều lớp; nước 3 nấc có bọt; bóng tiếp xúc xanh lá.
8. Cây, đá lệch màu và kích thước; mọi thứ lắc lư nhẹ.
9. Chuyển động nảy, overshoot, giảm chấn.
10. Camera xiên 50°, FOV hẹp, sương khớp khoảng zoom.
