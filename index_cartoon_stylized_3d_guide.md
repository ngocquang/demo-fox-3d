# Hướng dẫn phong cách Cartoon Stylized 3D cho three.js

Tài liệu này là **công thức** để dựng lại phong cách của `index_cartoon_stylized_3d.html` (Clash Village) trong một dự án three.js mới. Danh mục đầy đủ 41 kỹ thuật kèm số dòng nằm ở `index_cartoon_stylized_3d.md`; ở đây chỉ giữ thứ tự làm, đoạn code cốt lõi và các núm chỉnh.

- three.js **r186**, ES module, không build. Addon cần dùng: `BufferGeometryUtils.mergeGeometries`, `RoundedBoxGeometry`.
- Code trích nguyên văn từ `index_cartoon_stylized_3d.html`; số dòng ghi dạng `html:NNN`.
- Các bước sắp theo phụ thuộc: làm xong bước 1–6 là đã ra "chất" hoạt hình, bước 7 trở đi là đánh bóng.

---

## 0. Hợp đồng phong cách

Những thứ làm nên phong cách này, và những thứ phá nó.

**Nên:**

| Yếu tố | Cách làm |
| - | - |
| Màu | Bảng màu bão hoà, sáng; ánh sáng ấm, bóng ngả xanh tím (không xám) |
| Đổ bóng bề mặt | Toon 5 nấc phẳng, không gradient mượt |
| Viền | Nét tối **cùng tông** với vật thể, dày cố định theo pixel (~1.7 px), chi tiết nhỏ nét mảnh hơn |
| Khối hình | "Mập", bo góc (chamfer), tỉ lệ phóng đại; mái nhọn, tháp tròn |
| Chất liệu | Vertex color + họa tiết thủ tục trong shader; không texture ảnh |
| Chiều sâu | Gradient đỉnh bake sẵn (chân tối, đỉnh sáng), bóng tiếp xúc, rim light |
| Chuyển động | Nảy, squash & stretch, `easeOutBack`; mọi thứ hơi lắc lư |
| Camera | Xiên cố định ~50°, FOV hẹp (28°), gần như isometric |

**Không nên:** PBR (`MeshStandardMaterial`), tone mapping (ACES làm nhạt màu), HDRI/IBL, EffectComposer/bloom/SSAO, texture ảnh chụp, viền bằng post-process (Sobel) vì dày mỏng theo độ phân giải. Toàn bộ cảnh render trong **một** lần `renderer.render`.

---

## 1. Renderer, scene, camera

```js
// html:285
const renderer = new THREE.WebGLRenderer({ antialias: true, powerPreference: 'high-performance' });
renderer.setPixelRatio(clamp(devicePixelRatio || 1, 1.5, 2) * PARAMS.scale);
renderer.setSize(innerWidth, innerHeight);
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFShadowMap;
// ...
scene.background = new THREE.Color(PAL.haze);
scene.fog = new THREE.Fog(PAL.haze, 72, 160);
const camera = new THREE.PerspectiveCamera(28, innerWidth / innerHeight, 1, 500);
const U = { uTime: { value: 0 }, uRes: { value: new THREE.Vector2(1, 1) }, uPx: { value: 1 } };
```

- **Không** đặt `toneMapping` (mặc định `NoToneMapping`); output sRGB mặc định. Màu hex trong bảng màu hiện ra gần đúng như chọn.
- MSAA gốc (`antialias: true`) là đủ vì không có post-process.
- `U` là uniform dùng chung: `uTime` cho gió/nước, `uRes` + `uPx` cho viền theo pixel. Cập nhật trong `onResize` (`html:297`):

```js
const v = renderer.getDrawingBufferSize(new THREE.Vector2());
U.uRes.value.copy(v); U.uPx.value = renderer.getPixelRatio();
```

- Vignette là CSS, không phải pass (`html:62`):

```css
#vig { position: fixed; inset: 0; pointer-events: none; z-index: 4; background: radial-gradient(ellipse at center, transparent 58%, rgba(20,60,40,.32) 100%); }
```

**Bảng màu** (`html:273`): đặt một chỗ, mọi builder chỉ đọc từ đây. Màu haze (nền + fog) là xanh lá nhạt `0xa6dcae` để rìa cảnh hoà vào cỏ.

---

## 2. Ánh sáng: ấm từ trên, lạnh từ dưới

Đây là núm rẻ nhất mà ảnh hưởng nhiều nhất tới "chất" hoạt hình.

```js
// html:782
const sun = new THREE.DirectionalLight(0xffecc8, Math.PI * 0.52);
sun.position.set(-24, 36, 15);
sun.castShadow = true;
sun.shadow.mapSize.set(PARAMS.shadow, PARAMS.shadow);
Object.assign(sun.shadow.camera, { left: -34, right: 34, top: 34, bottom: -34, near: 1, far: 100 });
sun.shadow.bias = -0.0004; sun.shadow.normalBias = 0.04;
scene.add(sun, new THREE.HemisphereLight(0xd8efff, 0x8f9bd0, Math.PI * 0.64));
```

- Màu **đất** của Hemisphere là tím lạnh `0x8f9bd0`: mặt khuất sáng ngả xanh tím thay vì xám đục. Đổi thành xám là cảnh "chết" ngay.
- Cường độ nhân `Math.PI` vì từ r155 đèn dùng đơn vị vật lý.
- Shadow camera trực giao ôm vừa khu vực chơi; `normalBias` chống acne trên khối bo góc.

---

## 3. Vật liệu toon 5 nấc

```js
// html:306
const gradientMap = (() => {
  const t = new THREE.DataTexture(new Uint8Array([104, 152, 200, 236, 255]), 5, 1, THREE.RedFormat);
  t.minFilter = t.magFilter = THREE.NearestFilter; t.generateMipmaps = false; t.needsUpdate = true;
  return t;
})();
```

- `NearestFilter` + không mipmap là bắt buộc, nếu không các nấc bị nội suy thành gradient mượt.
- Nấc tối nhất `104` (~41%) chứ không phải 0: bóng vẫn còn màu. Hạ xuống thì tương phản gắt kiểu manga; nâng lên thì phẳng kiểu flat-design.
- Vật liệu luôn là `new THREE.MeshToonMaterial({ vertexColors: true, gradientMap })`; màu nằm trong vertex color (bước 6), nên **cả làng dùng chung một material** → ít đổi state, ít shader.

---

## 4. Tiêm shader: rim light, specular cắt cứng, họa tiết, gió

Mở rộng `MeshToonMaterial` bằng `onBeforeCompile` thay vì viết `ShaderMaterial` riêng, để giữ nguyên đèn, bóng, fog của three.js.

```js
// html:353
function makeToon({ sway = 0 } = {}) {
  const m = new THREE.MeshToonMaterial({ vertexColors: true, gradientMap });
  m.onBeforeCompile = (sh) => {
    sh.uniforms.uTime = U.uTime;
    sh.vertexShader = 'attribute vec2 aM; varying vec2 vM; varying vec2 vUvW;\nuniform float uTime;\n' + sh.vertexShader.replace('#include <begin_vertex>', `#include <begin_vertex>
      vM = aM; vUvW = uv;` + (sway ? `
      #ifdef USE_INSTANCING
        float ph = uTime * 1.7 + instanceMatrix[3].x * 0.8 + instanceMatrix[3].z * 1.1;
        transformed.xz += vec2(sin(ph), cos(ph * 0.83)) * ${sway.toFixed(3)} * max(transformed.y, 0.0);
      #endif` : ''));
    sh.fragmentShader = PAT_GLSL + '\n' + sh.fragmentShader
      .replace('#include <color_fragment>', `#include <color_fragment>
        if (vM.x > 0.5) diffuseColor.rgb *= patternShade(vM.x, vUvW);`)
      .replace('#include <emissivemap_fragment>', `#include <emissivemap_fragment>
        vec3 Vv = normalize(vViewPosition);
        totalEmissiveRadiance += vec3(1.0, 0.85, 0.68) * pow(1.0 - saturate(dot(normal, Vv)), 2.4) * 0.14;
        #if NUM_DIR_LIGHTS > 0
          float sp = pow(max(dot(normal, normalize(directionalLights[0].direction + Vv)), 0.0), 48.0);
          totalEmissiveRadiance += vec3(1.0, 0.96, 0.85) * smoothstep(0.5, 0.58, sp) * vM.y;
        #endif`);
  };
  m.customProgramCacheKey = () => 'toon2:' + sway;
  return m;
}
```

Thuộc tính `aM` (vec2) mang dữ liệu theo từng primitive:

| Kênh | Ý nghĩa |
| - | - |
| `aM.x` | ID họa tiết (`PAT.*`), 0 = trơn |
| `aM.y` | Lượng specular: 0 cho tường/gỗ, 0.5–0.7 cho vàng, kim loại, nước |

Điểm cần nhớ:

- **Rim**: `pow(1 - N·V, 2.4) * 0.14`, màu ấm. Cộng vào emissive nên không bị toon ramp cắt nấc, tạo viền sáng mềm quanh khối.
- **Specular**: `smoothstep(0.5, 0.58, …)` biến đốm bóng thành mảng cắt cứng kiểu vẽ tay. Chỉ vật có `aM.y > 0` mới bóng.
- **`customProgramCacheKey`** bắt buộc khi source tiêm vào khác nhau (ở đây theo `sway`). Thiếu nó, three.js dùng lại program đã biên dịch của material khác và gió "biến mất" hoặc lan sang vật đứng yên.

---

## 5. Viền: inverted hull dày cố định theo pixel

Mỗi mesh có một mesh con dùng cùng hình học, vẽ `BackSide`, đẩy đỉnh ra theo normal **trong clip space** để độ dày tính bằng pixel màn hình.

```glsl
// html:378 — HULL_VS (rút gọn phần khai báo)
vec4 clip = projectionMatrix * modelViewMatrix * p;
vec3 vn = normalize(normalMatrix * n);
clip.xy += normalize(vn.xy + 1e-5) * uThick * aW * uPx * 2.0 / uRes * clip.w;
gl_Position = clip;
vC = mix(dot(color, color) < 1e-4 ? uColor : color * 0.3, uColor, uSolid);
```

```glsl
// html:395 — HULL_FS
varying vec3 vC;
void main() { gl_FragColor = vec4(vC, 1.0);
  #include <colorspace_fragment>
}
```

- `* clip.w` khử phép chia phối cảnh → nét không mảnh đi khi zoom xa.
- `2.0 / uRes` đổi pixel sang NDC; `uPx` (pixel ratio) giữ độ dày như nhau trên màn retina.
- Màu viền = **30% vertex color** (`color * 0.3`): mái đỏ có nét nâu đỏ, cỏ có nét xanh đậm. Đây là khác biệt lớn nhất so với viền đen thuần.
- `aW` (0.4–1, theo kích thước primitive, tính trong `Kit.add`) → cửa sổ, đinh tán có nét mảnh; khối lớn nét đậm.
- `#include <colorspace_fragment>` bắt buộc trong mọi `ShaderMaterial` tự viết, nếu không màu bị lệch so với phần còn lại.

**Hàn normal** (`hullGeo`, `html:411`): hộp có cạnh cứng sẽ có 3 normal khác nhau tại mỗi góc; đẩy theo chúng làm viền bị hở. Gộp normal của các đỉnh trùng vị trí (khoá theo vị trí làm tròn `* 400`) rồi chuẩn hoá. Kết quả cache trong `WeakMap` theo geometry gốc.

**Gắn viền** (`html:432`):

```js
function addOutline(mesh, w = 1, sway = 0) {
  const geo = hullGeo(mesh.geometry);
  let hull;
  if (mesh.isInstancedMesh) { hull = new THREE.InstancedMesh(geo, hullMat('normal', w, sway), mesh.count); hull.instanceMatrix = mesh.instanceMatrix; }
  else hull = new THREE.Mesh(geo, hullMat('normal', w, sway));
  hull.raycast = () => {}; hull.frustumCulled = false;
  hull.userData.hull = { w, sway };
  mesh.add(hull);
  return mesh;
}
```

- Hull của `InstancedMesh` **dùng chung** `instanceMatrix` → chỉ cập nhật một chỗ.
- `raycast = () => {}` để click không trúng viền; `frustumCulled = false` vì bounding sphere của hull nhỏ hơn phần đã đẩy ra.
- Trạng thái (thường / chọn / hợp lệ / sai chỗ) đổi bằng cách **hoán material** đã cache (`hullMat(state, w, sway)`, `setHull`), không clone. Viền chọn dày gấp đôi (3.4 px) và màu đặc (`uSolid = 1`).
- Nếu vật thể có gió lay, hull phải chạy **đúng công thức gió** của mesh chính (cùng `ph`, cùng biên độ), nếu không viền tách khỏi tán cây.

---

## 6. `Kit`: dựng công trình từ primitive có vertex color

Mỗi công trình là hàng chục primitive được gộp thành **1 mesh + 1 hull** = 2 draw call.

```js
// html:473 (rút gọn)
class Kit {
  // p = surface pattern id (PAT.*), sh = specular amount, ao = bake the bottom-dark / top-light gradient per primitive
  constructor() { this.geos = []; this.p = 0; this.sh = 0; this.ao = true; this.gr = [0.84, 1.06]; }
  add(geo, color, pos, rot, scl) { /* toNonIndexed → fixUV → gradient → transform → color/aM/aW */ }
  box(...)  rbox(...)  cyl(...)  cone(...)  sph(...)  torus(...)  prism(...)  poly(...)
  build() { const g = mergeGeometries(this.geos, false); /* ... */ return g; }
}
function meshOf(kit, w = 1) {
  const m = new THREE.Mesh(kit.build(), toonMat);
  m.castShadow = m.receiveShadow = true;
  return addOutline(m, w);
}
```

Trong `Kit.add` (`html:476`):

1. `toNonIndexed()` để mọi primitive cùng định dạng trước khi merge.
2. `fixUV` viết lại UV theo **đơn vị thế giới** (1 đơn vị = 1 ô): gạch trên tháp lớn và tháp nhỏ có cùng kích thước viên.
3. **Gradient đỉnh bake sẵn**: nhân màu theo chiều cao tương đối của primitive, từ `gr[0]` ở chân lên `gr[1]` ở đỉnh. Mặc định `[0.84, 1.06]`; tán lá `[0.66, 1.14]`; thân cây `[0.78, 1.04]`. Đây là AO giả không tốn pass nào.
4. Ghi `color`, `aM = [p, sh]`, `aW = clamp(0.35 + size * 0.9, 0.4, 1)`.

Cách viết một công trình (trích Town Hall, `html:932`): đặt `k.p` / `k.sh` như "bút" hiện hành, vẽ, rồi trả về 0.

```js
k.p = PAT.brick;
k.rbox(3.86, 0.14, 3.86, CL.stoneD, [0, 0.07, 0]); k.rbox(3.56, 0.16, 3.56, CL.stone, [0, 0.22, 0]);
// ...
k.p = PAT.tile; k.cone(2.82, 1.5, roof, [0, 2.65, 0], [0, Math.PI / 4, 0], null, 4); k.p = 0;
// ...
k.sh = 0.7; k.sph(0.13, gold, [0, 4.92, 0], null, null, 10, 8); k.sh = 0;
```

Quy tắc hình khối:

- Khối chính dùng `rbox` (`RoundedBoxGeometry`, bán kính 0.07) cho cảm giác "mập"; chỉ dùng `box` sắc cho dầm gỗ, chi tiết mảnh.
- Mái dốc bằng `prism` (`ExtrudeGeometry` từ tam giác); mái chóp bằng `cone` 4 cạnh xoay 45°.
- Nền đá dày, rộng hơn thân nhà; viền vàng chỉ xuất hiện khi nâng cấp (`lvl >= 2`) → đọc được cấp độ bằng mắt.
- Đá tự nhiên: `rockGeo` (`html:526`) đẩy đỉnh icosphere bằng nhiễu lượng giác **chỉ phụ thuộc vị trí** (đỉnh trùng vẫn khít) và ghi độ lõm vào thuộc tính `shade` để khe nứt tối hơn.

---

## 7. Họa tiết thủ tục với `fwidth`

`PAT_GLSL` (`html:313`) có 9 họa tiết: ván dọc, ván ngang, gạch, ngói vảy cá, rạ, đá cuội, lá/đá (value noise), sọc, vỏ cây. Chúng **nhân** vào `diffuseColor` nên màu gốc vẫn đến từ vertex color.

Mấu chốt chống răng cưa là hàm `jn` dùng `fwidth` làm độ rộng chuyển tiếp:

```glsl
float jn(float f, float w, float fw) { float d = min(f, 1.0 - f); return smoothstep(w - fw, w + fw + 1e-4, d); }
```

Ví dụ gạch so le:

```glsl
vec2 q = vec2(uv.x / 0.5, uv.y / 0.24); q.x += 0.5 * mod(floor(q.y), 2.0);
vec2 c = floor(q), f = fract(q), fw = fwidth(q);
m = mix(0.56, 1.0, jn(f.x, 0.05, fw.x) * jn(f.y, 0.09, fw.y)) * (0.88 + 0.22 * pH(c));
```

- `0.5 × 0.24` là kích thước viên gạch tính theo đơn vị thế giới (nhờ `fixUV`).
- `pH(c)` lệch sáng tối từng viên ±11% → không đều như máy in.
- Mạch vữa tối tới 0.56, không tới 0 → vẫn đọc là "màu tường", không thành lưới đen.

Họa tiết noise (lá, đá) dùng phép chiếu xiên liên tục `(x + 0.5z, y + 0.35z)` trong `fixUV` để cầu và nón không có đường nối hay điểm co ở cực.

---

## 8. Mặt đất, nước, bóng

### Mặt đất vẽ trên canvas (`html:630`)

Một `CanvasTexture` 48 px/ô, gán vào `MeshToonMaterial({ map, gradientMap })`. Thứ tự lớp:

1. Nền viền ngoài + đốm nhiễu mờ.
2. Ô cỏ bàn cờ hai tông (`grassA` / `grassB`) ±3% độ sáng, mỗi ô có vạch sáng trên và vạch tối dưới.
3. Đường đất: vẽ cùng polyline nhiều lần với độ rộng giảm dần: viền cỏ (alpha 0.32) → viền đất → lõi → dải sáng giữa (alpha 0.28); thêm 2 vệt bánh xe và ~4400 viên sỏi có bóng lệch.
4. Quảng trường lát đá bo góc, mỗi viên có highlight trên + bóng dưới.
5. ~2500 chùm cỏ (`quadraticCurveTo`), hoa 5 cánh, sỏi.

Bắt buộc: `tex.colorSpace = THREE.SRGBColorSpace` và `tex.anisotropy = renderer.capabilities.getMaxAnisotropy()` (camera xiên nên anisotropy quyết định độ nét ở xa).

### Nước toon (`html:733`)

`ShaderMaterial` riêng (nước không cần đèn):

- Độ sâu theo khoảng cách tới bờ, **lượng tử hoá 3 nấc** (`floor(... * 3.0) / 2.0`) giữa xanh ngọc `(0.29, 0.86, 0.84)` và xanh dương `(0.09, 0.47, 0.86)`.
- Sọc sóng: `smoothstep(0.94, 0.985, sin(...))` → vạch trắng mảnh, không phải gợn mượt.
- Vòng bọt trắng ở bờ, méo theo `sin(a * 9 + t)` và noise.
- Kết thúc bằng `#include <colorspace_fragment>`.

### Bóng phụ

- **Bóng mây** (`html:790`): 6 icosphere dẹt với `MeshBasicMaterial({ colorWrite: false, depthWrite: false })`, `castShadow = true`, trôi chậm. Vô hình nhưng vẫn đổ bóng vào shadow map.
- **Bóng tiếp xúc** (`html:798`): plane với gradient tròn trên canvas (`rgba(20,40,10,.5)` → 0), `transparent`, `depthWrite: false`, `renderOrder = 1`, đặt ở `y = 0.03` dưới mỗi công trình. Bóng xanh lá đậm thay vì đen để hợp với cỏ.

---

## 9. Thiên nhiên: instancing, tint, gió

```js
// html:1451
const im = new THREE.InstancedMesh(treeKit(kind).build(), kind === 3 ? toonMat : mat, list.length);
im.castShadow = im.receiveShadow = true; im.frustumCulled = false;
list.forEach(([x, z, sc, yaw], n) => {
  im.setMatrixAt(n, _m.compose(_v.set(x, 0, z), _q.setFromAxisAngle(UP, yaw), _s.set(sc, sc * rr(0.92, 1.12), sc)));
  im.setColorAt(n, _col.setRGB(1 + rr(-0.1, 0.08), 1 + rr(-0.08, 0.1), 1 + rr(-0.12, 0.04)));
});
addOutline(im, 1, kind === 3 ? 0 : 0.032);
```

- Mỗi loại (cây tròn gần, cây tròn xa, thông, bụi, đá) là **một** `InstancedMesh`; cây được dựng bằng chính `Kit`.
- `setColorAt` lệch ±10% mỗi kênh + scale Y ngẫu nhiên → rừng không lặp.
- LOD đơn giản: cây gần làng (< `HI_D`) dùng kit chi tiết (~1.000 tam giác, có gốc xoè, cành), rừng xa dùng kit rẻ (~390).
- Cây tròn = 4–6 quả cầu chồng nhau, mỗi quả một tông xanh; thông = 4 nón chồng, sáng dần lên ngọn.
- Gió: `swayMat(0.032)` (bước 4); biên độ nhân `max(y, 0)` nên gốc đứng yên, ngọn lay nhiều. Đá dùng `toonMat` không gió.

---

## 10. "Juice": chuyển động làm nên chất hoạt hình

| Hiệu ứng | Công thức | Vị trí |
| - | - | - |
| Nảy khi chọn | `w = sin(t*22) * exp(-t*7)`; `scale = (1 - 0.05w, 1 + 0.11w, 1 - 0.05w)` trong 0.8 s | `html:2345` |
| Hiện công trình | `scale = easeOutBack(t)` trong 0.55 s | `html:2344` |
| Cờ | `rotation.y = sin(t*3) * 0.3`, `scale.x = 1 + sin(t*5) * 0.06` | `html:971` |
| Khói | Cầu instanced bay lên, kích thước `sin(π·u)`, màu trắng → xanh nhạt | `html:1527` |
| Nhân vật đi | Chân ±0.7 rad, tay vung ngược pha, thân nhún `abs(sin(t*11))*0.06` | `html:1383` |

Squash & stretch giữ thể tích gần đúng (X, Z co khi Y giãn). Mọi giá trị dùng `exp` giảm chấn thay vì tween tuyến tính.

**Nhân vật gắn khớp** (`makeChar`, `html:1354`): đầu to (cầu 0.21 trên thân 0.42), mắt là 2 chấm đen, tóc là cầu dẹt. Thân, 2 chân, 2 tay là mesh riêng (mỗi cái qua `meshOf` với viền `0.5`), **pivot đặt ở khớp** (gốc toạ độ của kit tay ở vai, hình học kéo xuống `-y`), nên animation chỉ là xoay `rotation.x`.

---

## 11. Camera

```js
// html:1575
const rig = { x: 0, z: 1, dist: PARAMS.test ? 60 : 100, goal: 60, yaw: Math.PI / 4, goalYaw: Math.PI / 4, pitch: THREE.MathUtils.degToRad(50) };
```

- FOV 28°, pitch 50°, yaw 45°: gần isometric nhưng vẫn có phối cảnh nhẹ. FOV rộng hơn làm công trình méo và mất cảm giác "đồ chơi trên bàn".
- Pan bằng "nắm điểm trên mặt đất" (giao tia với mặt phẳng `y = 0`) nên điểm dưới con trỏ đứng yên. Zoom và xoay đều qua `damp(a, b, k, dt) = lerp(a, b, 1 - exp(-k*dt))`.
- Fog `72 → 160` khớp với khoảng `dist`: zoom xa thì rìa rừng tan vào haze.

---

## 12. Bảng núm chỉnh

| Núm | Giá trị gốc | Tăng → | Giảm → |
| - | - | - | - |
| Gradient map | `[104, 152, 200, 236, 255]` | Ít nấc (3): phẳng, kiểu cel | Nấc tối thấp hơn: tương phản gắt |
| Hemisphere ground | `0x8f9bd0` (tím lạnh) | Ấm hơn: cảnh hoàng hôn | Xám: bóng đục, mất chất |
| Độ dày viền | `1.7` px (chọn: `3.4`) | Nét đậm kiểu comic | Mảnh, gần low-poly |
| Màu viền | `color * 0.3` | `0.5`: viền nhạt, mềm | `0`: viền đen thuần, cứng |
| `aW` | `0.35 + size*0.9`, kẹp `[0.4, 1]` | Đều nét | Chênh nét lớn/nhỏ rõ hơn |
| Rim | `pow(…, 2.4) * 0.14` | Hệ số lớn: phát sáng viền kiểu "glow" | Mũ lớn: rim mảnh |
| Specular | `smoothstep(0.5, 0.58)`, mũ 48 | Khoảng hẹp hơn: đốm sắc hơn | Mũ nhỏ: đốm to |
| Gradient đỉnh | `[0.84, 1.06]`, lá `[0.66, 1.14]` | Khoảng rộng: khối nổi, AO đậm | Hẹp: phẳng |
| Gió | `0.032` | Lay mạnh, kiểu bão | Gần đứng yên |
| Fog | `72 → 160` | Xa hơn: thấy nhiều rừng | Gần: không khí dày |

---

## 13. Lỗi hay gặp (r186)

- `PCFSoftShadowMap` đã bị bỏ ở r186 → dùng `PCFShadowMap`.
- Cường độ đèn phải nhân `Math.PI` (đơn vị vật lý từ r155), nếu không cảnh tối.
- `ShaderMaterial` tự viết (hull, nước) phải có `#include <colorspace_fragment>`.
- `onBeforeCompile` với source khác nhau phải có `customProgramCacheKey`.
- `gradientMap` phải `NearestFilter` + `generateMipmaps = false`.
- `CanvasTexture` màu phải `SRGBColorSpace`; texture dữ liệu (gradient, mask) thì không.
- Lớp trong suốt (bóng tiếp xúc, lưới, hạt) cần `depthWrite: false` và `renderOrder` rõ ràng để không cắt nhau.
- Hull không hàn normal → hở ở góc hộp; hull không cùng công thức gió → viền tách khỏi tán cây.
- Render vào target nhỏ (ảnh Shop, `html:2092`): tạm đặt `uRes` = kích thước target và giảm `uPx`, nếu không viền dày gấp mấy lần.

---

## 14. Checklist dựng cảnh mới theo phong cách này

1. Renderer: `antialias`, không tone mapping, `PCFShadowMap`, fog = màu nền.
2. Đèn: Directional ấm + Hemisphere (trời xanh nhạt / đất tím lạnh), cường độ × π.
3. `gradientMap` 5 nấc + `makeToon()` có rim, specular, họa tiết.
4. Hull outline có hàn normal, dày theo pixel, màu 30% vertex color.
5. Mọi vật thể dựng bằng `Kit` → `meshOf` (1 mesh + 1 hull).
6. Mặt đất vẽ canvas; nước shader nấc; bóng tiếp xúc + bóng mây.
7. Cây/đá instanced, tint từng instance, gió dùng chung công thức với hull.
8. Thêm nảy, `easeOutBack`, cờ, khói.
9. Camera FOV 28°, pitch 50°, yaw 45°, pan/zoom giảm chấn.
10. Kiểm tra: tắt hết đèn trừ Hemisphere, cảnh vẫn phải đọc được khối nhờ gradient đỉnh và viền.
