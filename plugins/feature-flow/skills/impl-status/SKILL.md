---
name: impl-status
description: "Sinh và duy trì file HTML trạng thái sống cho task implement tại `docs/features/<feature>/implementation_status.html`. CHỈ trigger khi user gọi rõ ràng: `/impl-status`, `impl-status`, 'dùng skill impl-status', 'bật status file', 'maintain status file', 'theo dõi implementation status', hoặc 'resume impl-status <feature>' / 'load status <feature>' / 'tiếp tục feature X' để load lại từ proposal + implementation_status.html đã có trong chat mới. KHÔNG tự động trigger ngay cả khi user yêu cầu implement feature lớn. Sau khi user kích hoạt, skill ở trạng thái active đến hết session hoặc khi user nói 'stop status file' / 'skip status'. File ghi tiến độ, decisions ngoài thiết kế, compromises, bugs trong khi code, file thay đổi và rollout checklist. Hỗ trợ sinh proposal.html trước nếu user yêu cầu thiết kế trước; implementation_status.html chỉ sinh khi user xác nhận bắt đầu implement. Cũng hỗ trợ resume context từ session trước bằng cách đọc lại 2 file HTML đó."
---

# impl-status — Live Implementation Status HTML

Sinh + duy trì file HTML tracker khi user implement task đa bước. File là **artifact giao tiếp** — ghi quyết định, thoả hiệp, bugs để user đọc nhanh và challenge điểm không đồng ý.

## Khi nào skill này áp dụng

**Chỉ trigger khi user kích hoạt rõ ràng**:
- `/impl-status`
- "dùng skill impl-status", "bật impl-status"
- "bật status file", "maintain status file"
- "theo dõi implementation status"
- Hoặc user nhắc tới skill này theo cách tương đương

**Không tự động trigger** ngay cả khi user yêu cầu implement feature lớn / refactor / migration. Đây là yêu cầu rõ ràng từ owner skill: skill này phải opt-in mỗi session.

Sau khi user kích hoạt, skill **active đến hết session** (hoặc đến khi user pause). Mọi task implement tiếp theo trong session đều áp dụng workflow này — không cần user nhắc lại.

**Tạm dừng** (giữ file hiện có, không update tiếp): "stop status file", "skip status", "khỏi cần status", "đừng update status nữa". **Resume** khi user yêu cầu lại.

## Nguyên tắc

**File HTML là live document, không phải log.** Ngắn gọn, organized. Update vì user cần biết — không update vì "đã làm xong bước 4".

**Ngôn ngữ theo user.** VI prompt → VI HTML. EN prompt → EN HTML.

**Source-of-truth là file HTML.** Đọc trước khi update để giữ liên tục. Không rewrite trừ khi cấu trúc thay đổi lớn.

## Workflow

### Bước -1 — Resume từ chat trước (nếu user yêu cầu)

Khi user nói "resume impl-status <feature>", "load status <feature>", "tiếp tục feature X", "đọc lại status file", "tiếp tục từ session trước":

1. **Locate folder.** Tìm `docs/features/<feature>/` trong working directory. Nếu user không nói rõ tên feature:
   - List `docs/features/*/` (Bash hoặc Glob).
   - Nếu có 1 folder → dùng luôn.
   - Nhiều folder → hỏi user chọn (bullet list các tên feature).
   - Không có folder → báo lỗi: "Không tìm thấy `docs/features/<feature>/`. Skill chưa từng chạy cho feature này — bật skill mới bằng `/impl-status`."

2. **Đọc file theo thứ tự**:
   - `proposal.html` (nếu tồn tại) — hiểu design intent + decisions ban đầu.
   - `implementation_status.html` (nếu tồn tại) — pickup tiến độ + decisions ngoài thiết kế + bugs đã catch + files đã đổi + rollout checklist.
   - `README.md` (nếu tồn tại) — index/summary.

3. **Reconstruct context.** Sau khi đọc, trình bày tóm tắt ngắn cho user (≤10 dòng):
   - Tiến độ hiện tại (bước done / in_progress / todo).
   - Decisions chính đã chốt.
   - Bugs đã fix.
   - Rollout step nào còn pending.

4. **Verify still-valid.** Memory check: các file/path/symbol nhắc trong status có thể đã đổi giữa các session. Trước khi suggest tiếp tục bước nào, grep / read file thật để confirm. Nếu phát hiện skew (file đã rename/xoá) → cập nhật status file ngay (section mới "Drift detected" hoặc update entry liên quan).

5. **Đợi user chỉ đạo** việc tiếp theo (resume implement bước nào, edit decision nào, finalize, vv). Không tự động code.

Sau bước này, skill chuyển sang chế độ active bình thường (Bước 3 — update khi có event).

### Bước 0 — Xác định feature name

Suy luận `snake_case` / `kebab-case` từ:
1. Prompt user (vd: "thêm project_id" → `project_id`).
2. Git branch (vd: `develop-feature-project_id` → `project_id`).
3. Nội dung proposal nếu đã có.

Không hỏi user trừ khi 3 nguồn đều rỗng/ambiguous.

### Bước 1 — Proposal (khi user yêu cầu thiết kế trước)

Khi user nói "đánh giá giải pháp", "viết đề xuất", "thiết kế trước", "viết proposal HTML":

#### 1a — Hỏi lại để clear ý (BẮT BUỘC trước khi sinh proposal)

Proposal chốt design intent — sai giả định ở đây sẽ lan xuống status file + code thật. Rẻ hơn nhiều nếu hỏi trước. Vì vậy **luôn** đặt **3–5 câu hỏi** làm rõ trước khi viết `proposal.html`.

**Chọn số câu hỏi theo độ chi tiết user đã mô tả:**
- Mô tả kỹ, rõ scope + data model + API + edge case → **3** câu (chỉ hỏi phần thật sự mơ hồ).
- Mô tả trung bình → **4** câu.
- Mô tả sơ sài / một câu / nhiều chỗ hổng → **5** câu.

Đừng hỏi cho đủ số — hỏi đúng chỗ còn mơ hồ. Nếu mọi thứ đã rõ tới mức không nghĩ ra được 3 câu đáng hỏi, vẫn phải hỏi ≥3: chuyển sang xác nhận giả định ("Tôi đang hiểu X, đúng không?", "Scope có bao gồm Y không?", "Có ràng buộc Z nào không?").

**Ưu tiên hỏi** những thứ làm proposal đi sai hướng nếu đoán nhầm: scope (làm gì / không làm gì), data model + nguồn dữ liệu, hành vi edge case, ràng buộc tương thích ngược / migration, ai là consumer của API.

**Ngữ cảnh session trước là LOW TRUST.** Có thể tái dùng để *gợi ý* câu trả lời, nhưng không coi là chốt. Nếu yêu cầu hiện tại **mâu thuẫn** với điều đã nói trước đó trong session (vd: trước nói dùng Postgres, giờ ngầm định Mongo) → **phải hỏi lại để giải quyết mâu thuẫn**, nêu rõ hai phía conflict, đừng tự chọn.

Dùng tool hỏi nhiều lựa chọn (AskUserQuestion) khi câu hỏi có phương án rời rạc; hỏi open-ended khi cần mô tả tự do. Sau khi user trả lời, mới sinh proposal. Nếu câu trả lời lại mở ra mơ hồ mới đáng kể, hỏi thêm 1 vòng ngắn — đừng kéo dài vô hạn.

#### 1b — Sinh proposal

Sinh `docs/features/<feature>/proposal.html` với:

- Mục tiêu (1 đoạn).
- Thay đổi data model (table).
- API surface (table: method, path, mô tả).
- Luồng chi tiết cho API mới (numbered steps).
- Đánh giá: Ưu / Rủi ro / Alternative đã loại.
- Checklist triển khai high-level.

Template: `assets/proposal_template.html`. **Không** sinh `implementation_status.html` ở giai đoạn này — proposal là design doc, status là live tracker chỉ tạo khi đã bắt đầu code.

### Bước 2 — Khi user nói "implement"/"làm đi"/"code đi" → init status file

Tạo `docs/features/<feature>/implementation_status.html` từ `assets/status_template.html`. Section "Tiến độ" liệt kê các bước high-level (sao chép từ checklist proposal nếu có), status mặc định `todo`. Flip `→ in_progress → done` khi bước thực sự bắt đầu / xong.

Thông báo cho user 1 dòng: "Đã init status file tại `<path>`."

### Bước 3 — Update khi có event quan trọng

| Event | Section | Nội dung |
|-------|---------|----------|
| Bắt đầu / xong 1 bước | Tiến độ | Flip status tag |
| Quyết định không nằm trong design gốc | Decisions | Title + **Why** |
| Compromise / sai khác | Compromises | Bullet + lý do + impact |
| Bug catch (chính bạn hoặc advisor) | Bugs caught & fixed | Mô tả + fix |
| Tạo / sửa file | Files changed | Path (group "Mới" / "Sửa") |
| Hoàn thành task | Badge top + Rollout checklist | `IN PROGRESS` → `DONE`, populate checklist |

**Không update** khi: đang debug nội bộ, retry tool call, fix syntax, đọc file để hiểu code, bước trung gian không ảnh hưởng kết quả.

### Bước 4 — Final pass

Trước khi báo "xong":
1. Đọc lại HTML. Xác nhận: tất cả progress row = `done`, badge top = `DONE`, files changed đủ, rollout checklist có item rõ ràng.
2. Section "Smoke check" (optional): lệnh test/import-check đã chạy + outcome.
3. Thông báo user 1 dòng: "Đã update status file."

## Cấu trúc `implementation_status.html`

Bắt buộc 4 section, theo thứ tự:

1. **Tiến độ** — table 3 cột (`#`, `Bước`, `Trạng thái`). Tag: `t-done`/`t-prog`/`t-todo`.
2. **Decisions + Compromises + Bugs** — gộp / chia card kiểu `.decision`. Mỗi entry: title ngắn + **Why** + **How to apply** / **Fix**.
3. **Files changed** — chia "Mới" / "Sửa".
4. **Rollout checklist** — ordered list các bước user làm sau merge.

Optional khi liên quan: **Smoke check**, **Things you should know**.

Dùng template `assets/status_template.html`. **Không** viết HTML mới từ đầu — luôn dựa template.

## Folder structure user thu được

```
docs/features/<feature>/
├── README.md                       (optional index, khi xong)
├── proposal.html                   (nếu user yêu cầu design trước)
└── implementation_status.html      (live tracker)
```

Folder đã tồn tại → **append** vào file hiện có, không overwrite. Đọc file trước khi edit.

## Lý do skill tồn tại

User chạy nhiều task implement song song. Cần artifact đọc lại sau 1 tuần vẫn hiểu được "Claude đã làm gì, đã quyết định gì khác design, có gì cần review". Câu trả lời dài cuối session không đủ. HTML cho phép structured (table, section), dễ share, diff trong git.

Skill này không thay PR description / commit message. Là **scratchpad cho user**.

## Đọc thêm

- `assets/status_template.html` — template HTML cho `implementation_status.html`.
- `assets/proposal_template.html` — template HTML cho `proposal.html`.
- `references/triggering.md` — danh sách phrase trigger / pause / resume.
