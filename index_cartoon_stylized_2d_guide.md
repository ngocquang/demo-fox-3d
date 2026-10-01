# Guide: làm game theo phong cách Clash Village 2D

Tài liệu ngắn mô tả phong cách đồ hoạ (style art) của `index_cartoon_stylized_2d.html` và cách dựng một game 2D isometric theo phong cách này, kèm gợi ý áp dụng với PixiJS. Chi tiết kỹ thuật từng phần nằm trong `index_cartoon_stylized_2d.md`. Phần logic game (kinh tế, raid, shop…) làm riêng, không thuộc guide này.

## 1. Nhận diện phong cách

- **Màu bão hoà, bảng màu nhỏ dùng chung:** cỏ lime, đường đất be, đá xám ấm, gỗ nâu, mái đỏ / xanh / cam, vàng và hồng cho tài nguyên. Mọi vật lấy màu từ cùng một bảng.
- **Ánh sáng hai nguồn:** nắng ấm từ trên-trái cộng ánh trời lạnh từ trên. Mặt trên sáng nhất, mặt trái vừa, mặt phải tối nhất.
- **Tô khối bằng 5 nấc phẳng:** không chuyển màu mượt, mỗi mặt rơi vào một nấc sáng. Vùng tối ngả tím xanh, không xám hay đen.
- **Chân tối, đỉnh sáng:** mỗi khối có gradient nhẹ từ chân lên đỉnh để có chiều sâu mà không cần làm tối khe.
- **Viền cùng tông:** nét viền là màu tối của chính vật đó (khoảng 30% màu gốc), không dùng đen thuần; dày ~1.7 px, chi tiết nhỏ nét mảnh hơn khối lớn.
- **Bóng ngả xanh lạnh:** đổ về phía phải-dưới (nắng từ trái-trên), thêm bóng tiếp xúc mềm dưới chân công trình và bóng mây trôi trên đất.
- **Điểm nhấn:** viền sáng ấm ở mép khối tròn, đốm bóng loáng cứng cho vàng, kim loại, nước.
- **Hình khối mập, ít chi tiết nhỏ:** gạch, ván, ngói vảy cá, mái rạ vẽ bằng nét đơn giản, mật độ đều giữa vật lớn và nhỏ.

## 2. Quy trình làm asset

1. **Chốt bảng màu và ánh sáng trước**, dùng chung cho mọi asset để cả game nhất quán.
2. **Dựng vật từ khối cơ bản:** hộp, trụ, nón, cầu, mái dốc; xếp từ dưới lên, từ sau ra trước. Mỗi mặt tự lấy màu theo hướng của nó, không tô tay.
3. **Thêm viền và họa tiết** (gạch, ván, ngói), tính theo đơn vị thế giới để mật độ đều.
4. **Bake thành ảnh một lần:** mỗi công trình, cây, nhân vật là một sprite. Phần chuyển động (cờ, lửa, khói, tay chân nhân vật) tách ra vẽ riêng.
5. **Tạo biến thể bằng tint** (lệch màu ±10%) cho cây, bụi, đá thay vì vẽ thêm.

## 3. Dựng cảnh game

- **Góc nhìn isometric cố định** (xoay 45°, nghiêng 50°), không zoom, chỉ kéo bản đồ; 1 ô = 1 đơn vị, công trình đặt theo lưới ô.
- **Mặt đất là một ảnh lớn:** ô cỏ bàn cờ, đường đất uốn cong, quảng trường lát đá; bóng đổ và bóng tiếp xúc được bake luôn vào ảnh này.
- **Thứ tự vẽ:** đất → nước → bóng mây → vật thể theo khoảng cách (xa vẽ trước, gần vẽ sau) → khói → lớp haze và vignette.
- **Bố cục:** quảng trường ở giữa, tường vòng quanh có cổng ở chỗ đường cắt, công trình tài nguyên và phòng thủ rải đều, rừng ở ngoài, vài cây bụi trong làng.
- **Làm cảnh sống động:** cờ lay, lửa trại, khói ống khói, gió lay cây, dân làng đi trên đường, mặt nước gợn sóng.

## 4. Áp dụng với PixiJS (v8)

| Việc cần làm | Dùng gì trong PixiJS |
| --- | --- |
| Đưa sprite đã bake vào game | `Texture` từ canvas (`CanvasSource`) + `Sprite` |
| Xa vẽ trước, gần vẽ sau | `sortableChildren` + `zIndex` |
| Bóng đổ, bóng mây (nhân màu) | `blendMode = 'multiply'` |
| Chuyển màu chân → đỉnh khi vẽ vector | `Graphics` + `FillGradient` |
| Viền cho sprite tạo lúc chạy | `OutlineFilter` của `pixi-filters` |
| Đổi hướng nắng lúc chạy (nâng cao) | `Filter` viết GLSL |
| Gió lay, cờ, nhân vật | `Ticker` + `skew` / `rotation` của sprite |
| Camera kéo bản đồ (pan), không zoom | `position` của container chứa cả thế giới |
| Haze, vignette | lớp CSS phủ lên canvas |

Cách nhanh nhất: bake sprite bằng Canvas 2D như file demo, để PixiJS lo sắp xếp, hiệu năng và camera.

## 5. Bảng nghiệm thu style

| Mục | Giá trị đúng |
| --- | --- |
| Số nấc sáng | 5, độ sáng 104 / 152 / 200 / 236 / 255 trên 255 |
| Hệ số sáng mặt trên / trái / phải | ≈ 0.96-0.99 / 0.79 / 0.62-0.70 |
| Gradient chân → đỉnh | 0.84 → 1.06 (tán cây 0.66 → 1.14) |
| Viền | 30% màu gốc, dày 1.7 px × (0.4 → 1 theo kích thước vật) |
| Màu bóng (nhân) | rgb(180, 197, 215) |
| Đối chiếu ảnh chụp | cỏ sáng ≈ (146, 211, 70), mái đỏ ≈ (193, 68, 45), đá ≈ (140, 132, 121) |

## 6. Lỗi thường gặp

- Trộn màu thẳng trên giá trị sRGB làm mất vùng tối ngả tím: phải trộn ở linear rồi mới đổi lại sRGB.
- Viền đen thuần thay vì viền cùng tông làm cảnh thô và nặng.
- Chia quá nhiều nấc sáng hoặc dùng chuyển màu mượt thì mất cảm giác hoạt hình.
- Quên bóng tiếp xúc dưới chân công trình, vật như nổi khỏi mặt đất.
- Game không zoom nên bake sprite đúng độ phân giải hiển thị: nét, viền giữ nguyên độ dày; đổi kích thước màn hình nhiều thì bake lại.
