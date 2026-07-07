---
name: ff-impl-status
description: "Sinh và duy trì file HTML trạng thái sống cho task implement tại `docs/features/<feature>/implementation_status.html`. CHỈ trigger khi user gọi rõ ràng: `/ff-impl-status`, `ff-impl-status`, 'dùng skill ff-impl-status', 'bật status file', 'maintain status file', 'theo dõi implementation status', hoặc 'resume ff-impl-status <feature>' / 'load status <feature>' / 'tiếp tục feature X' để load lại từ proposal + implementation_status.html đã có trong chat mới. KHÔNG tự động trigger ngay cả khi user yêu cầu implement feature lớn. Sau khi user kích hoạt, skill ở trạng thái active đến hết session hoặc khi user nói 'stop status file' / 'skip status'. File ghi tiến độ, decisions ngoài thiết kế, compromises, bugs trong khi code, file thay đổi và rollout checklist. Hỗ trợ sinh proposal.html trước nếu user yêu cầu thiết kế trước; implementation_status.html chỉ sinh khi user xác nhận bắt đầu implement. Cũng hỗ trợ resume context từ session trước bằng cách đọc lại 2 file HTML đó."
---

# ff-impl-status — Live Implementation Status HTML

Sinh + duy trì file HTML tracker khi user implement task đa bước. File là **artifact giao tiếp** — ghi quyết định, thoả hiệp, bugs để user đọc nhanh và challenge điểm không đồng ý.

## Khi nào skill này áp dụng

**Chỉ trigger khi user kích hoạt rõ ràng**:
- `/ff-impl-status`
- "dùng skill ff-impl-status", "bật ff-impl-status"
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

Khi user nói "resume ff-impl-status <feature>", "load status <feature>", "tiếp tục feature X", "đọc lại status file", "tiếp tục từ session trước":

1. **Locate folder.** Tìm `docs/features/<feature>/` trong working directory. Nếu user không nói rõ tên feature:
   - List `docs/features/*/` (Bash hoặc Glob).
   - Nếu có 1 folder → dùng luôn.
   - Nhiều folder → hỏi user chọn (bullet list các tên feature).
   - Không có folder → báo lỗi: "Không tìm thấy `docs/features/<feature>/`. Skill chưa từng chạy cho feature này — bật skill mới bằng `/ff-impl-status`."

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

**Hỏi theo thứ tự tác động kiến trúc — nặng & khó đảo ngược HỎI TRƯỚC.** Trả lời sai một câu impact cao đầu độc mọi thứ phía sau, nên phải lộ ra sớm nhất. Sắp câu hỏi theo thứ tự:
1. **Impact cao, khó sửa về sau** — data model, chọn storage, ràng buộc tương thích ngược / migration, ai là consumer của API.
2. **Impact vừa** — scope (làm gì / không làm gì), nguồn dữ liệu, hành vi edge case.
3. **Impact thấp / dễ đổi** — chi tiết cosmetic, đặt sau cùng.

**Ngữ cảnh session trước là LOW TRUST.** Có thể tái dùng để *gợi ý* câu trả lời, nhưng không coi là chốt. Nếu yêu cầu hiện tại **mâu thuẫn** với điều đã nói trước đó trong session (vd: trước nói dùng Postgres, giờ ngầm định Mongo) → **phải hỏi lại để giải quyết mâu thuẫn**, nêu rõ hai phía conflict, đừng tự chọn.

Dùng tool hỏi nhiều lựa chọn (AskUserQuestion) khi câu hỏi có phương án rời rạc; hỏi open-ended khi cần mô tả tự do. Sau khi user trả lời, mới sinh proposal. Nếu câu trả lời lại mở ra mơ hồ mới đáng kể, hỏi thêm 1 vòng ngắn — đừng kéo dài vô hạn.

**Sau khi user trả lời → chốt thành bảng quyết định.** Tổng hợp câu trả lời thành một **bảng quyết định** gọn (`Câu hỏi/Quyết định | Chốt | Lý do`). Bảng này là **input chuẩn** để dựng proposal — proposal KHÔNG được dựng trên giả định chưa ghi lại. Mọi giả định không tầm thường phải truy về được một dòng trong bảng. Bảng feed thẳng vào đầu proposal (section 1–2): chèn vào box "Quyết định từ interview" gần TL;DR / Scope (có sẵn trong `assets/proposal_template.html`).

#### 1b — Sinh proposal

Sinh `docs/features/<feature>/proposal.html` với 8 section, theo thứ tự WHY → WHAT → HOW.

**Trước section 1 — box TL;DR "Ý tưởng core".** 1–3 câu ngôn ngữ thường, KHÔNG jargon: feature làm gì + giải quyết đau gì. User đọc 10 giây nắm được ý chính trước khi vào chi tiết kĩ thuật. Đây là phần quan trọng nhất cho người đọc không sâu kĩ thuật — viết rõ, cụ thể, tránh thuật ngữ. (Box `.tldr` có sẵn trong template.)

1. **Bối cảnh & Vấn đề** — tại sao cần làm, đau ở đâu, trigger nào dẫn tới đề xuất (1–2 đoạn).
2. **Mục tiêu & Scope** — mục tiêu (xong = gì) + In-scope / Non-goals.
3. **Thay đổi data model** (table).
4. **API surface** (table: method, path, mô tả).
5. **Luồng chi tiết** cho API mới (numbered steps).
6. **Tiêu chí nghiệm thu** — acceptance criteria kiểm chứng được; `ff-test-case-writer` + `ff-feature-brief` map thẳng từ đây.
7. **Đánh giá** — Ưu / Rủi ro / Alternative đã loại.
8. **Checklist triển khai** high-level.

**Logic kĩ thuật → vẽ diagram ĐƠN GIẢN + NHIỀU MÀU cho dễ nhìn, không trình bày bằng text dài.** Template nhúng sẵn Mermaid.js (CDN, theme `base` + palette màu khớp). Dùng block `<pre class="mermaid">` trong `<div class="diagram">`:
- **Luồng chi tiết (5)** → mặc định `flowchart` (box + mũi tên + nhánh `{điều kiện}`) — đơn giản, dễ nhìn. Chỉ dùng `sequenceDiagram` khi cần nhấn tương tác nhiều bên, `stateDiagram-v2` khi cần vòng đời trạng thái. Đừng vẽ phức tạp khi flowchart đủ.
- **Data model (3)** → giữ table cột + `erDiagram` cho quan hệ giữa entity.
- Chỉ giữ text cho phần diagram không tải nổi (rule validate cụ thể, lý do edge case). API surface (4) giữ table — tabular, không cần vẽ.

**Làm diagram SINH ĐỘNG + dễ hiểu** — 3 lớp tín hiệu thị giác (mẫu đầy đủ trong template), user nhìn phát hiểu, không đọc chữ:

1. **Màu theo vai trò** (`classDef` + `class`, `stroke-width:2px`): 🔵 xanh dương = đầu vào/start · 🟡 vàng = quyết định/điều kiện · 🟣 tím = xử lý/logic · ⚪ xám = data store · 🟢 xanh lá = thành công · 🔴 đỏ = lỗi/từ chối. Dùng đúng palette này, không chế thêm màu.
2. **Hình khối theo loại node**: `([...])` stadium = điểm vào/ra (request, response) · `{...}` thoi = quyết định · `[/.../]` hình bình hành = bước xử lý · `[(...)]` trụ = DB/store. Hình khác nhau = vai trò khác nhau, nhận ra ngay.
3. **Icon emoji đầu label** gợi nghĩa nhanh: 👤 client · ⚙️ xử lý · 🗄️ DB · ✅ thành công · ❌ lỗi · 🔑 auth · 📤 gửi · 📥 nhận. 1 icon/node, đừng spam.

Thêm: nhãn cạnh ngắn có dấu (`-->|✓ Có|`, `-->|✗ Không|`); `subgraph` gom node cùng actor (Client / Service / DB) khi luồng qua nhiều bên.

Quy tắc giữ diagram dễ hiểu:
- **≤7 node mỗi diagram.** Nhiều hơn → tách thành 2 diagram hoặc gộp bước phụ.
- Nếu một đoạn mô tả >4–5 bước tuần tự / nhiều nhánh / nhiều quan hệ → chuyển sang diagram. Chọn loại diagram đơn giản nhất biểu diễn đủ logic.
- Label node ngắn (≤4–5 từ), tiếng Việt thường, không nhồi chi tiết vào box.

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
