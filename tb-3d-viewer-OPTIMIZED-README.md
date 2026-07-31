# tb-3d-viewer-extension.optimized.js

Bản tối ưu của `tb-3d-viewer-extension.js`. Giữ nguyên toàn bộ phần Angular đã biên dịch
(component/template/form settings), chỉ sửa phần **logic** nên nạp vào ThingsBoard y hệt bản cũ.

## 1. Sửa lỗi Fill-level (nhiều entity)

**Trước:** `updateLevelMesh()` chỉ lấy **giá trị level đầu tiên** trong `ctx.data` và **mesh đầu tiên**
khớp `levelMeshId` → chỉ `hinhtru.001` phản ứng, và level chỉ theo `hinhtru.001`.

**Sau:** lặp qua **từng entity**, dùng `entity.level` riêng và `entity.meshId` riêng để tìm đúng
mesh của entity đó rồi scale. 3 entity `hinhtru.001/.002/.003` giờ được điều chỉnh độc lập theo
level của từng device.

Quy tắc tìm mesh level cho mỗi entity (`findLevelMeshForEntity`):
1. Lấy mesh gốc của entity qua `meshId`.
2. Nếu tên mesh gốc chứa `Level mesh id` (vd meshId=`hinhtru.001`, levelMeshId=`hinhtru`) → scale chính mesh đó.
3. Nếu không, tìm mesh con của nó có tên chứa `Level mesh id` (vd `Grain_Volume` trong từng silo).
4. Fallback: tìm mesh có tên chứa **cả** id của entity **và** `Level mesh id`.

### Chọn trục scale — `levelAxis`
Mặc định scale trục **Y** (đúng với model của skill silo-glb-builder). Nếu model của bạn dùng trục
khác, thêm `levelAxis` vào **Advanced settings (JSON)** của widget:

```json
{ "levelAxis": "z" }   // "x" | "y" | "z"
```

> Lưu ý: mesh phải có gốc (origin) ở **đáy** khối để fill từ dưới lên. Nếu origin ở tâm, khối sẽ co về giữa.

## 2. Chạy offline / local (không cần Internet)

Bản gốc tải `three`, `three-stdlib` từ `esm.sh` và DRACO decoder từ `gstatic.com` khi chạy → hỏng khi offline.

Có 2 cách chạy offline:

### Cách A — Nạp sẵn thư viện vào `window` (offline hoàn toàn, không dùng `import()`)
Trong `index.html`/resource của ThingsBoard, nạp trước rồi gán:
```html
<script type="module">
  import * as THREE from '/local/three.module.js';
  import * as STDLIB from '/local/three-stdlib.module.js';
  window.TB3D_THREE = THREE;
  window.TB3D_STDLIB = STDLIB;
  window.TB3D_CONFIG = { dracoDecoderPath: '/local/draco/' };
</script>
```
Widget tự phát hiện `window.TB3D_THREE`/`window.TB3D_STDLIB` và **bỏ qua CDN hoàn toàn**.

### Cách B — Trỏ URL sang server nội bộ
```html
<script>
  window.TB3D_CONFIG = {
    threeUrl:  '/local/three.module.js',
    stdlibUrl: '/local/three-stdlib.module.js',
    dracoDecoderPath: '/local/draco/'
  };
</script>
```

Model GLB: đặt **Model source = ThingsBoard resource** (nạp qua `/api/resource/{id}/download`) hoặc URL nội bộ.

## 3. Các tối ưu khác đã áp dụng
- **Render theo yêu cầu** (on-demand): chỉ vẽ khi có tương tác/thay đổi thay vì mỗi frame → giảm mạnh CPU/GPU/pin khi cảnh đứng yên. Nếu thấy cảnh không tự cập nhật, đặt `window.TB3D_CONFIG={continuousRender:true}` để quay lại vẽ liên tục.
- **Giới hạn pixel ratio** ≤ 2 (`maxPixelRatio`) → nhẹ hơn nhiều trên màn 4K/Retina.
- **Giải phóng tài nguyên**: `PMREMGenerator`, `DRACOLoader` (worker), và geometry/material/texture khi huỷ widget → chống rò rỉ VRAM.
- **Bắt lỗi tải model**: hiện thông báo trong widget thay vì màn hình trống.
- **Tooltip an toàn với đa widget**: nút `#action`/`#close` được tìm trong đúng tooltip của widget (`querySelector` scoped), và listener chỉ gắn ở nhánh tooltip (trước đây mọi lần tính màu/label đều schedule `setTimeout` tìm id toàn cục).

## Còn tồn đọng (khuyến nghị, chưa sửa để tránh rủi ro)
- `levelAxis` chưa có ô nhập trong form settings (phải sửa template Angular đã biên dịch). Dùng Advanced JSON.
- `meshType` rỗng sẽ khớp mọi mesh (`includes("")`), nên luôn đặt giá trị.
- Phím `p`/`P` toàn cục vẫn log vị trí camera ra console (tiện debug, vô hại).
