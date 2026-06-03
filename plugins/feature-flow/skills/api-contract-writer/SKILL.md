---
name: api-contract-writer
description: "Đọc source code của một repo/service và viết API contract dạng Markdown ngắn gọn, chỉ chứa những gì caller cần biết. Dùng skill này bất cứ khi nào user gọi `/api-contract-writer`, hoặc nói các cụm như 'viết api docs', 'tạo api contract', 'document endpoints', 'viết tài liệu api', 'list các endpoint trong service X' — ngay cả khi user không nói rõ chữ 'skill'. Hoạt động bán-tự-động — liệt kê endpoint tìm được, chờ user xác nhận phạm vi, rồi mới sinh contract chi tiết."
---

# API Contract Writer

Tạo tài liệu API contract cho một service/module, output là file Markdown theo format ở `references/contract-format.md`.

**Triết lý**: contract mô tả những gì **caller quan sát được** — request shape, response shape, status codes, điều kiện lỗi. Không phải tài liệu thiết kế hệ thống. Người đọc là dev tích hợp với API, không phải dev maintain service.

## Ngôn ngữ

Mặc định tiếng Việt (mô tả, ghi chú); tên field/path/code giữ tiếng Anh. Nếu user yêu cầu ngôn ngữ khác (hoặc đang trao đổi bằng ngôn ngữ khác), viết theo ngôn ngữ đó.

## Workflow bán-tự-động (theo đúng thứ tự)

User chọn workflow này vì muốn **xác nhận scope trước khi viết chi tiết** — không tự ý sinh contract cho toàn bộ repo.

### Bước 1 — Làm rõ phạm vi

Nếu user chưa chỉ rõ service/module/folder, hỏi ngắn gọn:
- Service/module nào?
- Giới hạn router/file cụ thể không?
- File output đặt ở đâu? (mặc định gợi ý: `<service>/docs/api-contract.md`)

Nếu user đã chỉ rõ, bỏ qua.

### Bước 2 — Quét và liệt kê endpoint

Tìm tất cả HTTP endpoints trong scope. Phát hiện theo framework:

- **FastAPI / Starlette**: `@app.<method>`, `@router.<method>` (`get/post/put/patch/delete`), kết hợp `prefix=` của router để dựng full path.
- **Flask**: `@app.route(..., methods=[...])`, `@blueprint.route(...)`.
- **Express**: `app.<method>()`, `router.<method>()`.
- **Spring Boot**: `@GetMapping`, `@PostMapping`, `@RequestMapping`.
- **NestJS**: `@Get/@Post/@Put/@Delete` trong controller.
- Repo có `openapi.json` / `openapi.yaml` → ưu tiên đọc file đó.

Trình bảng tóm tắt:

```
Tìm thấy N endpoints trong <scope>:

| # | Method | Path                    | Handler          |
|---|--------|-------------------------|------------------|
| 1 | POST   | /api/v1/items           | items.create     |
| ...

Bạn muốn viết contract cho tất cả, hay chỉ một số endpoint?
```

**DỪNG LẠI chờ user xác nhận.** Không tự viết tiếp.

### Bước 3 — Đọc chi tiết và viết contract

Với mỗi endpoint đã chốt:

1. Đọc handler: parameters, request/response models (Pydantic/DTO), exceptions, auth dependencies.
2. Truy ngược schema được tham chiếu.
3. Phát hiện auth (dependency, middleware, decorator).
4. Phát hiện error responses thực sự có thể trả (`HTTPException`, exception handlers).

Viết theo format ở `references/contract-format.md` — không sáng tạo cấu trúc khác.

### Bước 4 — Lưu và báo cáo

Ghi file vào path đã thống nhất. Báo cáo: số endpoint đã document + các điểm không suy luận được từ code (để user bổ sung tay).

## Nguyên tắc nội dung

**Contract, không phải implementation.** Mô tả chỉ chứa behavior caller thấy được: idempotency, điều kiện trả từng status code, default values, giới hạn (max items, độ dài). KHÔNG nhắc tên công nghệ nội bộ (loại DB, message queue, cache, tên lock key, tên background job) — caller không cần và nó lộ chi tiết hệ thống. Ví dụ:

- ❌ "Dùng Redis lock chống upsert đồng thời, enqueue ARQ job `ingest_file`"
- ✅ "Upsert đồng thời cùng `(user_id, file_id)` → `409`. Nếu job xử lý trước đó còn đang chạy, trả thông tin job đó (`dedup_hit=true`)"

**Không dư thừa.**
- Endpoint không có path params / query params / body → **bỏ hẳn section đó**, không ghi placeholder "Không có".
- Lỗi chung toàn service (sai API key, validation schema mặc định của framework) → chỉ ghi **một lần ở Phụ lục** cuối file. Từng endpoint chỉ liệt kê lỗi đặc thù của nó (404, 409, 422 nghiệp vụ riêng).
- Mô tả endpoint: 1–3 câu. Đủ để hiểu khi nào gọi và nhận gì về.

**Suy luận từ code, không bịa.** Không xác định được:
- Kiểu field → ghi `unknown` + note `_(cần xác nhận)_`.
- Status code → chỉ liệt kê những gì code thực sự raise/return.
- Mô tả nghiệp vụ → đọc docstring/comment; không có thì mô tả ngắn theo tên handler, không sáng tác.

**Ưu tiên schema thật.** Body là model class → đọc model, liệt kê từng field với type, required, default, description.

**Generic.** Không đưa tên công ty/tổ chức, URL nội bộ thật, hay thông tin môi trường cụ thể vào contract — base URL dùng placeholder (`http://localhost:<port>` hoặc `https://api.example.com`) trừ khi user yêu cầu khác.

**Auth kế thừa.** Auth apply ở mức router → ghi một lần ở header file ("Tất cả endpoints yêu cầu ... trừ ghi chú khác"); từng endpoint chỉ ghi khi **khác mặc định**.

## Format output

Đọc `references/contract-format.md` trước khi viết. Tóm tắt:

1. **Header**: tên service, base URL (placeholder), auth chung.
2. **Mục lục**: bảng Method / Path / Mô tả ngắn, link anchor.
3. **Mỗi endpoint**: heading `## <số>. \`<METHOD> <path>\`` → Mô tả → Request (chỉ các bảng có nội dung) → Ví dụ curl → Response (mỗi status một sub-section, schema bảng khi field cần giải thích) → lỗi đặc thù.
4. **Phụ lục**: bảng lỗi chung toàn service.

Cột "Bắt buộc" dùng ✅/❌.
