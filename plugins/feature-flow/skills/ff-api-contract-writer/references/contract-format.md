# Format chuẩn cho API Contract Markdown

Template **bắt buộc** cho output của skill `ff-api-contract-writer`. Sao chép cấu trúc và điền vào — không tự sáng tạo cấu trúc khác. Ví dụ dưới viết tiếng Việt; nếu user yêu cầu ngôn ngữ khác, dịch phần mô tả/heading phụ ("Mục lục" → "Table of Contents", v.v.) nhưng giữ nguyên cấu trúc.

---

## Template đầy đủ

```markdown
# API Contract — <Tên Service>

> **Base URL**: `http://localhost:<port>` (placeholder — thay theo môi trường)
> **Authentication**: <auth mặc định, ví dụ: API Key qua header `X-API-Key`, áp dụng mọi endpoint trừ ghi chú khác>

## Mục lục

| # | Method | Path | Mô tả ngắn |
|---|--------|------|------------|
| 1 | `POST` | [`/api/v1/items`](#1-post-apiv1items) | Tạo item, đưa vào pipeline xử lý nền |
| 2 | `GET`  | [`/api/v1/items/{item_id}`](#2-get-apiv1itemsitem_id) | Lấy trạng thái item |

---

## 1. `POST /api/v1/items`

**Mô tả**: Tạo (hoặc cập nhật) item và đưa vào xử lý nền. Idempotent theo `(user_id, item_id)` — nếu job xử lý trước đó còn đang chạy, trả thông tin job đó (`dedup_hit=true`) thay vì tạo mới.

### Request

#### Body (`application/json`)

| Field | Kiểu | Bắt buộc | Mặc định | Mô tả |
|-------|------|----------|----------|-------|
| `item_id` | `string` | ✅ | — | ID item (client cung cấp) |
| `user_id` | `string` | ✅ | — | ID người dùng sở hữu |
| `name` | `string` | ✅ | — | Tên hiển thị |
| `group_id` | `string` | ❌ | `"default"` | Nhóm chứa item. Độ dài 1–128 ký tự |

#### Ví dụ

```bash
curl -X POST "http://localhost:8000/api/v1/items" \
  -H "X-API-Key: <your-key>" \
  -H "Content-Type: application/json" \
  -d '{"item_id": "it_001", "user_id": "u_123", "name": "Item A"}'
```

### Response

#### `202 Accepted` — Đã nhận và enqueue (hoặc dedup hit)

```json
{
  "item_id": "it_001",
  "status": "pending",
  "job_id": "trace_abc123",
  "dedup_hit": false
}
```

| Field | Kiểu | Mô tả |
|-------|------|-------|
| `status` | `string` | `pending` / `processing` / `ready` / `failed` |
| `job_id` | `string` | ID job tracking. Dedup hit → giữ `job_id` của job đang chạy |
| `dedup_hit` | `boolean` | `true` khi job cũ còn chạy, không tạo job mới |

#### `409 Conflict` — Đang có request ghi đồng thời cùng item

```json
{
  "detail": "Concurrent upsert in progress for this item, retry shortly"
}
```

---

## 2. `GET /api/v1/items/{item_id}`

**Mô tả**: Lấy trạng thái hiện tại của một item.

### Request

#### Path params

| Tên | Kiểu | Bắt buộc | Mô tả |
|-----|------|----------|-------|
| `item_id` | `string` | ✅ | ID item |

#### Query params

| Tên | Kiểu | Bắt buộc | Mặc định | Mô tả |
|-----|------|----------|----------|-------|
| `user_id` | `string` | ✅ | — | ID người sở hữu (scope truy vấn) |

#### Ví dụ

```bash
curl -X GET "http://localhost:8000/api/v1/items/it_001?user_id=u_123" \
  -H "X-API-Key: <your-key>"
```

### Response

#### `200 OK`

```json
{
  "item_id": "it_001",
  "status": "ready",
  "error": null,
  "updated_at": "2026-01-01T10:00:30Z"
}
```

| Field | Kiểu | Mô tả |
|-------|------|-------|
| `status` | `string` | `pending` / `processing` / `ready` / `failed` |
| `error` | `string \| null` | Thông báo lỗi khi `status=failed`, ngược lại `null` |
| `updated_at` | `string (ISO 8601)` | Thời điểm cập nhật gần nhất |

#### `404 Not Found`

```json
{
  "detail": "Item not found"
}
```

---

## Phụ lục — Lỗi chung

Áp dụng cho mọi endpoint (trừ ghi chú khác), không lặp lại ở từng endpoint:

| Status | Khi nào | Shape |
|--------|---------|-------|
| `401 Unauthorized` | Thiếu hoặc sai header `X-API-Key` | `{"detail": "Invalid API key"}` |
| `422 Unprocessable Entity` | Body/query/path không qua validation schema | `{"detail": [{"type": "...", "loc": [...], "msg": "...", "input": ...}]}` |
| `500 Internal Server Error` | Lỗi không bắt được trong handler | — |
```

---

## Quy tắc trình bày

1. **Heading endpoint**: `## <số>. \`<METHOD> <path>\`` — số thứ tự để link mục lục.
2. **Cột "Bắt buộc"**: chỉ `✅` / `❌`.
3. **Section trống**: endpoint không có Path params / Query params / Body / Headers riêng → **bỏ hẳn section**, không ghi placeholder. Header auth chung đã khai báo đầu file → không lặp bảng Headers ở từng endpoint.
4. **Status code**: mỗi status một sub-heading `#### \`<code>\` <reason> — <điều kiện ngắn>`. Chỉ liệt kê lỗi **đặc thù của endpoint** (404, 409, 422 nghiệp vụ riêng kèm điều kiện cụ thể); lỗi chung → Phụ lục.
5. **Bảng field response**: chỉ thêm khi field cần giải thích (enum values, điều kiện null, ý nghĩa không hiển nhiên). Response mà example JSON tự giải thích đủ → bỏ bảng. Field hiển nhiên (`item_id` = "ID item") không cần dòng riêng nếu cả bảng chỉ toàn dòng như vậy.
6. **JSON example**: phải parse được. Giá trị không xác định → placeholder rõ ràng `"<item_id>"`.
7. **Mô tả**: 1–3 câu, behavior caller thấy được. Không nhắc công nghệ nội bộ (DB, queue, cache, lock).
8. **Field suy luận không chắc**: note `_(suy luận từ code, cần xác nhận)_` ngay sau mô tả.
