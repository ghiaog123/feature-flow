---
name: ff-planning
description: Biến một vấn đề/quyết định đã chốt thành implementation plan đã được phản biện kỹ. Claude Code và Codex cùng deep-dive vào code thật (Claude tỏa sub-agent song song khi phạm vi lớn), tranh luận ngang hàng qua /codex-think-about để thống nhất cách hiện thực, viết plan ra file .md kèm dependency contract (Owns files/Depends on), rồi chạy /codex-plan-review để hai bên review lại plan (adversarial, sửa tại chỗ tới khi APPROVE hoặc stalemate). Dùng khi user nói "lên implement plan", "deep dive code rồi lập kế hoạch", "claude với codex bàn cách làm rồi review plan", "kế hoạch hiện thực cho vấn đề này", "plan rồi review plan", "tạo plan có phản biện". Thường chạy SAU ff-problem-solver (nhận analysis_brief.md) nhưng cũng nhận trực tiếp một vấn đề rõ ràng. KHÔNG dùng cho: tranh luận quyết định chưa chốt (ff-problem-solver), review code đã viết (codex-review), tracking tiến độ (ff-impl-status), viết test (ff-test-case-writer).
---

# Implementation Planner (Claude ⇄ Codex)

Từ một **quyết định/vấn đề đã chốt** → **implementation plan đã qua phản biện**. Hai pha tranh luận ngang hàng nối tiếp nhau:

```
Quyết định/vấn đề (thường là analysis_brief.md từ ff-problem-solver)
   │
   ▼
[1] Deep-dive code (Claude độc lập, rào thông tin; adaptive: inline hoặc tỏa sub-agent)
   │
   ▼
[2] Debate CÁCH hiện thực   ← delegate /codex-think-about
   │
   ▼
[3] Viết plan ra file .md (kèm dependency contract: Owns files / Depends on)   ← skill này
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
- **Bám giải pháp đã chốt (BẮT BUỘC)**: plan chỉ lo *cách hiện thực* quyết định đã chốt, KHÔNG trôi/đổi/thêm/bớt phạm vi. Nếu deep-dive phát hiện giải pháp đã chốt **không chạy được** → **DỪNG, báo ngược về `ff-problem-solver`** để chốt lại; KHÔNG tự quyết lại giải pháp trong skill này. (Fidelity ngược lên upstream.)
- **Plan bám code thật**, không lý thuyết — mọi bước phải trỏ tới file/hàm/module thật sẽ đụng.
- **Dependency-honest**: mỗi bước khai `Owns files` (file nó sở hữu) + `Depends on` (bước phụ thuộc). Chỉ đánh "song song được" khi file disjoint VÀ không phụ thuộc lẫn nhau. Nghi ngờ → mặc định **tuần tự**. Khai sai "parallel" là nguy hiểm cho executor.
- **Codex KHÔNG sửa file project** ở pha debate (`/codex-think-about`). Ở pha review (`/codex-plan-review`), file được sửa **chỉ là file plan**, không phải code — Claude sửa, không phải Codex.
- **Plan dạng `.md`** để `/codex-plan-review` sửa tại chỗ dễ dàng.
- **Artifact ra đĩa, main giữ gọn**: report deep-dive của sub-agent + transcript debate là trung gian; kết quả sống trong plan `.md`.

## Workflow

### Bước 0 — Nhận đầu vào

Cần một **quyết định/vấn đề đủ rõ để lập kế hoạch**:

- Nếu user vừa chạy `ff-problem-solver` → **lấy `analysis_brief.md` làm đầu vào chính** (quyết định + lý do + đánh đổi + bước tiếp theo + ràng buộc). Đây là nguồn sự thật; plan phải bám nó.
- Nếu chưa có brief → hỏi user phát biểu: cần làm gì, tiêu chí thành công, ràng buộc (stack, không được đổi gì, deadline). Nếu vấn đề còn đang phân vân nhiều phương án → gợi ý chạy `ff-problem-solver` trước, đừng lập plan trên nền chưa chốt.
- Xác định **feature/scope** → chọn thư mục output `docs/features/<feature>/implementation_plan.md` (theo convention của bundle). Hỏi nếu chưa rõ feature.
- Chốt **effort** cho Codex (mặc định `high`).

### Bước 1 — Deep-dive code (RÀO THÔNG TIN, adaptive, Claude độc lập)

Trước khi gọi Codex, Claude tự đọc code để hiểu địa hình. **Chọn cách theo quy mô (adaptive-by-size — cổng chống over-engineer):**

- **Phạm vi nhỏ / đã biết chỗ** → deep-dive **inline** bằng Glob/Grep/Read ngay trên main.
- **Phạm vi lớn / phải quét rộng** (nhiều module, chưa biết touchpoint) → **tỏa parallel, main giữ lean**:
  1. **Index rẻ trước**: nếu có công cụ index code (vd `graphify`) → chạy để có bản đồ nhanh; nếu không → một lượt Glob/Grep định vị vùng.
  2. **Fan-out trong 1 message**: nhiều sub-agent `Explore`/`general-purpose` song song, mỗi con một góc (entry points, module sẽ sửa, data model, integration points, test hiện có, convention). Yêu cầu mỗi con trả **report gọn `file:line` + phát hiện**, KHÔNG dump code.
  3. **Synthesis trên main**: gộp report thành **bản đồ vùng tác động** (file:line touchpoint, phần phụ thuộc, chỗ dễ vỡ). Main không nuốt nguyên file.

Từ bản đồ đó, Claude phác **cách hiện thực sơ bộ** của riêng mình: các bước, thứ tự, file mỗi bước đụng, rủi ro, phần cần thử nghiệm. Đây phải xong và là lập trường độc lập của Claude TRƯỚC Bước 2.

**Kiểm fidelity ngay tại đây**: nếu deep-dive cho thấy giải pháp đã chốt trong brief **mâu thuẫn với code thật / không khả thi** → DỪNG, nêu rõ vì sao, báo ngược về `ff-problem-solver`. Không âm thầm đổi giải pháp rồi lập plan cho cái khác.

### Bước 2 — Debate CÁCH hiện thực: invoke /codex-think-about

Giao phần tranh luận cách tiếp cận cho `/codex-think-about` (engine lo init→poll→cross-analysis→resume→consensus/stalemate→cleanup).

Invoke Skill `codex-think-about`, truyền:
- **QUESTION** = "Cách hiện thực tốt nhất cho `<quyết định đã chốt>` là gì?" (kèm tiêu chí thành công). Debate về *cách làm*, không mở lại *có nên làm không*.
- **PROJECT_CONTEXT** = bản đồ vùng tác động + convention + ràng buộc từ Bước 0-1 + tóm tắt `analysis_brief.md`.
- **RELEVANT_FILES** = các file touchpoint tìm được ở Bước 1 (đường dẫn).
- **CONSTRAINTS** = điều không được vi phạm (gồm: không đổi phạm vi giải pháp đã chốt).
- **EFFORT** = mức đã chốt.

Mang **cách hiện thực sơ bộ ở Bước 1** làm lập trường mở màn của Claude. Tranh luận để hội tụ về: kiến trúc cách làm, thứ tự bước, điểm tích hợp, chiến lược test/rollback, các đánh đổi, **ranh giới sở hữu file giữa các bước** (phục vụ dependency contract ở Bước 3). Tôn trọng mọi luật của codex-think-about (rào thông tin, không cap vòng, File Modification Guard, luôn finalize+stop). Nếu skill không khả dụng → báo user cài codex, KHÔNG tự chế giao thức.

### Bước 3 — Viết implementation plan ra file .md (kèm dependency contract)

Tổng hợp kết quả debate thành plan **hành động được**, ghi `docs/features/<feature>/implementation_plan.md`. Cấu trúc tối thiểu (xem `references/plan-template.md`):

| Mục | Nội dung |
|---|---|
| **Mục tiêu / Outcomes** | Cái gì được coi là xong (nguồn để derive acceptance criteria) |
| **Bối cảnh & quyết định** | Tóm tắt quyết định + lý do (link `analysis_brief.md` / decision record nếu có) |
| **Vùng tác động** | Bảng file:line sẽ đụng + vai trò |
| **Các bước hiện thực** | Đánh số, mỗi bước khai: làm gì, **Owns files** (file nó sở hữu), **Depends on** (bước phụ thuộc), song song được không |
| **Test & verify** | Cách kiểm chứng từng phần + tiêu chí pass |
| **Rủi ro & rollback** | Điều dễ vỡ + cách lùi |
| **Ngoài phạm vi** | Chủ đích không làm lần này |

**Dependency contract** (`Owns files` / `Depends on` cho mỗi bước) là hợp đồng để `ff-implement` parallelize an toàn: bước có file disjoint + không phụ thuộc → chạy song song; còn lại tuần tự. Khai **honest** — nghi thì để tuần tự.

Nếu project cần plan dạng JSON theo `.claude/schemas/plan-schema.json` (xem `.claude/docs/plan-execution-guide.md`) → sinh kèm; nhưng bản `.md` là bản để `/codex-plan-review` review. Giữ heading rõ ràng (`#`/`##`) — `/codex-plan-review` cần cấu trúc heading để bám.

### Bước 4 — Review plan: invoke /codex-plan-review

Giao plan vừa viết cho `/codex-plan-review` để Claude và Codex review lại theo lối adversarial (engine lo loop poll→verdict→apply/rebut→resume tới APPROVE hoặc stalemate, sửa plan tại chỗ).

Invoke Skill `codex-plan-review`, truyền:
- **PLAN_PATH** = đường dẫn tuyệt đối tới `implementation_plan.md` vừa viết.
- **USER_REQUEST** = quyết định/vấn đề gốc (link `analysis_brief.md` nếu có).
- **SESSION_CONTEXT** = plan này sinh từ debate Claude⇄Codex; ràng buộc; feature scope; **plan phải bám giải pháp đã chốt** (đừng để review nới phạm vi).
- **ACCEPTANCE_CRITERIA** = lấy từ mục Mục tiêu/Outcomes.
- **EFFORT** = mức đã chốt.

Mỗi issue Codex nêu mà hợp lệ → Claude **sửa thẳng vào file plan** rồi resume; issue không hợp lệ → rebut có lý do. Lặp tới `verdict === APPROVE` hoặc stalemate. Luôn finalize+stop. Nếu review đề xuất **đổi giải pháp cốt lõi** (không phải cách hiện thực) → đó là tín hiệu quay về `ff-problem-solver`, không tự quyết trong plan review.

### Bước 5 — Hoàn tất + bàn giao

- **APPROVE** → báo plan đã sẵn sàng hiện thực, đường dẫn file, tóm tắt: số vòng review, issue tìm/sửa/bác, rủi ro còn lại, bước tiếp theo.
- **Stalemate** → liệt kê điểm bế tắc (`Điểm | Claude | Codex`), khuyến nghị nên nghiêng bên nào, hỏi user chốt.
- **Chain**: `implementation_plan.md` APPROVE là đầu vào cho `ff-implement` (hiện thực + review code). Nhắc bước kế tiếp của bundle: hiện thực → `ff-impl-status` (tracking), `ff-test-case-writer`/`ff-service-test-runner` (test), `ff-feature-brief` (bàn giao PO/QC).

## Vệ sinh context (compact có kiểm soát)

Đừng `/compact` trần. Sau khi plan APPROVE, đề xuất block compact điền path thật, rõ GIỮ/BỎ:

```
/compact GIỮ: nội dung docs/features/<feature>/implementation_plan.md (bản APPROVE, gồm dependency contract),
tóm tắt: số vòng review + issue đã sửa, rủi ro còn lại.
BỎ: report deep-dive của sub-agent, transcript /codex-think-about và /codex-plan-review, phác thảo plan trung gian
(đã tổng hợp vào plan file; diff còn ở git).
```

Cái gì chưa vào plan file → gắn cờ must-keep.

## Anti-patterns

- ❌ Bỏ deep-dive Bước 1, debate cách làm trên không khí → plan không bám code thật.
- ❌ **Trôi khỏi giải pháp đã chốt** — tự đổi/thêm/bớt phạm vi thay vì báo ngược về `ff-problem-solver`.
- ❌ Lập plan khi quyết định chưa chốt (đáng lẽ chạy `ff-problem-solver` trước).
- ❌ Nuốt nguyên file vào main khi phạm vi lớn (nên tỏa sub-agent trả report gọn); hoặc tỏa sub-agent cho việc nhỏ đã biết chỗ (over-engineer).
- ❌ Khai "song song được" bừa khi file chồng nhau/có phụ thuộc → executor vỡ. Nghi thì tuần tự.
- ❌ Viết plan văn xuôi không bước đánh số / không file touchpoint / không dependency contract → không review được, không parallelize được.
- ❌ Để Codex sửa code project (chỉ được sửa file plan, và do Claude sửa).
- ❌ Tự viết giao thức gọi codex thay vì delegate hai skill có sẵn.
- ❌ Nhận APPROVE giả khi `/codex-plan-review` chưa thật sự ra `verdict: APPROVE`.
