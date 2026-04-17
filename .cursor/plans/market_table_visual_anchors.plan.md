# Market: visual anchor theo stall level (0–4)

## Bối cảnh code hiện tại

- **Số slot gameplay** đã đúng: [`GetTableStackCapacityForStallLevel`](Assets/Scripts/Config/MarketConfig.fcg) → 3 / 4 / 5; [`MarketService`](Assets/Scripts/Services/MarketService.fcg) dùng `_TableItemIDs[0 .. cap-1]`.
- **Visual bàn** hiện dùng offset cứng trong [`MarketVisual.UpdateTableDisplay`](Assets/Scripts/Market/MarketVisual.fcg).
- Prefab có 5 entity `SlotVisualAnchor` (0…4).

## Mapping logical slot → anchor index (data → vị trí visual)

| Stall level | cap | Logical 0 | 1 | 2 | 3 | 4 |
|-------------|-----|-----------|---|---|---|---|
| 1 | 3 | 0 | 2 | 4 | — | — |
| 2 | 4 | 0 | 1 | 2 | 4 | — |
| 3 | 5 | 0 | 1 | 2 | 3 | 4 |

Thêm hàm trong [`MarketConfig.fcg`](Assets/Scripts/Config/MarketConfig.fcg), ví dụ `GetTableVisualAnchorIndexForLogicalSlot(stallLevel int, logicalSlot int) int` (trả `-1` nếu không hợp lệ).

## Nguồn ref: Custom Component `MarketSlot` + hydrate một lần

**Không** thêm 5 field `entity<Entity>` trên graph và **không** leo parent/children để tìm anchor.

Bạn đã gắn component **MarketSlot** trên prefab Market. Theo [`ECATypeSetting.asset`](ProjectSettings/ECATypeSetting.asset), CC này có property **`VisualAnchor`** kiểu **`ListT_Entity`** (list entity) — thứ tự phần tử trong list = **anchor index 0…4** (trùng thứ tự `SlotVisualAnchor` trong scene).

### Triển khai trong [`MarketRuntimeData.fcg`](Assets/Scripts/Market/MarketRuntimeData.fcg)

- `import "EditorGenLib.fcc" as gen` (giống [`CropSlot.fcg`](Assets/Scripts/Farming/CropSlot.fcg)).
- Runtime: `_TableVisualAnchors List<entity<Entity>> = nil` (hoặc tên gọn `_SlotVisualAnchors`).
- Flag `_VisualAnchorsHydrated bool = false` (tuỳ chọn) để chỉ copy list **một lần**.
- `func HydrateMarketSlotRefs()`:
  - Cast entity chứa graph (chính `thisEntity` của `MarketRuntimeData`) sang `entity<gen.MarketSlot>` (tên type gen cần khớp codegen sau khi Studio sinh — thường là `MarketSlot`).
  - Đọc list từ `ser<gen.MarketSlot>.VisualAnchor` (đúng tên field trong EditorGenLib; nếu codegen dùng tên khác property ECA, chỉnh theo `gen`).
  - Gán vào `_TableVisualAnchors` (copy từng phần tử hoặc giữ reference list tùy API).
- Gọi `HydrateMarketSlotRefs()` từ `EnsureInit()` (hoặc `OnAwake` lần đầu) trước khi `MarketVisual` cần dùng.

**Lưu ý:** Sau khi project build / codegen cập nhật `EditorGenLib.fcc`, kiểm tra tên chính xác: `gen.MarketSlot` và tên property list (`VisualAnchor` vs alias khác).

### [`MarketVisual.UpdateTableDisplay`](Assets/Scripts/Market/MarketVisual.fcg)

1. `market<marketData>.EnsureInit()` (đảm bảo đã hydrate list).
2. `stallLv` + `cap` như plan trước.
3. Với mỗi logical `slot` có đồ: `anchorIdx = GetTableVisualAnchorIndexForLogicalSlot(stallLv, slot)`.
4. Lấy entity anchor: nếu `_TableVisualAnchors != nil` và `anchorIdx < list.Length(_TableVisualAnchors)` → dùng phần tử đó; `mdl<Transform>.Position = anchor<Transform>.Position` (giống [`CropSlot.GetSpawnWorldPosition`](Assets/Scripts/Farming/CropSlot.fcg)).
5. Nếu list nil / index lệch / thiếu phần tử: fallback `GetTableSlotOffset(anchorIdx)` + `RotateOffset` như hiện tại.

**Góc xoay (tuỳ chọn):** copy `Rotation` từ anchor nếu cần khớp hướng kệ.

## Studio

- Trên entity gắn `MarketRuntimeData` + CC **MarketSlot**: kéo 5 entity `SlotVisualAnchor` vào list **VisualAnchor** đúng thứ tự 0→4.

## Hành vi upgrade stall (nhắc ngắn)

Khi đổi level, mapping logical → anchor đổi — đồ trên bàn có thể nhảy vị trí. Chấp nhận trừ khi sau này lưu visual slot riêng.

## File chạm tới

- [`Assets/Scripts/Config/MarketConfig.fcg`](Assets/Scripts/Config/MarketConfig.fcg) — hàm mapping.
- [`Assets/Scripts/Market/MarketRuntimeData.fcg`](Assets/Scripts/Market/MarketRuntimeData.fcg) — list runtime + `HydrateMarketSlotRefs()` từ `gen.MarketSlot`.
- [`Assets/Scripts/Market/MarketVisual.fcg`](Assets/Scripts/Market/MarketVisual.fcg) — đọc list đã hydrate + fallback.

```mermaid
flowchart LR
  CC[MarketSlot CC VisualAnchor list]
  H[HydrateMarketSlotRefs once]
  R[_TableVisualAnchors in MarketRuntimeData]
  V[MarketVisual positions by anchorIdx]
  CC --> H --> R --> V
```

## Todos

- [ ] `GetTableVisualAnchorIndexForLogicalSlot` trong MarketConfig.fcg
- [ ] MarketRuntimeData: list + hydrate từ `gen.MarketSlot` (property list entity)
- [ ] MarketVisual: dùng list + mapping; fallback offset
- [ ] Studio: list VisualAnchor trên MarketSlot đủ 5 entity đúng thứ tự
