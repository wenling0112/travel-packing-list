# 旅遊行李清單管理系統(Travel Packing List) — RESTful API Design

本文件依 [Requirements](01-requirements.md) 定義六條 API，完整介面契約見 [OpenAPI 3.1.0](03-openapi.yaml)。本階段為 **Design Only（尚未實作）**，不提供可連線服務。

## 1. Resources 與共通約定

| Resource | 意義 |
| --- | --- |
| Trip（`/trips`） | 一次旅行的條件與所屬清單 |
| PackingItem（`/trips/{tripId}/items`） | 指定旅行的行李項目 |
| Progress（`/trips/{tripId}/progress`） | 由目前清單計算的進度表示，不是新 Entity |

- Request／Response 使用 JSON（`application/json`）；DELETE 成功無回應內容。
- `tripId`、`itemId` 為 UUID。格式不合法回傳 HTTP 400；格式合法但找不到資源回傳 HTTP 404。
- API 使用 `tripId`、`itemId` 表示識別碼，分別對應 Entity 的 `Trip.id`、`PackingItem.id`；項目中的 `tripId` 對應 `PackingItem.trip_id`。其餘業務欄位沿用需求命名，例如 `is_packed`。
- Request Schema 不接受未知欄位或 `null`；不做字串與數字／Boolean 之間的自動轉型。
- 名稱及單位在儲存前去除前後空白，去除後不可空白；同名依去除前後空白後完全相同比對，不做同義詞或大小寫轉換。
- GET 回傳目前完整清單，不分頁，也不保證項目順序。計畫存在但項目全數刪除時，回傳 HTTP 200 與空陣列。
- 不設登入或授權機制；項目所屬計畫檢查屬資料一致性驗證，非使用者權限驗證。

## 2. API 總覽與 Requirement–API Mapping

| Method | Path | 用途／Resource | 成功 | 失敗 | 對應需求／規則 |
| --- | --- | --- | --- | --- | --- |
| POST | `/trips` | 建立 Trip 與初始 PackingItem | 201 | 400、500 | FR-01、FR-02；BR-01～BR-07、BR-10、BR-11 |
| GET | `/trips/{tripId}/items` | 查詢 PackingItem 集合 | 200 | 400、404、500 | FR-03；BR-14（計畫存在） |
| POST | `/trips/{tripId}/items` | 手動新增 PackingItem | 201 | 400、404、500 | FR-05；BR-07、BR-08、BR-12～BR-14 |
| PATCH | `/trips/{tripId}/items/{itemId}` | 修改內容或準備狀態 | 200 | 400、404、500 | FR-03、FR-06；BR-08、BR-12～BR-14 |
| DELETE | `/trips/{tripId}/items/{itemId}` | 刪除 PackingItem | 204 | 400、404、500 | FR-07；BR-08、BR-14 |
| GET | `/trips/{tripId}/progress` | 查詢計算後的 Progress | 200 | 400、404、500 | FR-04；BR-08 |

BR-09 維持不變：不提供修改 Trip 條件或重新推薦的 API。總計六個 Operations、四個不同 Path，維持 Trip、PackingItem 兩個 Entity。

## 3. POST /trips：建立旅遊計畫與推薦清單

Request Body：`CreateTripRequest`，四項皆必填。

| 欄位 | 型別 | 規則 |
| --- | --- | --- |
| days | integer | 1～30 |
| climate | string | WARM、COLD、RAINY 擇一 |
| activity | string | SIGHTSEEING、BEACH、HIKING 擇一 |
| region | string | DOMESTIC、INTERNATIONAL 擇一 |

```json
{"days":5,"climate":"COLD","activity":"HIKING","region":"INTERNATIONAL"}
```

HTTP 201 回傳 `CreateTripResponse`：`trip`（`Trip`）及 `items`（`PackingItem[]`）。Trip 包含 `tripId` 及原四項條件，項目均歸屬該計畫、數量依推薦規則計算、`is_packed` 初始為 `false`。OpenAPI 提供國內與國外的完整建立回應範例。

初始推薦依 BR-04、BR-05、BR-11，使用 BR-06 去重並取最大數量。國內包含身分證件、健保卡；國外包含護照、簽證確認、旅行轉接插頭；共同包含充電器。簽證確認須提醒自行確認需求，不代表系統判定簽證必要性。不呼叫外部 API。

Trip 與全部初始項目須作為一次完整建立操作；全部成功才回傳 201。處理失敗回傳 500，不保留只有 Trip 或部分項目的結果。此為未來實作的原子性要求，本次不建立資料庫。

缺少 region、非法 Enum、錯誤型別、days 超出範圍、未知欄位或無法解析的 JSON 皆回傳 400。

## 4. GET /trips/{tripId}/items：查詢目前清單

Path Parameter：必填 `tripId`（UUID），無 Request Body。

HTTP 200 回傳 `ItemListResponse`：`tripId` 與完整 `items`。每個項目包含 `itemId`、`tripId`、`name`、`quantity`、`unit`、`reason`、`is_packed`。`quantity` 為目前數量，`reason` 為原始推薦或新增依據。

計畫存在且無項目時：

```json
{"tripId":"11111111-1111-4111-8111-111111111111","items":[]}
```

UUID 格式不合法回傳 400；計畫不存在回傳 404。查詢不重新推薦，也不補回已刪除項目。

## 5. POST /trips/{tripId}/items：手動新增

Path Parameter：必填 `tripId`（UUID）。Request Body：`CreatePackingItemRequest`。

| 欄位 | 型別 | 規則 |
| --- | --- | --- |
| name | string | 必填，去除前後空白後非空，不得與同計畫項目同名 |
| quantity | integer | 必填，至少 1 |
| unit | string | 必填，去除前後空白後非空 |

```json
{"name":"個人藥品","quantity":1,"unit":"份"}
```

HTTP 201 直接回傳 `PackingItem`，由系統產生 `itemId`，設定指定 `tripId`、`reason = 使用者自行新增`、`is_packed = false`。新增後查詢清單與進度即須反映結果。

缺漏、不合法值、同名或提供 `id`、`itemId`、`trip_id`、`tripId`、`reason`、`is_packed` 等非允許欄位皆回傳 400，不自動合併。計畫不存在回傳 404。

## 6. PATCH /trips/{tripId}/items/{itemId}：修改內容或狀態

Path Parameters：必填 `tripId`、`itemId`（UUID）。Request Body：`UpdatePackingItemRequest`，至少提供以下一個欄位：

| 欄位 | 型別 | 規則 |
| --- | --- | --- |
| name | string | 去除前後空白後非空，不與另一項目同名 |
| quantity | integer | 至少 1，可覆寫原建議數量 |
| unit | string | 去除前後空白後非空 |
| is_packed | boolean | 僅接受 true、false |

```json
{"quantity":4,"is_packed":true}
```

HTTP 200 直接回傳更新後的 `PackingItem`。只更新提供的欄位；未提供的欄位與原始原因保持不變。內容修改不自動重設準備狀態，系統也不重新套用推薦覆寫修改結果。這是本專案的 JSON 部分更新格式，不採用 JSON Patch 操作陣列。

空物件、未知／不可寫欄位、null、不合法型別或值、與另一項目同名皆回傳 400，整筆修改不生效。判斷同名時排除自身。計畫、項目不存在或項目不屬於指定計畫皆回傳 404。

## 7. DELETE /trips/{tripId}/items/{itemId}：刪除項目

Path Parameters：必填 `tripId`、`itemId`（UUID），無 Request Body。

系統推薦與手動新增項目皆可刪除，包括護照等文件。成功回傳 HTTP 204，不帶 JSON 或其他回應內容。後續清單及進度查詢排除該項目，不自動補回，也不提供復原。

UUID 格式不合法回傳 400。計畫、項目不存在、項目不屬於指定計畫或重複刪除已不存在的項目，皆回傳 404。刪除其他計畫項目的請求不得影響該項目。

## 8. GET /trips/{tripId}/progress：查詢準備進度

Path Parameter：必填 `tripId`（UUID），無 Request Body。HTTP 200 回傳 `ProgressResponse`：

| 欄位 | 型別 | 說明 |
| --- | --- | --- |
| tripId | string（UUID） | 所屬計畫 |
| total_count | integer ≥ 0 | 目前全部項目數 |
| packed_count | integer ≥ 0 | 已準備項目數 |
| unpacked_count | integer ≥ 0 | 未準備項目數 |
| completion_rate | number，0～100 | 完成率百分比，70 表示 70% |
| unpacked_items | PackingItem[] | 目前 is_packed = false 的完整項目 |

`total_count = packed_count + unpacked_count`，`unpacked_count = unpacked_items.length`。完成率為 `packed_count / total_count × 100`；空清單時為 0。以項目數計算，不依 quantity 加權，不規定固定顯示小數位數。

每次查詢依目前清單產生一致的計數與未準備清單。新增未準備項目後分母增加，刪除項目後移除相應計數；僅改名稱、數量或單位不影響完成率。

UUID 格式不合法回傳 400；計畫不存在回傳 404；計畫存在但空清單回傳 200，所有計數及完成率為 0、`unpacked_items` 為空陣列。

## 9. 統一錯誤回應

非成功回應使用 `Error` Schema：

```json
{"code":"INVALID_REQUEST","message":"region 必填，僅接受 DOMESTIC 或 INTERNATIONAL"}
```

| HTTP Status | code | 情境 |
| --- | --- | --- |
| 400 | INVALID_REQUEST | JSON、UUID、必填欄位、型別、範圍、Enum 或不可寫欄位不合法 |
| 400 | DUPLICATE_ITEM_NAME | 手動新增或修改後與同計畫另一項目同名 |
| 404 | TRIP_NOT_FOUND | 指定計畫不存在 |
| 404 | ITEM_NOT_FOUND | 項目不存在或不屬於指定計畫 |
| 500 | INTERNAL_ERROR | 非預期處理失敗，不回傳內部例外細節 |

`message` 為可讀說明，不作為固定字串契約。請求驗證失敗不改動資料；建立計畫及 PATCH 分別遵守完整建立與整筆更新的要求。Schema 可驗證欄位形狀，跨項目同名、所屬計畫、推薦計算與進度一致性仍須由未來 Service 層依業務規則處理。

## 10. 契約與驗證

本文件與 OpenAPI 共用相同欄位及狀態碼。OpenAPI 使用獨立的 Create／Update Request Schema，避免將回應中的系統欄位誤作可寫欄位；回應中的 Trip、PackingItem、Progress 為資料表示，不增加 Entity。

OpenAPI 格式依 [OpenAPI Specification 3.1.0](https://spec.openapis.org/oas/v3.1.0.html)。可將 [03-openapi.yaml](03-openapi.yaml) 匯入 Swagger Editor 檢視；本階段的文件檢查不代表已執行 API 或系統測試。

本次已完成本地 YAML 解析、JSON Schema、所有本地參照與範例檢查，並核對推薦與進度資料、不合法輸入案例。尚未透過 Swagger Editor 或完整 OpenAPI 規格驗證器檢查。
