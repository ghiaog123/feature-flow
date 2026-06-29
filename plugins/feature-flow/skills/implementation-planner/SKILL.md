---
name: implementation-planner
description: Biến một vấn đề/quyết định đã chốt thành implementation plan đã được phản biện kỹ. Claude Code và Codex cùng deep-dive vào code thật, tranh luận ngang hàng qua /codex-think-about để thống nhất cách hiện thực, viết plan ra file .md, rồi chạy /codex-plan-review để hai bên review lại plan (adversarial, sửa tại chỗ tới khi APPROVE hoặc stalemate). Dùng khi user nói "lên implement plan", "deep dive code rồi lập kế hoạch", "claude với codex bàn cách làm rồi review plan", "kế hoạch hiện thực cho vấn đề này", "plan rồi review plan", "tạo plan có phản biện". Thường chạy SAU problem-solver (đã có quyết định) nhưng cũng nhận trực tiếp một vấn đề rõ ràng. KHÔNG dùng cho: tranh luận quyết định chưa chốt (problem-solver), review code đã viết (codex-review), tracking tiến độ (impl-status), viết test (test-case-writer).
---

# Implementation Planner (Claude ⇄ Codex)

Từ một **quyết định/vấn đề đã chốt** → **implementation plan đã qua phản biện**. Hai pha tranh luận ngang hàng nối tiếp nhau:

```
Quyết định/vấn đề (thường từ problem-solver)
   │
   ▼
[1] Deep-dive code (Claude độc lập, rào thông tin)
   │
   ▼
[2] Debate CÁCH hiện thực   ← delegate /codex-think-about
   │
   ▼
[3] Viết plan ra file .md   ← skill này
   │
   ▼
[4] Review plan (adversarial, sửa tại chỗ)   ← delegate /codex-plan-review
   │
   ▼
[5] Plan cuối APPROVE + bước hành động
```

Skill này **không tự chế giao thức codex** — nó nối hai engine có sẵn (`/codex-think-about` cho debate approach, `/codex-plan-review` cho review plan) và thêm phần deep-dive + viết plan ở giữa.

## Nguyên tắc cốt lõi

- **Claude và Codex ngang hàng** — không reviewer/implementer. Hai kỹ sư phản biện nhau.
- **Rào thông tin** ở pha deep-dive: Claude PHẢI tự đọc code + dựng cách tiếp cận riêng TRƯỚC khi đọc output Codex.
- **Plan bám code thật**, không lý thuyết — mọi bước phải trỏ tới file/hàm/module thật sẽ đụng.
- **Codex KHÔNG sửa file project** ở pha debate (`/codex-think-about`). Ở pha review (`/codex-plan-review`), file được sửa **chỉ là file plan**, không phải code — Claude sửa, không phải Codex.
- **Plan dạng `.md`** để `/codex-plan-review` sửa tại chỗ dễ dàng.

## Workflow

### Bước 0 — Nhận đầu vào

Cần một **quyết định/vấn đề đủ rõ để lập kế hoạch**:

- Nếu user vừa chạy `problem-solver` → lấy quyết định + lý do + ràng buộc + đánh đổi làm đầu vào.
- Nếu chưa có → hỏi user phát biểu: cần làm gì, tiêu chí thành công, ràng buộc (stack, không được đổi gì, deadline). Nếu vấn đề còn đang phân vân nhiều phương án → gợi ý chạy `problem-solver` trước, đừng lập plan trên nền chưa chốt.
- Xác định **feature/scope** → chọn thư mục output `docs/features/<feature>/implementation_plan.md` (theo convention của bundle). Hỏi nếu chưa rõ feature.
- Chốt **effort** cho Codex (mặc định `high`).

### Bước 1 — Deep-dive code (RÀO THÔNG TIN, Claude độc lập)

Trước khi gọi Codex, Claude tự đọc code để hiểu địa hình:

- Dùng Glob/Grep/Read (hoặc agent Explore) tìm: entry points, module sẽ sửa, data model, integration points, test hiện có, pattern/convention đang dùng.
- Lập **bản đồ vùng tác động**: file:line các touchpoint, phần phụ thuộc, chỗ dễ vỡ.
- Phác **cách hiện thực sơ bộ** của riêng Claude: các bước, thứ tự, rủi ro, phần cần thử nghiệm.

Phần này phải xong và là lập trường độc lập của Claude TRƯỚC Bước 2.

### Bước 2 — Debate CÁCH hiện thực: invoke /codex-think-about

Giao phần tranh luận cách tiếp cận cho `/codex-think-about` (engine lo init→poll→cross-analysis→resume→consensus/stalemate→cleanup).

Invoke Skill `codex-think-about`, truyền:
- **QUESTION** = "Cách hiện thực tốt nhất cho `<quyết định>` là gì?" (kèm tiêu chí thành công).
- **PROJECT_CONTEXT** = bản đồ vùng tác động + convention + ràng buộc từ Bước 0-1.
- **RELEVANT_FILES** = các file touchpoint tìm được ở Bước 1.
- **CONSTRAINTS** = điều không được vi phạm.
- **EFFORT** = mức đã chốt.

Mang **cách hiện thực sơ bộ ở Bước 1** làm lập trường mở màn của Claude. Tranh luận để hội tụ về: kiến trúc cách làm, thứ tự bước, điểm tích hợp, chiến lược test/rollback, các đánh đổi. Tôn trọng mọi luật của codex-think-about (rào thông tin, không cap vòng, File Modification Guard, luôn finalize+stop). Nếu skill không khả dụng → báo user cài codex, KHÔNG tự chế giao thức.

### Bước 3 — Viết implementation plan ra file .md

Tổng hợp kết quả debate thành plan **hành động được**, ghi `docs/features/<feature>/implementation_plan.md`. Cấu trúc tối thiểu (xem `references/plan-template.md`):

| Mục | Nội dung |
|---|---|
| **Mục tiêu / Outcomes** | Cái gì được coi là xong (nguồn để derive acceptance criteria) |
| **Bối cảnh & quyết định** | Tóm tắt quyết định + lý do (link decision record nếu có) |
| **Vùng tác động** | Bảng file:line sẽ đụng + vai trò |
| **Các bước hiện thực** | Đánh số, mỗi bước: làm gì, file nào, phụ thuộc bước nào, song song được không |
| **Test & verify** | Cách kiểm chứng từng phần + tiêu chí pass |
| **Rủi ro & rollback** | Điều dễ vỡ + cách lùi |
| **Ngoài phạm vi** | Chủ đích không làm lần này |

Nếu project cần plan dạng JSON theo `.claude/schemas/plan-schema.json` (xem `.claude/docs/plan-execution-guide.md`) → sinh kèm; nhưng bản `.md` là bản để `/codex-plan-review` review.

Giữ heading rõ ràng (`#`/`##`) — `/codex-plan-review` cần cấu trúc heading để bám.

### Bước 4 — Review plan: invoke /codex-plan-review

Giao plan vừa viết cho `/codex-plan-review` để Claude và Codex review lại theo lối adversarial (engine lo loop poll→verdict→apply/rebut→resume tới APPROVE hoặc stalemate, sửa plan tại chỗ).

Invoke Skill `codex-plan-review`, truyền:
- **PLAN_PATH** = đường dẫn tuyệt đối tới `implementation_plan.md` vừa viết.
- **USER_REQUEST** = quyết định/vấn đề gốc.
- **SESSION_CONTEXT** = plan này sinh từ debate Claude⇄Codex; ràng buộc; feature scope.
- **ACCEPTANCE_CRITERIA** = lấy từ mục Mục tiêu/Outcomes.
- **EFFORT** = mức đã chốt.

Mỗi issue Codex nêu mà hợp lệ → Claude **sửa thẳng vào file plan** rồi resume; issue không hợp lệ → rebut có lý do. Lặp tới `verdict === APPROVE` hoặc stalemate. Luôn finalize+stop.

### Bước 5 — Hoàn tất + bàn giao

- **APPROVE** → báo plan đã sẵn sàng hiện thực, đường dẫn file, tóm tắt: số vòng review, issue tìm/sửa/bác, rủi ro còn lại, bước tiếp theo.
- **Stalemate** → liệt kê điểm bế tắc (`Điểm | Claude | Codex`), khuyến nghị nên nghiêng bên nào, hỏi user chốt.
- Nhắc bước kế tiếp tự nhiên của bundle: hiện thực → `impl-status` (tracking), `test-case-writer`/`service-test-runner` (test), `feature-brief` (bàn giao PO/QC).

## Anti-patterns

- ❌ Bỏ deep-dive Bước 1, debate cách làm trên không khí → plan không bám code thật.
- ❌ Lập plan khi quyết định chưa chốt (đáng lẽ chạy `problem-solver` trước).
- ❌ Viết plan dạng văn xuôi không bước đánh số / không file touchpoint → không review được, không thực thi được.
- ❌ Để Codex sửa code project (chỉ được sửa file plan, và do Claude sửa).
- ❌ Tự viết giao thức gọi codex thay vì delegate hai skill có sẵn.
- ❌ Nhận APPROVE giả khi `/codex-plan-review` chưa thật sự ra `verdict: APPROVE`.
